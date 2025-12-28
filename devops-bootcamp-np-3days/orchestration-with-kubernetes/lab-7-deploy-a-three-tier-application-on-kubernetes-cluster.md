# Lab 7 - Deploy a three tier application on Kubernetes Cluster

In this lab, you will deploy a complete three-tier application on GKE using a provided GitHub repository. You will containerize the frontend and backend, deploy all three tiers to Kubernetes, and expose the application correctly using Kubernetes Services.

### Application Architecture

**Stack overview**

UI (NGINX static site)\
→ API (Node.js backend)\
→ Database (PostgreSQL)

**Service exposure model**

* PostgreSQL: ClusterIP (internal only)
* API: LoadBalancer (public)
* UI: LoadBalancer (public)

***

### What You Will Build

At a high level, you will create:

* Namespace: `user-management`
* PostgreSQL:
  * Deployment
  * PersistentVolumeClaim
  * Secret
  * ClusterIP Service
* Backend API (Node.js):
  * Deployment (2 replicas)
  * LoadBalancer Service
* Frontend UI (NGINX):
  * Deployment
  * ConfigMap for `config.json`
  * LoadBalancer Service

You will validate the setup using `curl` and a web browser, then cleanly remove all resources.

***

### Prerequisites

You must have the following ready:

* An GKE cluster up and running
* `kubectl` configured to access GKE
* Docker installed and logged into your container registry
  * Docker Hub or Azure Container Registry
* Git installed locally
* Basic understanding of Kubernetes Deployments and Services

***

### Step 1: Get the Application Code

Clone the required repository:

```bash
git clone https://github.com/saurabhd2106/usermanagement-lab-ih.git
cd usermanagement-lab-ih
```

#### Repository Structure

```
api/        Node.js backend (uses environment variables)
ui/         Frontend static site (reads config.json)
```

***

### Step 2: Build and Push Container Images

Replace `YOUR_DOCKERHUB` with your Docker Hub username or equivalent registry name.

***

#### Step 2.1: Frontend Image (NGINX)

Create `ui/Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build and push the image:

```bash
cd ui
docker build --platform linux/amd64 -t YOUR_DOCKERHUB/frontend-user-management:latest .
docker login -u YOUR_DOCKERHUB
docker push YOUR_DOCKERHUB/frontend-user-management:latest
cd ..
```

***

#### Step 2.2: Backend Image (Node.js)

Create `api/Dockerfile`:

```dockerfile
FROM node:20-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

Build and push the image:

```bash
cd api
docker build --platform linux/amd64 -t YOUR_DOCKERHUB/backend-user-management:latest .
docker push YOUR_DOCKERHUB/backend-user-management:latest
cd ..
```

***

### Step 3: Create Kubernetes Manifests Directory

```bash
mkdir k8s_solution
cd k8s_solution
```

***

### Step 4: Create the Namespace

Create `namespace.yml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: user-management
```

Apply it:

```bash
kubectl apply -f namespace.yml
```

***

### Step 5: Database Tier (PostgreSQL)

#### Step 5.1: Persistent Volume Claim

Create `db-pvc.yml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: user-management
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

***

#### Step 5.2: Database Secret

Create `db-secret.yml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: user-management
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: admin123
  POSTGRES_DB: postgres_db
```

***

#### Step 5.3: PostgreSQL Deployment

Create `db-deploy.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-db
  namespace: user-management
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-pvc
```

***

#### Step 5.4: Database Service (ClusterIP)

Create `db-svc.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: user-management
spec:
  type: ClusterIP
  selector:
    app: db
  ports:
    - port: 5432
      targetPort: 5432
```

Apply all DB resources:

```bash
kubectl apply -f db-pvc.yml
kubectl apply -f db-secret.yml
kubectl apply -f db-deploy.yml
kubectl apply -f db-svc.yml
kubectl get all -n user-management
```

***

### Step 6: API Tier (Node.js)

#### Step 6.1: Backend Deployment

Create `api-deploy.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: user-management
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend-app
  template:
    metadata:
      labels:
        app: backend-app
    spec:
      containers:
        - name: backend-app
          image: YOUR_DOCKERHUB/backend-user-management:latest
          ports:
            - containerPort: 3000
          env:
            - name: DB_SERVER
              value: postgres-db
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

***

#### Step 6.2: API LoadBalancer Service

Create `api-svc-lb.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lb-backend-service
  namespace: user-management
spec:
  type: LoadBalancer
  selector:
    app: backend-app
  ports:
    - port: 80
      targetPort: 3000
```

Apply and verify:

```bash
kubectl apply -f api-deploy.yml
kubectl apply -f api-svc-lb.yml
kubectl get svc -n user-management -o wide
```

Test API:

```bash
curl -I http://<API_EXTERNAL_IP>/health
```

***

### Step 7: UI Tier (NGINX)

#### Step 7.1: ConfigMap for UI Configuration

Create `ui-configmap.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-configmap
  namespace: user-management
data:
  config.json: |
    {
      "API_URL": "http://REPLACE_WITH_API_LB_IP/"
    }
```

Replace the placeholder with the backend LoadBalancer IP.

***

#### Step 7.2: UI Deployment

Create `ui-deploy.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
  namespace: user-management
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend-app
  template:
    metadata:
      labels:
        app: frontend-app
    spec:
      volumes:
        - name: ui-config
          configMap:
            name: frontend-configmap
      containers:
        - name: frontend-app
          image: YOUR_DOCKERHUB/frontend-user-management:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: ui-config
              mountPath: /usr/share/nginx/html/config.json
              subPath: config.json
              readOnly: true
```

***

#### Step 7.3: UI LoadBalancer Service

Create `ui-svc-lb.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lb-frontend-service
  namespace: user-management
spec:
  type: LoadBalancer
  selector:
    app: frontend-app
  ports:
    - port: 80
      targetPort: 80
```

Apply and verify:

```bash
kubectl apply -f ui-configmap.yml
kubectl apply -f ui-deploy.yml
kubectl apply -f ui-svc-lb.yml
kubectl get svc -n user-management -o wide
```

Open in browser:

```
http://<UI_EXTERNAL_IP>/
```

***

### Step 8: End-to-End Verification

Check Pods:

```bash
kubectl get pods -n user-management -o wide
```

Check Services and endpoints:

```bash
kubectl get svc -n user-management
kubectl get endpoints -n user-management
```

Test API:

```bash
curl -I http://<API_EXTERNAL_IP>/health
```

Test UI:

* Open UI in browser
* Perform user operations
* Verify API connectivity

If UI cannot reach API:

```bash
kubectl apply -f ui-configmap.yml
kubectl rollout restart deploy/frontend-deploy -n user-management
```

***

### Step 9: Cleanup

Delete the namespace to remove all resources:

```bash
kubectl delete ns user-management
```


# Lab 9: Exposing a Three-Tier Application Using Ingress on AWS (EKS)

## What you’ll need (continuation from the previous lab)

* An **Amazon EKS cluster** with `kubectl` configured
* AWS CLI configured and authenticated
* Namespace `user-management` already created
* App from the previous lab running:

  * `postgres-db` Service (ClusterIP)
  * `backend-deploy` Deployment
  * `frontend-deploy` Deployment
* Helm installed locally

---

## Architecture goal

* One public Load Balancer (created by NGINX Ingress Controller)
* Path-based routing:

  * `/` → frontend (ClusterIP)
  * `/backend/...` → backend (ClusterIP)
* Postgres remains internal only

---

## Step 0: Connect to your EKS cluster

If you are not already connected:

```bash
aws eks update-kubeconfig \
  --name <CLUSTER_NAME> \
  --region <AWS_REGION>
```

Verify access:

```bash
kubectl get nodes
kubectl get ns
```

---

## Step 1: Clean up previous LoadBalancer Services

If frontend or backend were exposed using `type: LoadBalancer`, delete them.

```bash
kubectl delete svc lb-backend-service -n user-management --ignore-not-found
kubectl delete svc lb-frontend-service -n user-management --ignore-not-found
```

Verify workloads are still healthy:

```bash
kubectl get deploy,rs,pods,svc -n user-management -o wide
```

Only ClusterIP services should remain for app components.

---

## Step 2: Create ClusterIP Services

### 2.1 Backend ClusterIP Service

Create `backend-clusterip.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-clusterip-service
  namespace: user-management
spec:
  type: ClusterIP
  selector:
    app: backend-app
  ports:
    - port: 80
      targetPort: 3000
```

Apply:

```bash
kubectl apply -f backend-clusterip.yml
```

---

### 2.2 Frontend ClusterIP Service

Create `frontend-clusterip.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-clusterip-service
  namespace: user-management
spec:
  type: ClusterIP
  selector:
    app: frontend-app
  ports:
    - port: 80
      targetPort: 80
```

Apply and verify:

```bash
kubectl apply -f frontend-clusterip.yml
kubectl get svc -n user-management
```

You should now see three ClusterIP services:

* `postgres-db`
* `backend-clusterip-service`
* `frontend-clusterip-service`

---

## Step 3: Install NGINX Ingress Controller on EKS (once per cluster)

On EKS, NGINX Ingress Controller creates **one Service of type LoadBalancer**, which becomes the **single public entry point**.

### 3.1 Add Helm repository

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

---

### 3.2 Install the controller

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx \
  --set controller.service.type=LoadBalancer
```

Verify installation:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Fetch the public endpoint (this is your `<INGRESS_HOST>`):

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

If `EXTERNAL-IP` is pending, wait and re-run. You can inspect events:

```bash
kubectl get events -n ingress-nginx --sort-by=.lastTimestamp | tail -n 30
```

---

## Step 4: Point the frontend to the backend via Ingress path

Update the frontend so it calls the backend using `/backend/` through the Ingress.

Create or update `ui-configmap.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-configmap
  namespace: user-management
data:
  config.json: |
    {
      "API_URL": "http://<INGRESS_HOST>/backend/"
    }
```

Replace `<INGRESS_HOST>` with the external hostname or IP from Step 3.

Apply and restart frontend:

```bash
kubectl apply -f ui-configmap.yml
kubectl rollout restart deploy/frontend-deploy -n user-management
```

---

## Step 5: Create Ingress resources (single public endpoint, two paths)

### 5.1 Frontend Ingress

Create `frontend-ingress.yml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: user-management
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: frontend-clusterip-service
                port:
                  number: 80
```

---

### 5.2 Backend Ingress

Create `backend-ingress.yml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  namespace: user-management
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /backend(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: backend-clusterip-service
                port:
                  number: 80
```

Apply and verify:

```bash
kubectl apply -f frontend-ingress.yml
kubectl apply -f backend-ingress.yml

kubectl get ingress -n user-management
```

---

## Step 6: End-to-end testing

### 6.1 Frontend via Ingress

```bash
curl -I http://<INGRESS_HOST>/
```

Or open in browser:

```
http://<INGRESS_HOST>/
```

---

### 6.2 Backend via Ingress (direct)

```bash
curl -I http://<INGRESS_HOST>/backend/health
curl -s http://<INGRESS_HOST>/backend/api/user | head
```

---

### 6.3 Frontend to Backend (UI wiring)

Open:

```
http://<INGRESS_HOST>/
```

Test UI flows that call backend APIs.

If API calls fail:

* Ensure `API_URL` ends with `/backend/`
* Ensure frontend pods were restarted

---

## Troubleshooting checklist (EKS-specific)

### Ingress external IP or hostname pending

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
kubectl get events -n ingress-nginx --sort-by=.lastTimestamp | tail -n 30
```

---

### `/backend/...` returns 404

* Verify backend ingress path regex and rewrite annotations
* Re-apply backend ingress
* Check controller logs:

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=200
```

---

### Service has no endpoints

Ensure selectors match pod labels:

```bash
kubectl get pods -n user-management --show-labels
kubectl describe svc backend-clusterip-service -n user-management
kubectl describe svc frontend-clusterip-service -n user-management
```

---

## Key learning outcomes

* One NGINX Ingress controller creates one AWS Load Balancer
* Path-based routing replaces multiple per-service LoadBalancers
* ClusterIP services remain internal and reusable
* This pattern mirrors GKE Ingress behavior using AWS primitives
* Forms the foundation for Gateway API or ALB-based routing in advanced labs
# Lab 5 - Exposing an application with Kubernetes Services

In this lab, you will expose an NGINX application running on Kubernetes using three Service types: ClusterIP, NodePort, and LoadBalancer. You will verify connectivity and understand how Services route traffic to Pods using label selectors.

This lab is written with GKE in mind, but it works on any Kubernetes cluster that supports cloud load balancers.

***

### Prerequisites

You need the following:

* A running Kubernetes cluster\
  Examples: AKS, EKS, GKE, Minikube with LoadBalancer support
* `kubectl` installed
* `kubectl` configured to point to your cluster

#### Sanity Check

Run the following commands to confirm cluster access:

```bash
kubectl version --short
kubectl cluster-info
kubectl get nodes -o wide
```

***

### Learning Objectives

By the end of this lab, you will be able to:

* Understand how Services route traffic to Pods using label selectors
* Create and test ClusterIP, NodePort, and LoadBalancer Services
* Verify Service endpoints and troubleshoot common networking issues

***

### Section 0: Create a Clean Playground Namespace

Create a dedicated namespace to isolate all networking resources.

```bash
kubectl create ns lab-networking
kubectl config set-context --current --namespace=lab-networking
kubectl get ns
```

***

### Section 1: Deploy NGINX with 3 Replicas

#### Create the Deployment Manifest

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27.2
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
```

#### Apply and Verify

```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods -o wide
```

Expected outcome:

* One Deployment
* One ReplicaSet
* Three running Pods

***

### Section 2: ClusterIP Service (Internal Access Only)

ClusterIP exposes the application only inside the Kubernetes cluster.

#### Create the Service Manifest

Create a file named `svc-clusterip.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-clusterip
  labels:
    app: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

#### Apply and Inspect

```bash
kubectl apply -f svc-clusterip.yaml
kubectl get svc web-clusterip
kubectl get endpoints web-clusterip
kubectl describe svc web-clusterip
```

Explanation:

* The Service selects Pods with label `app=web`
* Endpoints should list the Pod IPs

#### Test ClusterIP from Inside the Cluster

Run a temporary BusyBox Pod:

```bash
kubectl run tbox --rm -it --image=busybox:1.36 -- /bin/sh
```

Inside the Pod:

```sh
wget -qO- http://web-clusterip
exit
```

You should see the NGINX default page HTML.

***

### Section 3: NodePort Service (Node-Level Access)

NodePort exposes the application on a static port on each node.

#### Create the Service Manifest

Create a file named `svc-nodeport.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  labels:
    app: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32080
```

Note:

* NodePort values must be between 30000 and 32767

#### Apply and Inspect

```bash
kubectl apply -f svc-nodeport.yaml
kubectl get svc web-nodeport
kubectl get nodes -o wide
```

***

#### How to Access NodePort on AKS

On AKS, nodes are not publicly accessible by default. NodePort is reachable only from within the virtual network.

Test from inside the cluster:

```bash
kubectl run tbox2 --rm -it --image=busybox:1.36 -- /bin/sh
```

Inside the Pod, replace `<node-internal-ip>` with a node Internal IP:

```sh
wget -qO- http://<node-internal-ip>:32080
exit
```

If your cluster nodes have public IPs, which is uncommon on AKS, you can access:

```
http://<node-public-ip>:32080
```

from your local machine.

***

### Section 4: LoadBalancer Service (Public Access on AKS)

LoadBalancer exposes the application using a cloud-managed public IP.

#### Create the Service Manifest

Create a file named `svc-lb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  labels:
    app: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

#### Apply and Watch

```bash
kubectl apply -f svc-lb.yaml
kubectl get svc web-lb -w
```

Within one to two minutes, an external IP should be assigned.

Verify:

```bash
kubectl get svc web-lb -o wide
```

#### Test from Your Machine

```bash
curl -I http://<EXTERNAL-IP>
```

Or open the URL in a browser:

```
http://<EXTERNAL-IP>
```

***

### Section 5: How Services Connect to Pods

#### Endpoints and EndpointSlices

```bash
kubectl get endpoints web-lb
kubectl get endpointslice | grep web
```

Explanation:

* Endpoints list the Pod IPs backing the Service
* EndpointSlices are the scalable replacement for Endpoints

#### Watch Endpoints Update During Scaling

```bash
kubectl scale deploy web-deploy --replicas=5
kubectl get endpoints web-lb -w
```

#### Verify Label Selectors

```bash
kubectl get pods --show-labels
kubectl get svc web-lb -o jsonpath='{.spec.selector}{"\n"}'
```

Explanation:

* Service selectors must match Pod labels
* Mismatches cause Services to have no endpoints

***

### Section 6: Quick Verification Cheatsheet

```bash
kubectl get all
kubectl get svc -o wide
kubectl describe svc web-lb
```

Continuous check during scaling or updates:

```bash
watch -n 1 'curl -sI http://<EXTERNAL-IP> | head -n 1'
```

***

### Section 7: Cleanup

Delete all resources created in this lab.

```bash
kubectl delete -f svc-lb.yaml -f svc-nodeport.yaml -f svc-clusterip.yaml -f deployment.yaml
kubectl delete ns lab-networking
```

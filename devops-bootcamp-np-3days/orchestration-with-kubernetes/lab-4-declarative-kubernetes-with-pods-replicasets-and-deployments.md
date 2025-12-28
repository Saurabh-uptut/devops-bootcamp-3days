# Lab 4 - Declarative Kubernetes with Pods, ReplicaSets, and Deployments

In this lab, you will create Pods, ReplicaSets, and Deployments using declarative YAML manifests. You will then practice scaling and rolling updates by modifying YAML files and reapplying them.

***

### Prerequisites

You need the following:

* A running Kubernetes cluster\
  Examples: Minikube, kind, Docker Desktop, k3s, GKS, or any managed Kubernetes service
* `kubectl` installed
* `kubectl` configured to point to your cluster

***

### Learning Objectives

By the end of this lab, you will be able to:

* Define Pod, ReplicaSet, and Deployment manifests using YAML
* Apply and manage resources declaratively
* Scale replicas by editing manifests
* Perform rolling updates safely and observe progress

***

### Section 0: Create a Tidy Playground Namespace

Create a dedicated namespace to isolate all lab resources.

```bash
kubectl create ns lab-declarative
kubectl config set-context --current --namespace=lab-declarative
kubectl get ns
```

***

### Section 1: Pod Using Declarative YAML

#### Create the Pod Manifest

Create a file named `pod.yaml` with the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: myapp
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
```

#### Apply and Verify

```bash
kubectl apply -f pod.yaml
kubectl get pods -o wide
kubectl describe pod web-pod
```

Explanation:

* The Pod runs a single NGINX container
* Labels are added for future selection and grouping

#### Clean Up the Pod

Remove the Pod before moving to higher-level controllers.

```bash
kubectl delete -f pod.yaml
```

***

### Section 2: ReplicaSet Using Declarative YAML

ReplicaSets ensure that a specified number of Pods are always running.

#### Create the ReplicaSet Manifest

Create a file named `replicaset.yaml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  labels:
    app: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
```

#### Apply and Explore

```bash
kubectl apply -f replicaset.yaml
kubectl get rs,pods -o wide
kubectl describe rs web-rs
```

Explanation:

* The ReplicaSet creates three Pods
* If any Pod is deleted, the ReplicaSet recreates it automatically

#### Scale the ReplicaSet Declaratively

1. Edit `replicaset.yaml` and change:

```yaml
replicas: 3
```

to:

```yaml
replicas: 5
```

2. Reapply the file:

```bash
kubectl apply -f replicaset.yaml
kubectl get rs,pods
```

#### Delete the ReplicaSet

This also deletes all Pods it manages.

```bash
kubectl delete -f replicaset.yaml
```

***

### Section 3: Deployment Using Declarative YAML

Deployments manage ReplicaSets and provide rolling updates and rollbacks.

#### Create the Deployment Manifest

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
  labels:
    app: web-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
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
kubectl describe deploy web-deploy
```

Explanation:

* The Deployment creates a ReplicaSet
* The ReplicaSet creates Pods
* Probes ensure traffic is sent only to healthy Pods

***

### Section 4: Rolling Update Using Declarative Changes

#### Update the Deployment Image

Edit `deployment.yaml` and change the image:

```diff
- image: nginx:1.25-alpine
+ image: nginx:1.27.2
```

#### Apply and Watch the Rollout

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/web-deploy
kubectl get pods -w
```

Expected behavior:

* New Pods are created gradually
* Old Pods are removed only after new Pods become Ready

#### Rollout History and Rollback

```bash
kubectl rollout history deployment/web-deploy
kubectl rollout undo deployment/web-deploy
kubectl rollout status deployment/web-deploy
```

***

### Section 5: Common kubectl Commands

```bash
kubectl get all
kubectl get pods --show-labels -o wide
kubectl describe deploy web-deploy
kubectl get deploy web-deploy -o yaml | head -n 40
```

#### Logs and Shell Access

Use an actual Pod name from `kubectl get pods`:

```bash
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/sh
```

#### View Recent Events

```bash
kubectl get events --sort-by=.lastTimestamp | tail -n 20
```

***

### Section 6: Cleanup

Remove all resources created during the lab.

```bash
kubectl delete svc web-svc --ignore-not-found
kubectl delete -f deployment.yaml --ignore-not-found
kubectl delete -f replicaset.yaml --ignore-not-found
kubectl delete -f pod.yaml --ignore-not-found
kubectl delete ns lab-declarative
```

***

### Troubleshooting Quick Reference

#### Rollout Stuck

* Inspect Deployment events:

```bash
kubectl describe deploy web-deploy
```

* Ensure the readiness probe is passing

#### Service Has No Endpoints

* Verify Pods are Ready
* Ensure labels match the Service selector:

```bash
kubectl get pods --show-labels
kubectl describe svc web-svc
```

#### ImagePullBackOff

* Verify the image tag
* Check network egress and registry access

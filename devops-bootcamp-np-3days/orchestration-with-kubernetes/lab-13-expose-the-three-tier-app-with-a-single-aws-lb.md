# Lab: Expose the Three-Tier App with a Single AWS Load Balancer (EKS + AWS Load Balancer Controller)

## What you will achieve

Expose both **frontend** and **backend** through a **single public AWS Application Load Balancer (ALB)** using path-based routing on Amazon EKS.

Routing behavior:

* `/` → frontend service (ClusterIP)
* `/backend/*` → backend service (ClusterIP) with prefix preserved
* Postgres remains internal (ClusterIP only)

This lab is the AWS equivalent of using a single Gateway on GKE, implemented with **AWS Load Balancer Controller + Ingress**.

---

## Prerequisites

* An existing **Amazon EKS cluster**
* `kubectl` configured to talk to the cluster
* Namespace `user-management` already exists
* Three-tier app from previous lab running:

  * `postgres-db` Service (ClusterIP)
  * `backend-deploy`
  * `frontend-deploy`
* AWS CLI configured
* IAM permissions to create ALB, Target Groups, IAM roles, security groups

---

## Architecture Overview

* Amazon EKS
* AWS Load Balancer Controller
* One public Application Load Balancer
* One Kubernetes Ingress
* Two ClusterIP services:

  * frontend
  * backend
* Path-based routing on ALB

---

## Step 0: Mandatory platform checks

### 0.1 Verify EKS connectivity

```bash
kubectl get nodes
kubectl get ns
```

You must see your cluster nodes in `Ready` state.

---

### 0.2 Ensure AWS Load Balancer Controller is installed

Check if the controller exists:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

If it is **not present**, install it.

#### Required components

* IAM OIDC provider for the cluster
* IAM role for the controller
* Helm chart installation

High-level verification only is shown here since installation is usually covered in an earlier lab.

Verify controller logs:

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

There should be no permission or credential errors.

---

## Step 1: Clean up previous per-service LoadBalancers

If your frontend or backend were previously exposed using `type: LoadBalancer`, delete them.

```bash
kubectl delete svc lb-backend-service -n user-management --ignore-not-found
kubectl delete svc lb-frontend-service -n user-management --ignore-not-found
```

Verify only ClusterIP services remain:

```bash
kubectl get svc -n user-management
```

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

---

### 2.3 Mandatory check: Service endpoints

Before proceeding, ensure endpoints exist.

```bash
kubectl get endpointslice -n user-management -l kubernetes.io/service-name=frontend-clusterip-service
kubectl get endpointslice -n user-management -l kubernetes.io/service-name=backend-clusterip-service
```

If endpoints are empty, fix label selectors immediately.

---

## Step 3: Create a single Ingress backed by AWS ALB

### 3.1 Create Ingress manifest

Create `userapp-ingress.yml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: userapp-ingress
  namespace: user-management
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /
spec:
  rules:
    - http:
        paths:
          - path: /backend
            pathType: Prefix
            backend:
              service:
                name: backend-clusterip-service
                port:
                  number: 80

          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-clusterip-service
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f userapp-ingress.yml
```

---

### 3.2 Verify Ingress provisioning

```bash
kubectl get ingress -n user-management
kubectl describe ingress userapp-ingress -n user-management
```

Wait until:

* `ADDRESS` field is populated
* ALB is created in AWS

Save the value as:

```bash
export ALB_DNS=<alb-dns-name>
```

---

## Step 4: Update frontend to call backend through ALB

The frontend must call the backend using the same public ALB.

### 4.1 Update frontend ConfigMap

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
      "API_URL": "http://ALB_DNS/backend/"
    }
```

Replace `ALB_DNS` with the actual DNS name.

Apply and restart frontend:

```bash
kubectl apply -f ui-configmap.yml
kubectl rollout restart deploy/frontend-deploy -n user-management
```

---

## Step 5: Fix backend 503 errors using ALB health checks

### Symptom

You may see:

```bash
curl -I http://ALB_DNS/backend/health
HTTP/1.1 503 Service Unavailable
```

This means the ALB target group considers the backend unhealthy.

---

### 5.1 Confirm backend health internally

```bash
kubectl run -n user-management curlpod \
  --image=curlimages/curl:8.5.0 -it --rm -- \
  sh -c 'curl -i http://backend-clusterip-service/health || true'
```

If this succeeds, the issue is **ALB health check mismatch**.

---

### 5.2 Override backend health check path

Create `backend-ingress-annotation.yml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-health-override
  namespace: user-management
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  rules:
    - http:
        paths:
          - path: /backend
            pathType: Prefix
            backend:
              service:
                name: backend-clusterip-service
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f backend-ingress-annotation.yml
```

This ensures the ALB health check hits `/health`.

---

## Step 6: End-to-end verification

### 6.1 Frontend

```bash
curl -I http://ALB_DNS/
```

Expected:

* HTTP 200

---

### 6.2 Backend

```bash
curl -I http://ALB_DNS/backend/health
curl -s http://ALB_DNS/backend/api/user | head
```

Expected:

* HTTP 200
* Valid backend response

---

### 6.3 UI test

Open in browser:

```
http://ALB_DNS/
```

Frontend should load and successfully call backend APIs.

---

## Troubleshooting checklist (AWS-specific)

### A) Ingress stuck without ADDRESS

* AWS Load Balancer Controller not installed
* IAM permissions missing
* Ingress class not set to `alb`

---

### B) ALB created but backend shows 503

* Health check path mismatch
* Wrong target port
* Service endpoints missing

---

### C) “Target group has no registered targets”

* `alb.ingress.kubernetes.io/target-type` must be `ip`
* Pods must be running and Ready

---

### D) Frontend loads but API calls fail

* Frontend config still pointing to old service
* Missing `/backend` prefix
* Frontend pod not restarted after ConfigMap change

---

## Key learning outcomes

* Single AWS ALB can expose multiple services cleanly
* Path-based routing replaces multiple LoadBalancer services
* ClusterIP services remain internal and reusable
* Health checks are critical for ALB stability
* This pattern mirrors GKE Gateway design using AWS primitives

If you want, I can also provide:

* Terraform version of this setup
* AWS Gateway API alternative using experimental support
* Diagram and teaching slides aligned to this lab

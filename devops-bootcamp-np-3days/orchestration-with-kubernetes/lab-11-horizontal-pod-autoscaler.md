# Lab 11 - Horizontal Pod Autoscaler

Let's append the existing lab to enable **Horizontal Pod Autoscaler (HPA)** for the backend (`backend-deploy`) on GKE.

Your backend Deployment already has CPU and memory requests/limits, so it is ready for CPU-based autoscaling.

### Step 1: Add HPA for Backend API

#### Prerequisite check: Metrics API is available

HPA needs metrics. On GKE, Metrics Server is commonly available, but verify:

```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1" | head
```

If that returns JSON, you are good.

You can also confirm metrics for pods:

```bash
kubectl top pods -n user-management
```

***

### 1.1 Create `hpa-backend.yml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: user-management
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deploy
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

What this does:

* Keeps backend at minimum 2 replicas (matches your lab design)
* Scales up to 6 replicas
* Targets average CPU utilization of 60%

***

### 1.2 Apply and verify

```bash
kubectl apply -f hpa-backend.yml
kubectl get hpa -n user-management
kubectl describe hpa backend-hpa -n user-management
```

***

### 1.3 Trigger scaling (optional validation)

Generate load against your backend LoadBalancer IP (from `lb-backend-service`):

```bash
kubectl get svc -n user-management lb-backend-service -o wide
```

Then run a simple load test (pick one):

Option A: Using `hey` (if installed)

```bash
hey -z 2m -c 30 http://<API_EXTERNAL_IP>/health
```

Option B: Using `curl` loop

```bash
for i in {1..5000}; do curl -s http://<API_EXTERNAL_IP>/health > /dev/null; done
```

Watch scaling:

```bash
watch -n 5 "kubectl get hpa -n user-management && kubectl get deploy -n user-management backend-deploy"
```

***

### Common issues and quick fixes

#### HPA shows: “unknown” metrics

* Metrics API not available or not ready:

```bash
kubectl top pods -n user-management
kubectl get apiservices | grep metrics
```

#### HPA does not scale up even under load

* Ensure backend pods have CPU requests (you already set `cpu: 100m`)
* Ensure traffic is actually hitting the pods (LB is healthy)


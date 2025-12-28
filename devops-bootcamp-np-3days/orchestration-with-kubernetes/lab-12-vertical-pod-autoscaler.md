# Lab 12 - Vertical Pod Autoscaler

This lab is an add-on to your existing GKE three tier app that adds **Vertical Pod Autoscaler (VPA)** to the same backend (`backend-deploy`).

Important: you already added **HPA on CPU**. Running VPA in full Auto mode on CPU requests can fight with HPA decisions. The safest, recommended teaching pattern is:

* Keep HPA scaling replicas on CPU
* Run VPA in **recommendation mode** so you can observe right-sizing without destabilizing the service

GKE supports enabling Vertical Pod Autoscaling on the cluster and then applying a `VerticalPodAutoscaler` resource.

#### Outcome

You will enable VPA on the cluster (if needed) and attach a VPA policy to `backend-deploy` to generate CPU/memory recommendations.

## Step 0: Enable VPA feature on the GKE cluster

Autopilot clusters have VPA enabled by default. Standard clusters need it enabled.&#x20;

### 0.1 Check if VPA CRD exists

```bash
kubectl get crd | grep verticalpodautoscaler || true
```

If you see a CRD like `verticalpodautoscalers.autoscaling.k8s.io`, you can proceed.

### 0.2 Enable VPA on Standard cluster (if CRD is missing)

```bash
gcloud container clusters update <CLUSTER_NAME> \
  --region <REGION> \
  --enable-vertical-pod-autoscaling
```

This is the GKE documented way to enable the feature.&#x20;

Re-check:

```bash
kubectl get crd | grep verticalpodautoscaler
```

***

## Step 1: Create the VPA resource for the backend

Create `vpa-backend.yml`:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
  namespace: user-management
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: backend-deploy
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: backend-app
        controlledResources:
          - cpu
          - memory
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "1"
          memory: "1Gi"
```

Why these settings:

* `updateMode: Off` means VPA only recommends values (no evictions, no automatic changes).
* min/max guardrails keep recommendations within a safe range for a lab environment.

Apply:

```bash
kubectl apply -f vpa-backend.yml
```

***

## Step 2: Verify VPA is producing recommendations

VPA needs some runtime data. Generate a bit of traffic to the backend and wait a few minutes.

### 2.1 Check VPA object

```bash
kubectl get vpa -n user-management
kubectl describe vpa backend-vpa -n user-management
```

### 2.2 View recommendations

```bash
kubectl get vpa backend-vpa -n user-management -o yaml | sed -n '/recommendation:/,$p'
```

You should see recommended CPU and memory requests for the container.

***

## Step 3: Optional teaching upgrade

### Option A: Apply recommendations manually (recommended for training)

1. Copy the recommended `cpu` and `memory` requests
2. Update `api-deploy.yml` in the backend container `resources.requests`
3. Apply and restart:

```bash
kubectl apply -f api-deploy.yml
kubectl rollout restart deploy/backend-deploy -n user-management
```

### Option B: Auto apply (use only if you are not using HPA on CPU)

If you ever teach VPA Auto mode, know that it can evict pods to apply new requests.&#x20;

To enable:

```yaml
updatePolicy:
  updateMode: "Auto"
```

I do not recommend `Auto` for this lab since you already have CPU HPA.

***

## Troubleshooting quick fixes

#### VPA CRD not found

Enable VPA on the cluster:

```bash
gcloud container clusters update <CLUSTER_NAME> --region <REGION> --enable-vertical-pod-autoscaling
```

#### Recommendations are empty

* Wait longer (VPA needs usage history).&#x20;
* Ensure the backend pods are receiving traffic
* Confirm backend pod requests exist (your deployment already has them)


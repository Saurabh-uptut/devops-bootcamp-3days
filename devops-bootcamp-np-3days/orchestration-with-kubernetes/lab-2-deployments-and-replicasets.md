# Lab 2 - Deployments and ReplicaSets

In this lab, you will move from single Pods to running a self-healing and scalable application using Kubernetes Deployments. Deployments manage ReplicaSets, and ReplicaSets ensure that the required number of Pods are always running.

All tasks in this lab use imperative `kubectl` commands and the official NGINX image.

***

### Prerequisites

You need the following:

* A running Kubernetes cluster\
  Examples: Minikube, kind, Docker Desktop, k3s, AKS, or any managed Kubernetes service
* `kubectl` installed
* `kubectl` configured to point to your cluster

***

### Learning Objectives

By the end of this lab, you will be able to:

* Understand how Deployments create and manage ReplicaSets and Pods
* Scale applications up and down manually
* Observe Kubernetes self-healing behavior
* Use common day-1 operational commands such as get, describe, scale, expose, and rollout

***

### Section 1: Create a Clean Playground Namespace

Create a dedicated namespace to isolate lab resources.

```bash
kubectl create ns lab-deploy
kubectl config set-context --current --namespace=lab-deploy
kubectl get ns
```

Explanation:

* Creates a namespace named `lab-deploy`
* Sets it as the default namespace for the current context
* Verifies that the namespace exists

***

### Section 2: Create a Deployment Imperatively

Create a Deployment named `web` running the NGINX container image.

```bash
kubectl create deployment web --image=nginx:latest
```

#### Verify the Created Resources

```bash
kubectl get deployments
kubectl get rs
kubectl get pods -o wide
```

Explanation:

* `kubectl get deployments` shows the Deployment
* `kubectl get rs` shows the ReplicaSet created by the Deployment
* `kubectl get pods -o wide` shows the running Pods and their node placement

#### Inspect the Deployment Internals

```bash
kubectl describe deployment web
kubectl describe rs
kubectl get deploy web -o yaml | head -n 30
```

What happened:

* The Deployment created a ReplicaSet
* The ReplicaSet created one Pod
* If a Pod fails, the ReplicaSet automatically creates a replacement

***

### Section 3: Scale the Application Manually

#### Scale Up to 5 Replicas

```bash
kubectl scale deployment web --replicas=5
kubectl get pods
```

#### Scale Down to 3 Replicas

```bash
kubectl scale deployment web --replicas=3
kubectl get deploy,rs,pods
```

#### View All Resources Together

```bash
kubectl get all
```

Explanation:

* Scaling the Deployment updates the desired replica count
* The ReplicaSet ensures the correct number of Pods are running

***

### Section 4: Observe Self-Healing Behavior

Delete a Pod manually and watch Kubernetes recover automatically.

1. List Pods:

```bash
kubectl get pods
```

2. Delete one Pod:

```bash
kubectl delete pod <one-pod-name>
```

3. Watch Pods in real time:

```bash
kubectl get pods -w
```

Expected outcome:

* A new Pod is created automatically to maintain the desired replica count

***

### Section 5: Deployment Rollouts and Rollbacks

#### Update the Container Image

Simulate a new application version by updating the image.

```bash
kubectl set image deployment/web nginx=nginx:1.27.2
kubectl rollout status deployment/web
```

#### View Rollout History

```bash
kubectl rollout history deployment/web
```

#### Roll Back to the Previous Version

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

Explanation:

* Kubernetes keeps revision history for Deployments
* Rollbacks are fast and safe

***

### Section 6: Labels, Selectors, and Events

#### Add Labels to the Deployment

```bash
kubectl label deployment web tier=frontend env=dev --overwrite
```

Verify labels:

```bash
kubectl get deploy -L tier,env
kubectl get pods --show-labels
```

Explanation:

* Labels added to the Deployment are applied to newly created Pods
* Labels are used for grouping and selection

#### Check the Deployment Selector

```bash
kubectl get deploy web -o jsonpath='{.spec.selector.matchLabels}{"\n"}'
```

Explanation:

* The selector defines which Pods are managed by the Deployment

#### View Recent Events

```bash
kubectl get events --sort-by=.lastTimestamp | tail -n 20
```

Explanation:

* Events are extremely useful for debugging failures and scheduling issues

***

### Section 7: Handy Operational Commands

#### Detailed Listings

```bash
kubectl get deploy,rs,pods -o wide
```

#### Built-In Documentation

```bash
kubectl explain deployment
kubectl explain deployment.spec.strategy
```

Explanation:

* `kubectl explain` shows Kubernetes API documentation directly from the cluster

#### Logs and Shell Access

Pick any current Pod name:

```bash
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/sh
```

***

### Section 8: Cleanup

Delete the namespace to remove all lab resources.

```bash
kubectl delete ns lab-deploy
```

***

### Troubleshooting Tips

* `ImagePullBackOff` or `ErrImagePull`\
  Check the image name and tag, for example `nginx:1.27.2`, and verify network access to the registry
*   Service has no endpoints\
    Ensure Pod labels match the Service selector\
    Use:

    ```bash
    kubectl get pods --show-labels
    kubectl describe svc web-svc
    ```


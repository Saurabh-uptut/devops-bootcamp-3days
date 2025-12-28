# Lab 3 - Rolling updates

#### Goal

Roll out a newer version of an application with near-zero downtime, observe the update in real time, and practice safe rollback.

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

* Understand how Deployments perform rolling updates using ReplicaSets
* Monitor update progress using `kubectl rollout status`
* Verify near-zero downtime using a Service and `curl`
* Pause, resume, view history, and roll back Deployments safely

***

### Section A: Create a Clean Playground Namespace

Create a dedicated namespace for this lab.

```bash
kubectl create ns lab-rolling
kubectl config set-context --current --namespace=lab-rolling
kubectl get ns
```

Explanation:

* Creates a namespace named `lab-rolling`
* Sets it as the default namespace for the current context
* Verifies that the namespace exists

***

### Section B: Deploy Version 1 of NGINX and Scale It

For this lab:

* `nginx:1.25-alpine` is treated as version 1
* `nginx:1.27.2` is treated as version 2

#### Create the Deployment

Start with a single replica.

```bash
kubectl create deployment web --image=nginx:1.25-alpine
```

Verify the created resources:

```bash
kubectl get deployments
kubectl get rs
kubectl get pods -o wide
```

Explanation:

* The Deployment creates a ReplicaSet
* The ReplicaSet creates one Pod

#### Scale the Deployment to 5 Replicas

Scaling makes rolling updates easier to observe.

```bash
kubectl scale deployment web --replicas=5
kubectl get deploy,rs,pods
```

***

### Section C: Make Rolling Updates Safer with Probes and Strategy

By default, Deployments use a rolling update strategy. To make updates safer, you will:

* Limit unavailable Pods
* Limit surge Pods
* Add readiness and liveness probes

This ensures traffic is only sent to healthy Pods.

#### Patch the Existing Deployment

Apply the following patch:

```bash
kubectl patch deployment web --type merge -p '
{
  "spec": {
    "strategy": {
      "type": "RollingUpdate",
      "rollingUpdate": {
        "maxUnavailable": 1,
        "maxSurge": 1
      }
    },
    "template": {
      "spec": {
        "containers": [{
          "name": "nginx",
          "image": "nginx:1.25-alpine",
          "ports": [{"containerPort": 80}],
          "livenessProbe": {
            "httpGet": {"path": "/", "port": 80},
            "initialDelaySeconds": 5,
            "periodSeconds": 10
          },
          "readinessProbe": {
            "httpGet": {"path": "/", "port": 80},
            "initialDelaySeconds": 3,
            "periodSeconds": 5
          }
        }]
      }
    }
  }
}'
```

#### Verify the Configuration

```bash
kubectl describe deploy web | sed -n '1,120p'
```

Expected outcome:

* Rolling update strategy is visible
* Readiness and liveness probes are configured

***

### Section D: Perform a Rolling Update to Version 2

#### Update the Container Image

```bash
kubectl set image deployment/web nginx=nginx:1.27.2
```

#### Watch the Rollout Progress

```bash
kubectl rollout status deployment/web
```

#### Observe Pods Being Replaced

```bash
kubectl get pods -w
```

Expected behavior:

* New Pods are created gradually
* Old Pods are terminated only after new Pods become Ready

#### Verify the Final State

```bash
kubectl get deploy,rs,pods -o wide
kubectl get events --sort-by=.lastTimestamp | tail -n 20
```

***

### Section E: Rollout History, Pause, Resume, and Rollback

#### View Rollout History

```bash
kubectl rollout history deployment/web
```

#### Roll Back to the Previous Version

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

#### Verify the Version Using curl

If your Service or port-forwarding is already configured:

```bash
curl -I http://localhost:8080 | grep -i ^Server
```

Expected outcome:

* The `Server` header reflects the previous NGINX version

***

### Section F: Cleanup

Remove all lab resources by deleting the namespace.

```bash
kubectl delete ns lab-rolling
```

***

### Troubleshooting

#### ImagePullBackOff or ErrImagePull

* Verify the image name and tag, for example `nginx:1.27.2`
* Check network egress access
* Verify private registry credentials if applicable

***

#### Service Has No Endpoints

* Ensure Pods are in Ready state
* Ensure Pod labels match the Service selector

Useful commands:

```bash
kubectl get pods --show-labels
kubectl describe svc web-svc
```

***

#### Rollout Is Stuck

* Inspect Deployment events:

```bash
kubectl describe deploy web
```

* Check that the readiness probe is passing
* Roll back to the previous version if needed:

```bash
kubectl rollout undo deployment/web
```


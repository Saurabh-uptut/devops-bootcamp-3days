# Lab 6: Deploy a Standalone Node.js Application on EKS Cluster

You will containerize a simple Node.js application, push the image to a container registry, deploy it to Google Kubernetes Engine (GKE Cluster), and expose it using appropriate Kubernetes Service types for both internal and public access.

### Scenario

You own a small Node.js web service that exposes a `/health` endpoint.

Product requirements:

* The service must be reachable publicly over HTTP from the internet
* The service must also be reachable internally by other Pods inside the cluster
* The solution must use the correct Kubernetes Service types on AKS
* The design should be simple, maintainable, and production-appropriate

You are responsible for choosing the right Kubernetes primitives and wiring everything together cleanly.

### Code Repository

Clone the starter Node.js application from:

**Sample Node App**

(The repository contains a minimal Node.js service skeleton. You may extend it as needed.)

### Learning Objectives

By completing this lab, you will demonstrate the ability to:

* Package a Node.js application into a container image
* Deploy a containerized application to AKS using Kubernetes manifests
* Choose and implement appropriate Service types:
  * ClusterIP for internal-only access
  * LoadBalancer for public access on AKS
* Validate application health, connectivity, scaling, and rollouts

### Prerequisites

You must have the following ready before starting:

* An AKS cluster with `kubectl` configured
* Access to a container registry (Azure Container Registry or Docker Hub)
* Permissions to push images to the registry
* Node.js installed locally to run and test the app
* Basic familiarity with Kubernetes YAML and `kubectl`

If you use Azure Container Registry:

* Be prepared to authenticate using `az acr login`, managed identity, or `imagePullSecrets`

### Constraints and Ground Rules

You must follow all of these rules:

* Use a **single dedicated namespace** for this lab\
  Example: `node-challenge`
* Use **declarative Kubernetes manifests (YAML)** only
* The application **must expose a `/health` endpoint** that returns HTTP 200
* Do not use port-forwarding as the final access method\
  Port-forwarding is allowed only for debugging
* The deployment must be fully reproducible:
  * `kubectl apply -f ...` creates everything
  * `kubectl delete -f ...` cleans everything up

***

### Deliverables

Your submission must include the following artifacts.

#### Node.js Application

A minimal Node.js app (Express or similar) with:

* Support for a configurable `PORT` environment variable
* `/` endpoint returning a simple page or JSON
* `/health` endpoint returning HTTP 200 quickly

***

#### Dockerfile

* Uses a small, production-appropriate base image
* Multi-stage build is preferred
* Image must run the app on the configured `PORT`

***

#### Kubernetes Manifests

You must provide YAML manifests for:

* Namespace
* ConfigMap and or Secret for runtime configuration\
  Example: app name, banner text, environment
* Deployment:
  * 3 replicas
  * Proper labels
  * Liveness and readiness probes on `/health`
  * Resource requests and limits
  * Rolling update strategy
* ClusterIP Service for internal access
* LoadBalancer Service for public access on AKS

***

#### Documentation

A `README.md` that clearly explains:

* How to build the container image
* How to push the image to the registry
* How to deploy the application to AKS
* How to validate internal and external connectivity
* How to clean up all resources

All commands in the README must work as written.

***

### Acceptance Criteria

Your solution will be evaluated against the following checklist.

***

#### Application and Image

* Application listens on `PORT` (default 3000)
* `/health` returns HTTP 200
* Container image builds successfully
* Image is pushed to the registry
* Image tag is immutable (version or git SHA, not `latest`)

***

#### Kubernetes Resources

* Namespace exists and is used consistently
* Deployment configuration includes:
  * `replicas: 3`
  * Readiness probe on `/health`
  * Liveness probe on `/health`
  * Sensible resource requests and limits
  * Rolling update strategy (defaults acceptable)
* ClusterIP Service:
  * Selects Pods using labels
  * Provides internal-only access
* LoadBalancer Service:
  * Exposes port 80 externally
  * Forwards traffic to container port 3000
* LoadBalancer receives a valid `EXTERNAL-IP` on AKS
* Service endpoints or EndpointSlices include all Ready Pods
* `kubectl rollout status` completes successfully

***

#### Verification

You must be able to demonstrate:

* External access from your machine:
  * `curl http://<EXTERNAL-IP>/health` returns HTTP 200
* Internal access from within the cluster:
  * Access via the ClusterIP Service returns HTTP 200
* Scaling behavior:
  * Scaling replicas increases endpoints
  * Application remains reachable
* Rollout behavior:
  * Updating the image tag triggers a rollout
  * Rollout completes with no perceived downtime

***

### High-Level Tasks

You are expected to complete the following, without step-by-step guidance.

1. Author the Node.js application
2. Containerize the application
3. Push the image to a registry
4. Create Kubernetes namespace and configuration
5. Deploy the application using a Deployment
6. Expose the application internally and publicly using Services
7. Validate connectivity, scaling, and rollouts
8. Clean up all resources

***

### Cleanup Requirement

Your cleanup process must remove everything created by this lab, including the namespace.

***

### Hints (Only If You Get Stuck)

* External IP stuck in `<pending>`\
  Check Service events and ensure AKS cloud integration is working
* Service has no endpoints\
  Check Pod labels and readiness state
* Rollout is stuck\
  Inspect Deployment events and probe configuration\
  Roll back if necessary

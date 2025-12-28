# Lab 1 - Get hands-on with Kubectl and launch your first pod

In this lab, you will practice essential `kubectl` commands and create your first Kubernetes Pod using the official NGINX container image.

***

### Prerequisites

You need the following on your Linux, Windows, or macOS machine:

* A running Kubernetes cluster\
  Examples: Minikube, kind, Docker Desktop, k3s, or any managed Kubernetes service
* `kubectl` installed
* `kubectl` configured to point to your cluster using kubeconfig

***

### Learning Objectives

By the end of this lab, you will be able to:

* Explore a Kubernetes cluster using `kubectl`
* Create and inspect a Pod running the NGINX container
* View logs and execute commands inside a container
* Add labels and annotations to resources
* Clean up resources safely

***

### Section A: Quick Cluster Health Check

Verify that your Kubernetes cluster is running and accessible.

Run the following commands:

```bash
kubectl version
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns
```

Expected outcome:

* At least one node should be in the `Ready` state.
* If you see a Ready node, your cluster is ready for this lab.

***

### Section B: Create a Playground Namespace

Use a dedicated namespace to keep lab resources isolated.

1. Create a new namespace:

```bash
kubectl create ns lab-nginx
```

2. Set the current context to use this namespace by default:

```bash
kubectl config set-context --current --namespace=lab-nginx
```

3. Verify the namespace:

```bash
kubectl get ns
```

Explanation:

* `kubectl create ns lab-nginx` creates a namespace named `lab-nginx`.
* `kubectl config set-context --current --namespace=lab-nginx` sets `lab-nginx` as the default namespace.
* `kubectl get ns` lists all namespaces in the cluster.

Optional check:

```bash
kubectl config view --minify -o jsonpath='{..namespace}{"\n"}'
```

To switch back to the default namespace later:

```bash
kubectl config set-context --current --namespace=default
```

***

### Section C: Create Your First Pod (NGINX)

Create a standalone Pod running the NGINX image.

```bash
kubectl run webpod --image=nginx:latest --restart=Never
```

Explanation:

* Creates a Pod named `webpod`.
* Uses the `nginx:latest` image.
* `--restart=Never` ensures Kubernetes does not create a Deployment or ReplicaSet.

#### Verify the Pod

Run the following commands:

```bash
kubectl get pods
kubectl get pod webpod -o wide
kubectl describe pod webpod
kubectl get events --sort-by=.lastTimestamp | tail -n 20
```

Explanation:

* `kubectl get pods` lists all Pods in the namespace.
* `kubectl get pod webpod -o wide` shows extended details such as node and Pod IP.
* `kubectl describe pod webpod` shows detailed information, including events and container status.
* `kubectl get events --sort-by=.lastTimestamp | tail -n 20` shows the most recent events.

Common Pod statuses:

* `ContainerCreating`: Image is being pulled or container is starting.
* `Running`: Pod is running successfully.
* `ImagePullBackOff` or `ErrImagePull`: Image name or network issue.

***

### Section D: Logs, Shell Access, and File Inspection

#### View Pod Logs

```bash
kubectl logs webpod
kubectl logs -f webpod
```

Explanation:

* `kubectl logs webpod` prints logs and exits.
* `kubectl logs -f webpod` streams logs in real time until you stop it.

Note:\
If the Pod has multiple containers, add `-c <container-name>`.

#### Execute a Shell Inside the Container

```bash
kubectl exec -it webpod -- /bin/sh
```

Inside the container, run:

```bash
ls /usr/share/nginx/html
exit
```

Explanation:

* Opens an interactive shell inside the container.
* Allows you to explore files and run commands.
* Use `exit` to leave the container shell.

***

### Section E: Useful kubectl Commands

#### View Detailed Pod Information

```bash
kubectl get pod -o wide
kubectl get pod webpod -o jsonpath='{.status.containerStatuses[0].ready}{"\n"}'
```

Explanation:

* Shows detailed Pod information including node and IP.
* Prints whether the first container in the Pod is ready (`true` or `false`).

***

#### Add Labels and Annotations

```bash
kubectl label pod webpod tier=frontend env=dev
kubectl annotate pod webpod owner="dev-team"
```

Verify labels and filters:

```bash
kubectl get pods --show-labels
kubectl get pods -l tier=frontend,env=dev
```

Explanation:

* Labels are used for selection and grouping.
* Annotations store metadata and are not used for selection.
* Filtering with `-l` uses logical AND between labels.

***

#### Explore Kubernetes Resource Documentation

```bash
kubectl explain pod
kubectl explain pod.spec.containers
```

Explanation:

* Shows built-in Kubernetes API documentation.
* Helps you understand available fields and structure.

Optional deep view:

```bash
kubectl explain pod.spec.containers --recursive
```

***

#### View Raw YAML and JSON Output

```bash
kubectl get pod webpod -o yaml | head -n 30
kubectl get pod webpod -o jsonpath='{.status.phase}{"\n"}'
```

Explanation:

* YAML output is useful for learning resource structure.
* JSONPath extracts specific fields for scripting or automation.

***

#### Check Events and Resources

```bash
kubectl get events --sort-by=.lastTimestamp | tail -n 30
kubectl get all
```

Explanation:

* Displays recent events in the current namespace.
* Lists all common resource types such as Pods, Services, and ReplicaSets.


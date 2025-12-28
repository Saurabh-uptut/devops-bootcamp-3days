# Lab 8 - Intelligent Pod Placement and Scheduling

#### Topics

Node Selector, Node Affinity, Taints and Tolerations, Resource Requests and Limits, LimitRange, DaemonSets

In this lab, you will learn how to control where Pods run on an Azure Kubernetes Service (AKS) cluster and how they consume CPU and memory. You will observe scheduling decisions using `kubectl get pods -o wide` and `kubectl describe`.

***

### Learning Objectives

By the end of this lab, you will be able to:

* Use `nodeSelector` to target a specific node or node pool
* Write node affinity rules using required and preferred constraints
* Apply a taint to a node and schedule Pods using tolerations
* Configure resource requests and limits and apply namespace defaults using a LimitRange
* Deploy a DaemonSet that runs one Pod per node

***

### Prerequisites

You need the following:

* An AKS cluster with `kubectl` configured
* At least one user node pool
* Permission to label and taint nodes
* A terminal with access to run `kubectl`

***

### Section 0: Create a Dedicated Namespace

Create a namespace for the lab and set it as the default namespace for your current context.

```bash
kubectl create namespace sched-lab
kubectl config set-context --current --namespace=sched-lab
```

***

### Section 1: Inspect Nodes and Identify Labels

View the nodes and note their names and labels.

```bash
kubectl get nodes -o wide
kubectl get nodes -o jsonpath='{.items[*].metadata.name}{"\n"}'
kubectl get nodes --show-labels
```

Find the value of the label `kubernetes.io/hostname` on any node. You will use that value in Part A.

***

### Part A: Node Selector

#### Goal

Schedule a Pod only on a specific node by matching a label.

#### Task

1. Replace `<HOSTNAME>` in the manifest below with a real value from your cluster, taken from `kubernetes.io/hostname`.

Create `A1-nginx-nodeselector.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ns
spec:
  nodeSelector:
    kubernetes.io/hostname: <HOSTNAME>
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
```

2. Apply and verify:

```bash
kubectl apply -f A1-nginx-nodeselector.yml
kubectl get pod nginx-ns -o wide
kubectl describe pod nginx-ns
```

#### What to Verify

* The `NODE` column shows the targeted node
* The Pod schedules successfully only if the label matches a Ready node

***

### Part B: Node Affinity

#### Goal

Use preferred and required node affinity rules to influence scheduling.

#### Step 1: Label One Node

Choose a worker node and apply a custom label.

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node $NODE disktype=ssd --overwrite
kubectl get nodes --show-labels | grep $NODE
```

***

#### Step 2: Preferred Node Affinity

Create `B1-busybox-affinity-preferred.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-affinity-preferred
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: bb
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Apply and verify:

```bash
kubectl apply -f B1-busybox-affinity-preferred.yml
kubectl get pod busybox-affinity-preferred -o wide
kubectl describe pod busybox-affinity-preferred
```

What to verify:

* The Pod should prefer the labeled node if capacity allows
* It may schedule elsewhere if the preferred node is not available

***

#### Step 3: Required Node Affinity

Create `B2-busybox-affinity-required.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-affinity-required
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: bb
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Apply and verify:

```bash
kubectl apply -f B2-busybox-affinity-required.yml
kubectl get pod busybox-affinity-required -o wide
kubectl describe pod busybox-affinity-required
```

What to verify:

* If no Ready node has `disktype=ssd`, the Pod remains Pending
* Events in `kubectl describe` explain why scheduling failed

***

### Part C: Taints and Tolerations

#### Goal

Prevent normal Pods from scheduling onto a special node, then allow a Pod that tolerates the taint.

#### Step 1: Taint the Labeled Node

Apply a NoSchedule taint:

```bash
kubectl taint nodes $NODE workload=critical:NoSchedule
kubectl describe node $NODE | grep -i Taints -A2
```

***

#### Step 2: Create a Pod Without Toleration

Create `C1-no-toleration.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-no-toleration
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
```

***

#### Step 3: Create a Pod With Toleration

Create `C2-with-toleration.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-toleration
spec:
  tolerations:
    - key: workload
      operator: Equal
      value: critical
      effect: NoSchedule
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
```

Apply and verify:

```bash
kubectl apply -f C1-no-toleration.yml
kubectl apply -f C2-with-toleration.yml
kubectl get pods -o wide
```

What to verify:

* `web-no-toleration` should not schedule onto the tainted node
* `web-with-toleration` can schedule onto it

Note:\
In small clusters, the non-tolerating Pod may still schedule successfully on other nodes. That is expected.

***

### Part D: Resource Requests and Limits and Namespace Defaults

#### Goal

Control CPU and memory per container and set default values using a LimitRange.

#### Step 1: Apply LimitRange Defaults

Create `D1-limitrange.yml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resources
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "250m"
        memory: "128Mi"
```

Apply and inspect:

```bash
kubectl apply -f D1-limitrange.yml
kubectl describe limitrange default-resources
```

***

#### Step 2: Pod With Explicit Resources

Create `D2-explicit-resources.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-mem-explicit
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          cpu: "300m"
          memory: "200Mi"
        limits:
          cpu: "600m"
          memory: "400Mi"
```

***

#### Step 3: Pod Without Resources

Create `D3-defaulted-resources.yml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-mem-defaulted
spec:
  containers:
    - name: bb
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Apply and verify:

```bash
kubectl apply -f D2-explicit-resources.yml
kubectl apply -f D3-defaulted-resources.yml
kubectl describe pod cpu-mem-explicit | egrep -i "Requests|Limits|QoS"
kubectl describe pod cpu-mem-defaulted | egrep -i "Requests|Limits|QoS"
```

What to verify:

* The explicit Pod shows your values
* The defaulted Pod shows LimitRange defaults
* Observe the QoS class and discuss scheduling impact

***

### Part E: DaemonSet

#### Goal

Run one Pod per node, including on tainted nodes by using tolerations.

#### Task

Create `E1-daemonset.yml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: hello-daemon
spec:
  selector:
    matchLabels:
      app: hello-daemon
  template:
    metadata:
      labels:
        app: hello-daemon
    spec:
      tolerations:
        - key: workload
          operator: Equal
          value: critical
          effect: NoSchedule
      containers:
        - name: hello
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo $(hostname); sleep 30; done"]
```

Apply and verify:

```bash
kubectl apply -f E1-daemonset.yml
kubectl get ds hello-daemon
kubectl get pods -l app=hello-daemon -o wide
```

What to verify:

* You see one DaemonSet Pod per node
* DaemonSet Pods can run on the tainted node due to toleration

Optional extension:

* Target only a specific node pool by adding a `nodeSelector` to the Pod template.

***

### Cleanup

Delete the namespace and optionally remove the node label and taint.

```bash
kubectl delete namespace sched-lab
```

Optional cleanup of node changes:

```bash
kubectl label node $NODE disktype-
kubectl taint nodes $NODE workload=critical:NoSchedule-
```

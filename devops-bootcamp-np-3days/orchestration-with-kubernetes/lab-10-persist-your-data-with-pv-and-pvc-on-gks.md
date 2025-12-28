# Lab 10: Persist Your Data with PV and PVC on AWS EKS (Amazon EFS Dynamic Provisioning)

In this lab, you will move PostgreSQL storage from node-local behavior to **Amazon EFS using dynamic provisioning** on **Amazon EKS**.
You will not create a PersistentVolume manually. A **PVC** will trigger EFS-backed PV creation through the **EFS CSI driver**.

This is the AWS equivalent of using Filestore on GKE with RWX volumes.

---

## What stays the same

* Backend and frontend manifests remain unchanged
* PostgreSQL Deployment still mounts a PVC at:

  ```
  /var/lib/postgresql/data
  ```

---

## What changes

1. Install and configure **Amazon EFS CSI Driver**
2. Create an **EFS-backed StorageClass** with `ReadWriteMany`
3. Replace the existing DB PVC with an EFS dynamic PVC
4. Verify persistence across pod restarts

---

## Prerequisites

* Amazon EKS cluster running
* `kubectl` configured for the cluster
* Namespace `user-management` already exists
* PostgreSQL deployment from previous lab
* AWS CLI configured
* IAM permissions for:

  * EFS
  * EC2
  * IAM
  * EKS

---

## Step 0: Install Amazon EFS CSI Driver (one-time per cluster)

### 0.1 Associate IAM OIDC provider with the cluster

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <CLUSTER_NAME> \
  --region <AWS_REGION> \
  --approve
```

---

### 0.2 Create IAM policy for EFS CSI driver

```bash
curl -o efs-csi-policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
```

```bash
aws iam create-policy \
  --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
  --policy-document file://efs-csi-policy.json
```

---

### 0.3 Create IAM service account for the driver

```bash
eksctl create iamserviceaccount \
  --cluster <CLUSTER_NAME> \
  --region <AWS_REGION> \
  --namespace kube-system \
  --name efs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKS_EFS_CSI_Driver_Policy \
  --approve
```

---

### 0.4 Install EFS CSI Driver using Helm

```bash
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
```

```bash
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=efs-csi-controller-sa
```

Verify:

```bash
kubectl get pods -n kube-system | grep efs
```

---

## Step 1: Create Amazon EFS File System

### 1.1 Create EFS file system

```bash
aws efs create-file-system \
  --region <AWS_REGION> \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --tags Key=Name,Value=eks-postgres-efs
```

Save the returned `FileSystemId`.

---

### 1.2 Create mount targets in all worker node subnets

List subnets used by EKS worker nodes:

```bash
aws ec2 describe-subnets \
  --filters Name=tag:eks:cluster-name,Values=<CLUSTER_NAME> \
  --query 'Subnets[*].SubnetId' \
  --output text
```

For each subnet:

```bash
aws efs create-mount-target \
  --file-system-id <EFS_ID> \
  --subnet-id <SUBNET_ID> \
  --security-groups <NODE_SECURITY_GROUP_ID>
```

---

## Step 2: Create EFS StorageClass

Create `efs-storageclass.yml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-rwx
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: <EFS_ID>
  directoryPerms: "700"
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

Apply:

```bash
kubectl apply -f efs-storageclass.yml
kubectl get storageclass
```

---

## Step 3: Replace DB PVC with EFS Dynamic PVC

Replace your existing `db-pvc.yml` with the following.

### New `db-pvc.yml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: user-management
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-rwx
  resources:
    requests:
      storage: 5Gi
```

Apply:

```bash
kubectl apply -f db-pvc.yml
kubectl get pvc -n user-management
```

Note: With EFS, storage size is logical. The request does not limit actual usage.

---

## Step 4: PostgreSQL Deployment

No change required in your DB Deployment.

Ensure this section exists in `db-deploy.yml`:

```yaml
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-pvc
```

Apply DB resources:

```bash
kubectl apply -f db-secret.yml
kubectl apply -f db-deploy.yml
kubectl apply -f db-svc.yml
```

---

## Step 5: Verification

### 5.1 Check PVC and PV

```bash
kubectl get pvc -n user-management
kubectl get pv | grep postgres-pvc || true
```

PVC should be `Bound`.

---

### 5.2 Confirm EFS volume is mounted

```bash
kubectl describe pod -n user-management -l app=db | sed -n '/Volumes:/,/Events:/p'
```

You should see `efs.csi.aws.com`.

---

### 5.3 Validate persistence across pod restart

```bash
kubectl exec -n user-management deploy/postgres-db -- \
  sh -c 'echo hello-$(date) >> /var/lib/postgresql/data/persist.txt && tail -n 3 /var/lib/postgresql/data/persist.txt'
```

Delete the pod:

```bash
kubectl delete pod -n user-management -l app=db
kubectl get pods -n user-management -w
```

After the pod is recreated:

```bash
kubectl exec -n user-management deploy/postgres-db -- \
  sh -c 'tail -n 3 /var/lib/postgresql/data/persist.txt'
```

The data must still be present.

---

## Key takeaways

* Amazon EFS provides `ReadWriteMany` volumes for EKS
* Dynamic provisioning removes the need to manage PVs manually
* EFS is ideal for shared or stateful workloads across pods and nodes
* This mirrors Filestore behavior on GKE using AWS-native services

---

## Cleanup (optional)

If you no longer need the storage:

```bash
kubectl delete pvc postgres-pvc -n user-management
aws efs delete-file-system --file-system-id <EFS_ID>
```

Only delete EFS after confirming no workloads depend on it.
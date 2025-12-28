# Lab 7: Provision an AWS Elastic Kubernetes Service Cluster using Terraform

## Overview

In this lab, you will use Terraform to provision an **Amazon EKS** cluster and a **managed node group**. You will then fetch cluster credentials and verify access using `kubectl`.

---

## Learning objectives

1. Configure Terraform for AWS
2. Create an EKS cluster using Terraform
3. Create a managed node group for worker nodes
4. Fetch kubeconfig and validate cluster access
5. Destroy the infrastructure using Terraform

---

## Prerequisites

* AWS account
* Terraform installed
* AWS CLI installed and configured
* kubectl installed
* VS Code (or any IDE)
* IAM permissions to create: VPC, Subnets, Route Tables, Internet Gateway, IAM Roles/Policies, EKS Cluster, EKS Node Group

Validate AWS access:

```bash
aws sts get-caller-identity
```

---

## Step 0: Confirm required AWS tooling

Check these commands work:

```bash
aws --version
kubectl version --client=true
terraform -version
```

---

## Step 1: Create a new Terraform project folder

```bash
mkdir eks_with_terraform
cd eks_with_terraform
```

Create these files:

* providers.tf
* variables.tf
* terraform.tfvars
* network.tf
* iam.tf
* eks.tf
* outputs.tf

---

## Step 2: Configure provider in `providers.tf`

Create `providers.tf`:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0.0, < 7.0.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

---

## Step 3: Define variables in `variables.tf`

Create `variables.tf`:

```hcl
variable "region" {
  type        = string
  description = "AWS region to deploy into"
  default     = "us-east-1"
}

variable "cluster_name" {
  type        = string
  description = "EKS cluster name"
  default     = "eks-cluster-1"
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR for the VPC"
  default     = "10.10.0.0/16"
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "Two public subnet CIDRs in different AZs"
  default     = ["10.10.1.0/24", "10.10.2.0/24"]
}

variable "node_instance_type" {
  type        = string
  description = "EC2 instance type for worker nodes"
  default     = "t3.medium"
}

variable "node_desired_size" {
  type        = number
  description = "Desired node count"
  default     = 2
}

variable "node_min_size" {
  type        = number
  description = "Minimum node count"
  default     = 1
}

variable "node_max_size" {
  type        = number
  description = "Maximum node count"
  default     = 3
}
```

---

## Step 4: Provide values in `terraform.tfvars`

Create `terraform.tfvars`:

```hcl
region       = "us-east-1"
cluster_name = "eks-saurabh"

vpc_cidr = "10.10.0.0/16"

public_subnet_cidrs = ["10.10.1.0/24", "10.10.2.0/24"]

node_instance_type = "t3.medium"
node_desired_size  = 2
node_min_size      = 1
node_max_size      = 3
```

---

## Step 5: Create VPC and public subnets in `network.tf`

Create `network.tf`:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "eks_vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.cluster_name}-vpc"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.eks_vpc.id

  tags = {
    Name = "${var.cluster_name}-igw"
  }
}

resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.eks_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${var.cluster_name}-public-rt"
  }
}

resource "aws_subnet" "public" {
  for_each = {
    for idx, cidr in var.public_subnet_cidrs :
    idx => {
      cidr = cidr
      az   = data.aws_availability_zones.available.names[idx]
    }
  }

  vpc_id                  = aws_vpc.eks_vpc.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.cluster_name}-public-subnet-${each.key}"
  }
}

resource "aws_route_table_association" "public_assoc" {
  for_each = aws_subnet.public

  subnet_id      = each.value.id
  route_table_id = aws_route_table.public_rt.id
}
```

Notes:

* EKS requires subnets in at least two AZs for high availability.
* This lab uses public subnets for simplicity.

---

## Step 6: Create IAM roles for EKS in `iam.tf`

Create `iam.tf`:

```hcl
resource "aws_iam_role" "eks_cluster_role" {
  name = "${var.cluster_name}-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  role       = aws_iam_role.eks_cluster_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role" "eks_node_role" {
  name = "${var.cluster_name}-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  role       = aws_iam_role.eks_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  role       = aws_iam_role.eks_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "ecr_readonly" {
  role       = aws_iam_role.eks_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}
```

---

## Step 7: Create EKS cluster and node group in `eks.tf`

Create `eks.tf`:

```hcl
resource "aws_eks_cluster" "eks" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster_role.arn

  vpc_config {
    subnet_ids              = [for s in aws_subnet.public : s.id]
    endpoint_public_access  = true
    endpoint_private_access = false
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy
  ]
}

resource "aws_eks_node_group" "primary" {
  cluster_name    = aws_eks_cluster.eks.name
  node_group_name = "${var.cluster_name}-primary-ng"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = [for s in aws_subnet.public : s.id]

  instance_types = [var.node_instance_type]

  scaling_config {
    desired_size = var.node_desired_size
    min_size     = var.node_min_size
    max_size     = var.node_max_size
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ecr_readonly
  ]
}
```

---

## Step 8: Add outputs in `outputs.tf`

Create `outputs.tf`:

```hcl
output "cluster_name" {
  value       = aws_eks_cluster.eks.name
  description = "EKS cluster name"
}

output "cluster_region" {
  value       = var.region
  description = "AWS region"
}

output "cluster_endpoint" {
  value       = aws_eks_cluster.eks.endpoint
  description = "EKS API server endpoint"
}
```

---

## Step 9: Run the Terraform workflow

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

---

## Step 10: Fetch kubeconfig and connect to the cluster

Update kubeconfig:

```bash
aws eks update-kubeconfig --name eks-saurabh --region us-east-1
```

Verify access:

```bash
kubectl get nodes
kubectl get namespaces
```

If nodes are in Ready state, the cluster is working.

---

## Step 11: Optional verification by deploying a test app

Deploy:

```bash
kubectl create deployment nginx --image=nginx
kubectl get pods
```

Clean up the test deployment:

```bash
kubectl delete deployment nginx
```

---

## Step 12: Destroy the infrastructure

```bash
terraform destroy --auto-approve
```

---

## Common troubleshooting

* Quota or capacity issues: reduce `node_desired_size` or switch `node_instance_type`
* Access denied errors: your IAM user may lack EKS, IAM, or VPC permissions
* kubeconfig issues: confirm your AWS CLI profile and region match the cluster
* Nodes not joining: verify IAM node role attachments and that subnets were created in two AZs
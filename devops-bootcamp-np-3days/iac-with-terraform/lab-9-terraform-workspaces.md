# Lab 9: Terraform Workspaces on AWS

In this lab, you will practice Terraform workspaces to keep separate state files (and therefore separate environments) inside the same codebase, using AWS as the target cloud.

---

## Learning Objectives

* Create, list, select, and delete Terraform workspaces
* Manage multiple non-overlapping environments from the same Terraform configuration
* Maintain multiple `terraform.tfstate` files using workspaces

---

## Prerequisites

* Terraform installed
* AWS CLI installed and configured (`aws configure`)
* A working AWS Terraform codebase (example in this lab: an `aws_instance` EC2 project)
* IAM permissions to create and destroy EC2 instances (and related networking defaults)

Validate AWS access:

```bash
aws sts get-caller-identity
```

---

## Lab Setup (Example AWS Project)

If you already have an AWS Terraform project, you can use it directly.
If not, use this minimal EC2 example.

### Create project folder

```bash
mkdir terraform-workspaces-aws
cd terraform-workspaces-aws
```

Create the following files:

* `providers.tf`
* `variables.tf`
* `main.tf`

### `providers.tf`

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

### `variables.tf`

```hcl
variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

variable "instance_name" {
  type        = string
  description = "EC2 Name tag"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}
```

### `main.tf`

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

data "aws_security_group" "default" {
  vpc_id = data.aws_vpc.default.id
  name   = "default"
}

data "aws_ami" "ubuntu_2404" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "example" {
  ami                         = data.aws_ami.ubuntu_2404.id
  instance_type               = var.instance_type
  subnet_id                    = data.aws_subnets.default_subnets.ids[0]
  vpc_security_group_ids       = [data.aws_security_group.default.id]
  associate_public_ip_address  = true

  tags = {
    Name        = var.instance_name
    Environment = terraform.workspace
  }
}
```

---

# Step 0: Prepare the project

Format and initialize:

```bash
terraform fmt
terraform init
```

---

# Part 1: Default Workspace

## 1) List workspaces

```bash
terraform workspace list
```

You should see:

* `default` (current)

## 2) Plan and apply in default workspace

Create a simple `terraform.tfvars` for default.

Create `terraform.tfvars`:

```hcl
region        = "us-east-1"
instance_name = "ec2-saurabh-default"
instance_type = "t3.micro"
```

Run:

```bash
terraform plan
terraform apply --auto-approve
```

Result:

* An EC2 instance is created on AWS
* A local state file is created: `terraform.tfstate`
* This is the state for the default workspace

---

# Part 2: Create and Use a “development” Workspace

## 3) Create a new workspace

```bash
terraform workspace new development
```

## 4) Confirm current workspace

```bash
terraform workspace list
```

You will notice:

* `development` is selected
* A folder appears: `terraform.tfstate.d/development/`
* This is where the `development` workspace state lives

---

# Part 3: Use a Separate tfvars file per workspace

## 5) Create `dev.terraform.tfvars`

Create `dev.terraform.tfvars`:

```hcl
region        = "us-east-1"
instance_name = "ec2-saurabh-dev"
instance_type = "t3.micro"
```

## 6) Plan/apply in development using var-file

```bash
terraform plan -var-file=dev.terraform.tfvars
terraform apply -var-file=dev.terraform.tfvars --auto-approve
```

Result:

* A separate state file is created under:

  * `terraform.tfstate.d/development/terraform.tfstate`
* Now you have two environments managed from the same code:

  * default workspace
  * development workspace

---

# Part 4: Destroy Resources Cleanly (Workspace by Workspace)

## 7) Destroy development resources

Make sure you are in development:

```bash
terraform workspace list
```

Destroy using the same var-file:

```bash
terraform destroy -var-file=dev.terraform.tfvars --auto-approve
```

## 8) Switch to default and destroy default resources

```bash
terraform workspace select default
terraform destroy --auto-approve
```

---

# Part 5: Delete the workspace

## 9) Delete development workspace

You must be in default before deleting development:

```bash
terraform workspace select default
terraform workspace delete development
```
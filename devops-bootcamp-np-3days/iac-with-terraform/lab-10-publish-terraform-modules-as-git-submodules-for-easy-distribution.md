# Lab 10: Publish Terraform Modules as Git Submodules for Easy Distribution (AWS)

> NOTE
> This lab continues from “Provision Infrastructure using Reusable Modules” on AWS. Complete that lab first so you already have your AWS modules folder (for example: `AWS/aws_vpc`, `AWS/aws_subnet`, `AWS/aws_security_group`, `AWS/aws_ec2_instance`).

---

## Lab Overview

You will:

* Publish Terraform modules to GitHub
* Consume those modules using Git submodules
* Use those modules in a Terraform consumer project on AWS

---

## What is a Git Submodule?

A Git submodule is a Git repository embedded inside another Git repository. It lets you:

* Reuse shared Terraform modules
* Lock the consumer project to a specific module commit
* Update modules independently and intentionally

---

# TASK 1: Publish Terraform Modules Repository (AWS)

## Step 1: Create a GitHub repository

Create a repository (public or private), for example:

* `terraform-aws-modules`

Copy the repository URL.

## Step 2: Push your modules from the `AWS/` folder

From the parent directory where `AWS/` exists:

```bash
cd AWS

git config --global user.name "<your_name>"
git config --global user.email "<your_email>"

git init
git add .
git commit -m "Initial AWS Terraform modules"
git branch -M main
git remote add origin <your_repository_url>
git push -u origin main
```

Result:

* Your AWS Terraform modules are now available in GitHub as a standalone repository.

---

# TASK 2: Use Modules via Git Submodule in a New Consumer Project

## Step 1: Create a consumer project

```bash
mkdir -p module_example_aws/example01
cd module_example_aws/example01
git init
```

## Step 2: Add the modules repo as a submodule

Add your modules repository as a submodule folder named `AWS`:

```bash
git submodule add <terraform-modules-repo-url> AWS
git submodule status
```

This creates:

* `AWS/` directory (submodule code)
* `.gitmodules`

---

## Important: Cloning later (students often miss this)

If someone clones `module_example_aws` later, they must do either:

```bash
git clone --recurse-submodules <consumer-repo-url>
```

Or after cloning:

```bash
git submodule update --init --recursive
```

---

## Project Structure

Your consumer project should look like this:

```
module_example_aws/
└── example01/
    ├── AWS/            # Git submodule
    ├── main.tf
    ├── locals.tf
    ├── providers.tf
    ├── ssh_key.tf
    └── outputs.tf
```

---

# Terraform Configuration (Consumer Project)

## providers.tf

Create `providers.tf`:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0.0, < 7.0.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = ">= 4.0.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

---

## ssh_key.tf (Generate SSH key + register AWS key pair)

Create `ssh_key.tf`:

```hcl
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "lab_key" {
  key_name   = "lab10-key"
  public_key = tls_private_key.ssh_key.public_key_openssh
}

output "tls_private_key" {
  value     = tls_private_key.ssh_key.private_key_pem
  sensitive = true
}
```

---

## locals.tf

Create `locals.tf`:

```hcl
locals {
  vpc = {
    name       = "vpc-saurabh"
    cidr_block = "10.0.0.0/16"
  }

  subnets = {
    public-subnet = {
      cidr_block = "10.2.0.0/24"
      az         = "us-east-1a"
      public     = true
    }
    private-subnet = {
      cidr_block = "10.3.0.0/24"
      az         = "us-east-1b"
      public     = false
    }
  }

  ingress_rules = [
    {
      description = "SSH"
      protocol    = "tcp"
      from_port   = 22
      to_port     = 22
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "HTTP"
      protocol    = "tcp"
      from_port   = 80
      to_port     = 80
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]

  instances = {
    public-vm = {
      subnet        = "public-subnet"
      public_ip     = true
      instance_type = "t3.micro"
      tags          = { environment = "dev" }
    }
    private-vm = {
      subnet        = "private-subnet"
      public_ip     = false
      instance_type = "t3.micro"
      tags          = { environment = "dev" }
    }
  }
}
```

---

## main.tf (Consume modules from the submodule folder)

Create `main.tf`:

```hcl
module "vpc" {
  source = "./AWS/aws_vpc"

  name       = local.vpc.name
  cidr_block = local.vpc.cidr_block
}

module "subnet" {
  for_each = local.subnets
  source   = "./AWS/aws_subnet"

  name              = each.key
  vpc_id            = module.vpc.vpc_id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az
  public            = each.value.public
}

module "security_group" {
  source = "./AWS/aws_security_group"

  name    = "allow-ssh-http"
  vpc_id  = module.vpc.vpc_id
  ingress = local.ingress_rules
}

module "ec2" {
  for_each = local.instances
  source   = "./AWS/aws_ec2_instance"

  name               = each.key
  instance_type      = each.value.instance_type
  subnet_id          = module.subnet[each.value.subnet].subnet_id
  security_group_ids = [module.security_group.security_group_id]
  key_name           = aws_key_pair.lab_key.key_name
  public_ip          = each.value.public_ip
  tags               = each.value.tags
}
```

Important:

* The `source` points to `./AWS/...` which is the checked out Git submodule.

---

## outputs.tf

Create `outputs.tf`:

```hcl
output "public_ips" {
  value = {
    for k, m in module.ec2 :
    k => m.public_ip
  }
}

output "private_ips" {
  value = {
    for k, m in module.ec2 :
    k => m.private_ip
  }
}
```

---

# Provision Infrastructure

From `module_example_aws/example01`:

```bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply --auto-approve
```

---

# Destroy Infrastructure

```bash
terraform destroy --auto-approve
```

---

# Updating the submodule (correct workflow)

## Option A: Update to latest commit on main

```bash
cd AWS
git checkout main
git pull origin main
cd ..

git add AWS
git commit -m "Bump terraform modules submodule"
git push
```

## Option B: Pin to a specific commit or tag (recommended for enterprises)

```bash
cd AWS
git fetch --tags
git checkout v1.0.0   # or a specific commit SHA
cd ..

git add AWS
git commit -m "Pin modules to v1.0.0"
git push
```

---

## When to Use Git Submodules vs Terraform Registry

* Git submodules: internal modules, strict pinning, controlled upgrades, private environments
* Terraform Registry: easier consumption, better UX, semantic versioning and documentation support
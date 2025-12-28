# Lab 4: Practice Count and For\_Each Meta-Arguments in Terraform

### Overview

In this lab, you will learn how to create multiple **Amazon EC2 instances** using a single Terraform resource block.

Terraform provides two meta-arguments for this purpose:

* `count`
* `for_each`

You will implement three scenarios:

1. Create multiple EC2 instances using a fixed `count`
2. Create multiple EC2 instances using `count` with a list
3. Create multiple EC2 instances using `for_each`

***

### Learning Objectives

By the end of this lab, you will be able to:

1. Create multiple AWS resources using Terraform meta-arguments
2. Understand how `count` works and when to use it
3. Understand how `for_each` works and when to use it
4. Compare `count` vs `for_each` for real infrastructure use cases

***

### Pre-Requisites

* AWS account with valid credentials
* Terraform installed locally
* AWS CLI installed locally and configured (`aws configure`)
* VS Code or any similar IDE
* IAM permissions to create EC2 instances

Validate AWS access:

```bash
aws sts get-caller-identity
```

***

### Notes (AWS specifics)

* This lab uses the **default VPC** and its default subnets.
* Instead of hardcoding an AMI ID, we fetch the **latest Ubuntu 24.04 LTS** AMI for the chosen region using a Terraform data source.
* Instances will be launched in the first available default subnet.

***

## PART 1: Using `count` with a Fixed Number

### Task 1: Provision Multiple EC2 Instances Using Fixed `count`

#### Step 1: Create Project Directory

```bash
mkdir count_example_aws
cd count_example_aws
```

***

#### Step 2: Provider Configuration (`providers.tf`)

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

***

#### Step 3: Define Variables (`variables.tf`)

Create `variables.tf`:

```hcl
variable "region" {
  type        = string
  description = "AWS region to deploy into"
  default     = "us-east-1"
}

variable "compute_name" {
  type        = string
  description = "Base name of the EC2 instances"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

variable "tags" {
  type        = map(string)
  description = "Common tags applied to instances"
  default     = {
    environment = "dev"
  }
}
```

***

#### Step 4: Create EC2 Instances (`ec2_instance.tf`)

Create `ec2_instance.tf`:

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default_vpc_subnets" {
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

resource "aws_instance" "compute" {
  count = 3

  ami                    = data.aws_ami.ubuntu_2404.id
  instance_type           = var.instance_type
  subnet_id               = data.aws_subnets.default_vpc_subnets.ids[0]
  vpc_security_group_ids  = [data.aws_security_group.default.id]
  associate_public_ip_address = true

  tags = merge(
    var.tags,
    {
      Name = "${var.compute_name}-${count.index}"
    }
  )
}
```

Explanation:

* `count = 3` creates three EC2 instances
* `count.index` generates unique names like:
  * compute-saurabh-0
  * compute-saurabh-1
  * compute-saurabh-2

***

#### Step 5: Provide Values (`terraform.tfvars`)

Create `terraform.tfvars`:

```hcl
region       = "us-east-1"
compute_name = "compute-saurabh"
instance_type = "t3.micro"

tags = {
  environment = "dev"
  owner       = "saurabh"
}
```

***

#### Step 6: Terraform Workflow

Run:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

Verify in AWS Console:

* EC2 -> Instances
* Confirm 3 instances exist with your naming pattern

***

#### Step 7: Destroy Resources

```bash
terraform destroy --auto-approve
```

***

## PART 2: Using `count` with a List

### Task 2: Provision Multiple EC2 Instances by Iterating Over a List

#### Step 1: Create Project Directory

```bash
mkdir count_example_2_aws
cd count_example_2_aws
```

***

#### Step 2: Provider Configuration (`providers.tf`)

Use the same `providers.tf` as Part 1.

***

#### Step 3: Define Variables (`variables.tf`)

Create `variables.tf`:

```hcl
variable "region" {
  type        = string
  default     = "us-east-1"
  description = "AWS region to deploy into"
}

variable "compute_names" {
  type        = list(string)
  description = "List of EC2 instance names"
}

variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type"
}

variable "tags" {
  type        = map(string)
  description = "Common tags applied to instances"
  default     = {
    environment = "dev"
  }
}
```

***

#### Step 4: EC2 Instance Definition (`ec2_instance.tf`)

Create `ec2_instance.tf`:

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default_vpc_subnets" {
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

resource "aws_instance" "compute" {
  count = length(var.compute_names)

  ami                         = data.aws_ami.ubuntu_2404.id
  instance_type                = var.instance_type
  subnet_id                    = data.aws_subnets.default_vpc_subnets.ids[0]
  vpc_security_group_ids       = [data.aws_security_group.default.id]
  associate_public_ip_address  = true

  tags = merge(
    var.tags,
    {
      Name = var.compute_names[count.index]
    }
  )
}
```

***

#### Step 5: Variable Values (`terraform.tfvars`)

Create `terraform.tfvars`:

```hcl
region        = "us-east-1"
instance_type = "t3.micro"

compute_names = ["compute-saurabh-1", "compute-saurabh-2"]

tags = {
  environment = "dev"
  owner       = "saurabh"
}
```

***

#### Step 6: Terraform Workflow

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

***

#### Step 7: Destroy Resources

```bash
terraform destroy --auto-approve
```

***

## PART 3: Using `for_each`

### Task 3: Provision Multiple EC2 Instances Using `for_each`

#### Step 1: Create Project Directory

```bash
mkdir for_each_example_aws
cd for_each_example_aws
```

***

#### Step 2: Provider Configuration (`providers.tf`)

Use the same `providers.tf` as Part 1.

***

#### Step 3: Define Variables (`variables.tf`)

Create `variables.tf`:

```hcl
variable "region" {
  type        = string
  default     = "us-east-1"
  description = "AWS region to deploy into"
}

variable "compute_names" {
  type        = set(string)
  description = "Set of EC2 instance names"
}

variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type"
}

variable "tags" {
  type        = map(string)
  description = "Common tags applied to instances"
  default     = {
    environment = "dev"
  }
}
```

***

#### Step 4: EC2 Instance Definition (`ec2_instance.tf`)

Create `ec2_instance.tf`:

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default_vpc_subnets" {
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

resource "aws_instance" "compute" {
  for_each = var.compute_names

  ami                         = data.aws_ami.ubuntu_2404.id
  instance_type                = var.instance_type
  subnet_id                    = data.aws_subnets.default_vpc_subnets.ids[0]
  vpc_security_group_ids       = [data.aws_security_group.default.id]
  associate_public_ip_address  = true

  tags = merge(
    var.tags,
    {
      Name = each.value
    }
  )
}
```

Explanation:

* `for_each` creates one EC2 instance per value in the set
* Each instance is tracked by key, which makes identities more stable than index-based tracking

***

#### Step 5: Variable Values (`terraform.tfvars`)

Create `terraform.tfvars`:

```hcl
region        = "us-east-1"
instance_type = "t3.micro"

compute_names = ["compute-saurabh-1", "compute-saurabh-2"]

tags = {
  environment = "dev"
  owner       = "saurabh"
}
```

***

#### Step 6: Terraform Workflow

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

***

#### Step 7: Destroy Resources

```bash
terraform destroy --auto-approve
```

***

### Key Takeaways

* Use `count` when resources are identical and a simple quantity is enough
* Be careful using `count` with lists because changing list order can recreate instances
* Prefer `for_each` when you need stable, unique identities (production-friendly)
* `for_each` makes drift and change management easier when items are added or removed

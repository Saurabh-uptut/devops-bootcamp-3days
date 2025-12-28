# Lab 5: Building Reusable Components in Terraform on AWS

### Learning objectives

* Build reusable resources using `for_each`
* Build reusable parts of resources using `dynamic` blocks

### Prerequisite

Complete a basic Terraform lab where you have deployed at least one EC2 instance on AWS (AWS CLI configured, Terraform working, IAM permissions ready).

***

### Target architecture

* 1 VPC
* Multiple subnets in the same VPC
* 1 EC2 instance per subnet
* 1 Security Group with multiple ingress rules using a `dynamic` block
* Public subnet(s) route to Internet Gateway
* Private subnet(s) do not have a default route to the internet

***

### Folder structure

```
.
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── network.tf
├── subnet.tf
├── routes.tf
├── security_group.tf
├── ec2_instance.tf
├── outputs.tf
└── locals.tf
```

***

## Task 0: Project setup

### Step 1: Create project directory

```bash
mkdir lab5_reusable_aws
cd lab5_reusable_aws
```

***

## Task 1: Create multiple subnets using `for_each`

### 1) provider.tf

Create `provider.tf`:

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

### 2) variables.tf (add subnet variable)

Add to `variables.tf`:

```hcl
variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR for the VPC"
  default     = "10.0.0.0/16"
}

variable "subnets" {
  type = map(object({
    cidr_block = string
    az         = string
    public     = bool
  }))
  description = "Subnets to create in the VPC"
}
```

### 3) terraform.tfvars (subnets)

Create `terraform.tfvars`:

```hcl
region   = "us-east-1"
vpc_cidr = "10.0.0.0/16"

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
```

### 4) network.tf (VPC + IGW)

Create `network.tf`:

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "lab5-vpc"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "lab5-igw"
  }
}
```

### 5) subnet.tf (for\_each)

Create `subnet.tf`:

```hcl
resource "aws_subnet" "subnet" {
  for_each = var.subnets

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.az
  map_public_ip_on_launch = each.value.public

  tags = {
    Name = each.key
    Tier = each.value.public ? "public" : "private"
  }
}
```

#### Common issue you will hit (and why)

If older code referenced a single subnet like:

```hcl
subnet_id = aws_subnet.subnet.id
```

Terraform will fail because `aws_subnet.subnet` is now a map of resources.

#### Temporary fix

Pick one subnet key explicitly:

```hcl
subnet_id = aws_subnet.subnet["public-subnet"].id
```

Later (Task 3) you will do the correct fix by selecting the subnet per instance using `each.value.subnet`.

***

## Task 2: Create multiple inbound rules using a `dynamic` block (Security Group)

On AWS, the equivalent of a firewall in this context is a Security Group.

### 1) variables.tf (add ingress variable)

Add to `variables.tf`:

```hcl
variable "ingress" {
  type = list(object({
    description = string
    protocol    = string
    from_port   = number
    to_port     = number
    cidr_blocks = list(string)
  }))
  description = "Ingress rules for the security group"
}
```

### 2) terraform.tfvars (ingress rules)

Add to `terraform.tfvars`:

```hcl
ingress = [
  {
    description = "SSH from internet"
    protocol    = "tcp"
    from_port   = 22
    to_port     = 22
    cidr_blocks = ["0.0.0.0/0"]
  },
  {
    description = "App port 8080 from internet"
    protocol    = "tcp"
    from_port   = 8080
    to_port     = 8080
    cidr_blocks = ["0.0.0.0/0"]
  },
  {
    description = "ICMP ping from internet"
    protocol    = "icmp"
    from_port   = -1
    to_port     = -1
    cidr_blocks = ["0.0.0.0/0"]
  }
]
```

### 3) locals.tf (optional common tags)

Create `locals.tf`:

```hcl
locals {
  common_tags = {
    lab = "lab5-reusable-components"
  }
}
```

### 4) security\_group.tf (dynamic ingress)

Create `security_group.tf`:

```hcl
resource "aws_security_group" "app_sg" {
  name        = "lab5-app-sg"
  description = "Security group with dynamic ingress rules"
  vpc_id      = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress
    content {
      description = ingress.value.description
      protocol    = ingress.value.protocol
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "lab5-app-sg"
  })
}
```

***

## Task 2.5: Route tables for public and private subnets

### routes.tf

Create `routes.tf`:

```hcl
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "lab5-public-rt"
  }
}

resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "lab5-private-rt"
  }
}

resource "aws_route_table_association" "subnet_assoc" {
  for_each = var.subnets

  subnet_id      = aws_subnet.subnet[each.key].id
  route_table_id = each.value.public ? aws_route_table.public_rt.id : aws_route_table.private_rt.id
}
```

***

## Task 3: Create multiple EC2 instances using `for_each`

### 1) variables.tf (instances map)

Add to `variables.tf`:

```hcl
variable "ec2_instances" {
  type = map(object({
    subnet        = string
    public_ip     = bool
    instance_type = string
  }))
  description = "EC2 instances to create, one per subnet"
}
```

### 2) terraform.tfvars (instances definition)

Add to `terraform.tfvars`:

```hcl
ec2_instances = {
  public-vm-saurabh = {
    subnet        = "public-subnet"
    public_ip     = true
    instance_type = "t3.micro"
  }

  private-vm-saurabh = {
    subnet        = "private-subnet"
    public_ip     = false
    instance_type = "t3.micro"
  }
}
```

### 3) ec2\_instance.tf (for\_each + correct subnet selection)

Create `ec2_instance.tf`:

```hcl
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

resource "aws_instance" "vm" {
  for_each = var.ec2_instances

  ami                         = data.aws_ami.ubuntu_2404.id
  instance_type               = each.value.instance_type
  subnet_id                    = aws_subnet.subnet[each.value.subnet].id
  vpc_security_group_ids       = [aws_security_group.app_sg.id]
  associate_public_ip_address  = each.value.public_ip

  tags = merge(local.common_tags, {
    Name   = each.key
    Subnet = each.value.subnet
  })
}
```

This is the correct fix for Task 1. Each VM chooses its subnet from the subnet map using the subnet key.

***

## Outputs: Multiple public and private IPs

### outputs.tf

Create `outputs.tf`:

```hcl
output "public_ip_addresses" {
  description = "Public IPs for instances that have a public IP"
  value = [
    for vm in values(aws_instance.vm) :
    vm.public_ip
    if vm.public_ip != ""
  ]
}

output "private_ip_addresses" {
  description = "Private IPs for all instances"
  value = [
    for vm in values(aws_instance.vm) :
    vm.private_ip
  ]
}
```

***

## Terraform workflow

Run these in order:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

***

### Expected outcome

* VPC created
* Two subnets created: `public-subnet`, `private-subnet`
* Route tables configured:
  * public subnet has a default route to the internet via IGW
  * private subnet has no internet default route
* Two EC2 instances created: one per subnet
* Security group has three ingress rules created via `dynamic`:
  * tcp 22
  * tcp 8080
  * icmp
* Outputs show:
  * list of public IPs (only for instances with public IPs)
  * list of private IPs (for all instances)

***

### Cleanup

```bash
terraform destroy --auto-approve
```

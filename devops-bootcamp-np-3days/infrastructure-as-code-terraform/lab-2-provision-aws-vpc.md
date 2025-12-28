# Lab 2: Provision AWS VPC

## Goal

Provision a complete VPC setup on AWS with:

* 1 VPC
* 1 Public subnet (with Internet Gateway and public route)
* 1 Private subnet (no direct Internet route)
* 2 Security Groups (public and private)
* 2 EC2 instances
  * Public EC2 runs Apache and is reachable over HTTP
  * Private EC2 has no public IP and is reachable only via SSH from the public EC2
* 1 Custom Network ACL (NACL) applied to both subnets

## Learning Objectives

By the end of this lab, you will be able to:

* Create a VPC, subnets, route tables, and an Internet Gateway using Terraform
* Design public vs private subnet routing correctly
* Create security groups with least-privilege ingress rules
* Use a bastion pattern (SSH to public VM, then SSH to private VM)
* Apply and understand NACL rules and subnet associations
* Use Terraform outputs to retrieve the public IP for verification

***

## Prerequisites

* AWS account (free tier or paid)
* IAM credentials with permissions for: VPC, EC2, Internet Gateway, Route Tables, Security Groups, Network ACL
* Terraform installed (Terraform v1.14.3 is current per HashiCorp install page) ([HashiCorp Developer](https://developer.hashicorp.com/terraform/install?utm_source=chatgpt.com))
* AWS CLI installed and configured
* VS Code (or any IDE)
* An existing EC2 Key Pair in the target region (recommended)
  * You must know its name (example: `myKey`)
  * You must have its `.pem` private key locally to SSH

### Configure AWS CLI

Run:

```bash
aws configure
```

Validate:

```bash
aws sts get-caller-identity
```

***

## Architecture (Text View)

* **VPC**: `10.0.0.0/16`
* **Public Subnet**: `10.1.0.0/24` in `us-east-1a`
  * Route table has `0.0.0.0/0` to Internet Gateway
  * Public EC2 has a public IP, allows HTTP (80) and SSH (22)
* **Private Subnet**: `10.2.0.0/24` in `us-east-1b`
  * Route table has only local VPC routing (no Internet route)
  * Private EC2 has no public IP, allows SSH only from public security group
* **NACL**: Custom rules applied to both subnets (stateless control)

***

## Project Structure

Create a folder named `vpc`:

```
vpc/
  providers.tf
  variables.tf
  terraform.tfvars
  vpc.tf
  subnet.tf
  gw.tf
  routes.tf
  security_group.tf
  nacl.tf
  ec2_instance.tf
  outputs.tf
```

***

## Step 1: Create providers.tf (latest provider range)

The latest AWS provider release is 6.27.0 (Dec 17, 2025). ([GitHub](https://github.com/hashicorp/terraform-provider-aws/releases?utm_source=chatgpt.com)) Use a safe constraint that stays within major version 6.

**providers.tf**

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

## Step 2: Create variables.tf

**variables.tf**

```hcl
variable "region" {
  type        = string
  description = "AWS region to deploy into"
  default     = "us-east-1"
}

variable "vpc_name" {
  type        = string
  description = "Name tag for the VPC"
}

variable "cidr_block" {
  type        = string
  description = "CIDR block for VPC"
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr_block" {
  type        = string
  description = "CIDR for public subnet"
  default     = "10.1.0.0/24"
}

variable "private_subnet_cidr_block" {
  type        = string
  description = "CIDR for private subnet"
  default     = "10.2.0.0/24"
}

variable "availability_zone" {
  type        = map(string)
  description = "AZ mapping for subnets"
  default = {
    public_subnet_az  = "us-east-1a"
    private_subnet_az = "us-east-1b"
  }
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"
}

variable "key_name" {
  type        = string
  description = "Existing EC2 Key Pair name in AWS (for SSH)"
}

variable "allowed_ssh_cidr" {
  type        = string
  description = "CIDR allowed to SSH into the public EC2 (restrict to your IP in real use)"
  default     = "0.0.0.0/0"
}
```

***

## Step 3: Create terraform.tfvars

**terraform.tfvars**

```hcl
region = "us-east-1"

availability_zone = {
  public_subnet_az  = "us-east-1a"
  private_subnet_az = "us-east-1b"
}

cidr_block                = "10.0.0.0/16"
vpc_name                  = "myVpc-saurabh"
public_subnet_cidr_block  = "10.1.0.0/24"
private_subnet_cidr_block = "10.2.0.0/24"

instance_type    = "t2.micro"
key_name         = "myKey"
allowed_ssh_cidr = "0.0.0.0/0"
```

***

## Step 4: Create the VPC

**vpc.tf**

```hcl
resource "aws_vpc" "myVpc" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = var.vpc_name
  }
}
```

***

## Step 5: Create subnets

**subnet.tf**

```hcl
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.myVpc.id
  cidr_block              = var.public_subnet_cidr_block
  map_public_ip_on_launch = true
  availability_zone       = var.availability_zone["public_subnet_az"]

  tags = {
    Name = "public_subnet"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id                  = aws_vpc.myVpc.id
  cidr_block              = var.private_subnet_cidr_block
  map_public_ip_on_launch = false
  availability_zone       = var.availability_zone["private_subnet_az"]

  tags = {
    Name = "private_subnet"
  }
}
```

***

## Step 6: Create Internet Gateway

**gw.tf**

```hcl
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.myVpc.id

  tags = {
    Name = "igw"
  }
}
```

***

## Step 7: Create route tables and associations

Important note: You do not need a special “SSH route” between subnets. Subnets inside the same VPC can route to each other via the default local route. What controls SSH access is Security Groups and NACL rules.

**routes.tf**

```hcl
resource "aws_route_table" "public_route" {
  vpc_id = aws_vpc.myVpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "public_route"
  }
}

resource "aws_route_table_association" "public_association" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_route.id
}

resource "aws_route_table" "private_route" {
  vpc_id = aws_vpc.myVpc.id

  tags = {
    Name = "private_route"
  }
}

resource "aws_route_table_association" "private_association" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.private_route.id
}
```

***

## Step 8: Create Security Groups (correct rules)

Key correction: For “private allows SSH only from public machines”, the safest and standard way is referencing the public Security Group, not using subnet CIDR.

**security\_group.tf**

```hcl
resource "aws_security_group" "public_security_group" {
  name        = "allow_public_traffic"
  description = "Allow HTTP and SSH to public instance"
  vpc_id      = aws_vpc.myVpc.id

  ingress {
    description = "SSH from allowed CIDR"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.allowed_ssh_cidr]
  }

  ingress {
    description = "HTTP from Internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Outbound anywhere"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow_public_traffic"
  }
}

resource "aws_security_group" "private_security_group" {
  name        = "allow_private_traffic"
  description = "Allow SSH only from public security group"
  vpc_id      = aws_vpc.myVpc.id

  ingress {
    description     = "SSH from public SG only"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.public_security_group.id]
  }

  egress {
    description = "Outbound anywhere (stays private unless a NAT route exists)"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow_private_traffic"
  }
}
```

***

## Step 9: Create Network ACL (NACL) and associate to subnets

This is a simple, teachable NACL:

* Public subnet: allow inbound HTTP (80), SSH (22), and ephemeral return traffic
* Private subnet: allow inbound SSH (22) only from public subnet CIDR, plus ephemeral return traffic
* Outbound: allow all (simplifies learning)

**nacl.tf**

```hcl
resource "aws_network_acl" "custom_nacl" {
  vpc_id = aws_vpc.myVpc.id

  tags = {
    Name = "custom_nacl"
  }
}

resource "aws_network_acl_association" "public_nacl_assoc" {
  network_acl_id = aws_network_acl.custom_nacl.id
  subnet_id      = aws_subnet.public_subnet.id
}

resource "aws_network_acl_association" "private_nacl_assoc" {
  network_acl_id = aws_network_acl.custom_nacl.id
  subnet_id      = aws_subnet.private_subnet.id
}

# Inbound rules
resource "aws_network_acl_rule" "in_public_ssh" {
  network_acl_id = aws_network_acl.custom_nacl.id
  rule_number    = 100
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = var.allowed_ssh_cidr
  from_port      = 22
  to_port        = 22
}

resource "aws_network_acl_rule" "in_public_http" {
  network_acl_id = aws_network_acl.custom_nacl.id
  rule_number    = 110
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 80
  to_port        = 80
}

# Allow ephemeral inbound for return traffic
resource "aws_network_acl_rule" "in_public_ephemeral" {
  network_acl_id = aws_network_acl.custom_nacl.id
  rule_number    = 120
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}

resource "aws_network_acl_rule" "in_private_ssh_from_public_subnet" {
  network_acl_id = aws_network_acl.custom_nacl.id
  rule_number    = 200
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = aws_subnet.public_subnet.cidr_block
  from_port      = 22
  to_port        = 22
}

resource "aws_network_acl_rule" "in_private_ephemeral" {
  network_acl_id = aws_network_acl.custom_nacl.id
  rule_number    = 210
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = aws_subnet.public_subnet.cidr_block
  from_port      = 1024
  to_port        = 65535
}

# Outbound rules (allow all)
resource "aws_network_acl_rule" "out_all" {
  network_acl_id = aws_network_acl.custom_nacl.id
  rule_number    = 100
  egress         = true
  protocol       = "-1"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
}
```

***

## Step 10: Create EC2 instances (with latest Ubuntu AMI via data source)

Instead of hardcoding AMI IDs (which change), use a data source to fetch the latest Ubuntu 24.04 LTS for the region.

**ec2\_instance.tf**

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

resource "aws_instance" "public_ec2" {
  ami                         = data.aws_ami.ubuntu_2404.id
  instance_type               = var.instance_type
  key_name                    = var.key_name
  availability_zone           = var.availability_zone["public_subnet_az"]
  subnet_id                   = aws_subnet.public_subnet.id
  vpc_security_group_ids      = [aws_security_group.public_security_group.id]
  associate_public_ip_address = true

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y apache2
    systemctl start apache2
    systemctl enable apache2
    echo "<html><body><h1>Hi there! This is Saurabh’s public VM</h1></body></html>" > /var/www/html/index.html
  EOF

  tags = {
    Name = "public_web_instance"
  }
}

resource "aws_instance" "private_ec2" {
  ami                    = data.aws_ami.ubuntu_2404.id
  instance_type          = var.instance_type
  key_name               = var.key_name
  availability_zone      = var.availability_zone["private_subnet_az"]
  subnet_id              = aws_subnet.private_subnet.id
  vpc_security_group_ids = [aws_security_group.private_security_group.id]

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y apache2
    systemctl start apache2
    systemctl enable apache2
    echo "<html><body><h1>Hi there! This is the private VM</h1></body></html>" > /var/www/html/index.html
  EOF

  tags = {
    Name = "private_instance"
  }
}
```

***

## Step 11: Output values

**outputs.tf**

```hcl
output "public_ip_address" {
  description = "Public IP of the public EC2 instance"
  value       = aws_instance.public_ec2.public_ip
}

output "private_ip_address" {
  description = "Private IP of the private EC2 instance"
  value       = aws_instance.private_ec2.private_ip
}
```

***

## Step 12: Execute the Terraform workflow

From the `vpc/` directory:

```bash
terraform fmt
terraform validate
terraform init
terraform plan
terraform apply
```

***

## Verification

### 1) Verify Apache on Public EC2

After apply, Terraform prints `public_ip_address`.

Open in browser:

```
http://<public_ip_address>
```

Or verify from terminal:

```bash
curl http://<public_ip_address>
```

You should see the HTML page from the public VM.

### 2) SSH to the public EC2 (bastion)

```bash
ssh -i myKey.pem ubuntu@<public_ip_address>
```

### 3) From public EC2, SSH to private EC2

Use the private IP output (`private_ip_address`):

```bash
ssh -i myKey.pem ubuntu@<private_ip_address>
```

If SSH fails, check:

* Your key pair name matches `key_name`
* Your local `allowed_ssh_cidr` allows your IP
* NACL rules allow SSH and ephemeral ports
* Security group rules are correct

***

## Cleanup (avoid charges)

```bash
terraform destroy
```

# Lab 8: Provision AWS Infrastructure using Reusable Terraform Modules

## Goal architecture

* 1 VPC
* 2 subnets: `public-subnet`, `private-subnet`
* Routeing

  * Public subnet routes `0.0.0.0/0` to an Internet Gateway
  * Private subnet has no internet route (no NAT in this lab)
* 1 Security Group: allow **tcp 22** and **tcp 80** (implemented using `dynamic` ingress rules)
* 2 EC2 instances: one per subnet

  * Public VM has a public IP
  * Private VM has no public IP

---

## Learning outcomes

* Build reusable Terraform modules for AWS VPC, subnets, security group, and EC2
* Use `for_each` at the module level to create multiple subnets and VMs
* Use `dynamic` blocks to make modules flexible (ingress rules, optional public IP)
* Consume module outputs cleanly as maps keyed by resource name

---

## Folder structure and module layout

Create this structure:

```bash
mkdir -p AWS/aws_vpc \
         AWS/aws_subnet \
         AWS/aws_security_group \
         AWS/aws_ec2_instance \
         infra
```

Inside each module folder you will create `variables.tf`, a main `.tf` file, and `outputs.tf`.

---

# MODULE 1: VPC (`AWS/aws_vpc`)

## `variables.tf`

```hcl
variable "name" { type = string }

variable "cidr_block" {
  type        = string
  description = "VPC CIDR"
}

variable "enable_dns_hostnames" {
  type        = bool
  description = "Enable DNS hostnames"
  default     = true
}

variable "enable_dns_support" {
  type        = bool
  description = "Enable DNS support"
  default     = true
}
```

## `vpc.tf`

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support

  tags = {
    Name = var.name
  }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.name}-igw"
  }
}
```

## `outputs.tf`

```hcl
output "vpc_id" {
  value = aws_vpc.this.id
}

output "vpc_cidr" {
  value = aws_vpc.this.cidr_block
}

output "igw_id" {
  value = aws_internet_gateway.this.id
}
```

---

# MODULE 2: Subnet (`AWS/aws_subnet`)

## `variables.tf`

```hcl
variable "name" { type = string }

variable "vpc_id" { type = string }

variable "cidr_block" { type = string }

variable "availability_zone" { type = string }

variable "public" {
  type        = bool
  description = "If true, map public IPs on launch"
}
```

## `subnet.tf`

```hcl
resource "aws_subnet" "this" {
  vpc_id                  = var.vpc_id
  cidr_block              = var.cidr_block
  availability_zone       = var.availability_zone
  map_public_ip_on_launch = var.public

  tags = {
    Name = var.name
    Tier = var.public ? "public" : "private"
  }
}
```

## `outputs.tf`

```hcl
output "subnet_id" {
  value = aws_subnet.this.id
}

output "cidr_block" {
  value = aws_subnet.this.cidr_block
}
```

---

# MODULE 3: Security Group (`AWS/aws_security_group`) using `dynamic` ingress rules

## `variables.tf`

```hcl
variable "name" { type = string }
variable "vpc_id" { type = string }

variable "ingress" {
  type = list(object({
    description = string
    protocol    = string
    from_port   = number
    to_port     = number
    cidr_blocks = list(string)
  }))
  description = "Ingress rules to apply using a dynamic block"
}
```

## `security_group.tf`

```hcl
resource "aws_security_group" "this" {
  name        = var.name
  description = "Security group created via reusable module"
  vpc_id      = var.vpc_id

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

  tags = {
    Name = var.name
  }
}
```

## `outputs.tf`

```hcl
output "security_group_id" {
  value = aws_security_group.this.id
}
```

---

# MODULE 4: EC2 Instance (`AWS/aws_ec2_instance`) with optional public IP

## `variables.tf`

```hcl
variable "name" { type = string }
variable "instance_type" { type = string }

variable "subnet_id" { type = string }
variable "security_group_ids" { type = list(string) }

variable "key_name" {
  type        = string
  description = "EC2 key pair name"
}

variable "public_ip" {
  type        = bool
  description = "Attach public IP if true"
}

variable "tags" {
  type        = map(string)
  description = "Extra tags"
  default     = {}
}
```

## `ec2_instance.tf`

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

resource "aws_instance" "this" {
  ami                         = data.aws_ami.ubuntu_2404.id
  instance_type               = var.instance_type
  subnet_id                    = var.subnet_id
  vpc_security_group_ids       = var.security_group_ids
  key_name                     = var.key_name
  associate_public_ip_address  = var.public_ip

  tags = merge(
    var.tags,
    {
      Name = var.name
    }
  )
}
```

## `outputs.tf`

```hcl
output "instance_id" {
  value = aws_instance.this.id
}

output "private_ip" {
  value = aws_instance.this.private_ip
}

output "public_ip" {
  value = aws_instance.this.public_ip
}
```

---

# INFRA PROJECT (`infra/`)

## Step 1: Create `providers.tf`

Create `infra/providers.tf`:

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
  region = var.region
}
```

If you want to use remote state (optional), add an S3 backend here after creating the bucket and DynamoDB table.

---

## Step 2: Create `variables.tf`

Create `infra/variables.tf`:

```hcl
variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}
```

---

## Step 3: Create SSH key (TLS) + AWS Key Pair

Create `infra/ssh_key.tf`:

```hcl
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "lab_key" {
  key_name   = "lab8-key"
  public_key = tls_private_key.ssh_key.public_key_openssh
}

output "private_key_pem" {
  value     = tls_private_key.ssh_key.private_key_pem
  sensitive = true
}
```

You will use `aws_key_pair.lab_key.key_name` for both VMs.

---

## Step 4: Create `locals.tf` (subnets + VMs)

Create `infra/locals.tf`:

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

  security_group_ingress = [
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

## Step 5: Create `main.tf` (wire modules together)

Create `infra/main.tf`:

```hcl
module "vpc" {
  source = "../AWS/aws_vpc"

  name       = local.vpc.name
  cidr_block = local.vpc.cidr_block
}

module "subnet" {
  for_each = local.subnets
  source   = "../AWS/aws_subnet"

  name              = each.key
  vpc_id            = module.vpc.vpc_id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az
  public            = each.value.public
}

module "security_group" {
  source = "../AWS/aws_security_group"

  name    = "allow-ssh-http"
  vpc_id  = module.vpc.vpc_id
  ingress = local.security_group_ingress
}

module "ec2" {
  for_each = local.instances
  source   = "../AWS/aws_ec2_instance"

  name               = each.key
  instance_type      = each.value.instance_type
  subnet_id          = module.subnet[each.value.subnet].subnet_id
  security_group_ids = [module.security_group.security_group_id]
  key_name           = aws_key_pair.lab_key.key_name
  public_ip          = each.value.public_ip
  tags               = each.value.tags
}
```

---

## Step 6: Create `routes.tf` (public vs private routing)

Create `infra/routes.tf`:

```hcl
resource "aws_route_table" "public_rt" {
  vpc_id = module.vpc.vpc_id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = module.vpc.igw_id
  }

  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table" "private_rt" {
  vpc_id = module.vpc.vpc_id

  tags = {
    Name = "private-rt"
  }
}

resource "aws_route_table_association" "subnet_assoc" {
  for_each = local.subnets

  subnet_id = module.subnet[each.key].subnet_id
  route_table_id = each.value.public ? aws_route_table.public_rt.id : aws_route_table.private_rt.id
}
```

---

## Step 7: Create `outputs.tf`

Create `infra/outputs.tf`:

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

# Run Terraform workflow

From inside `infra/`:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply --auto-approve
```

---

## What students should observe

* Modules are reusable across projects
* `for_each` at the module level creates multiple subnets and VMs cleanly
* `dynamic` blocks make modules flexible (security group rules)
* Outputs are easy to consume as maps keyed by VM name
* Public VM gets a public IP, private VM does not

---

## Cleanup

From `infra/`:

```bash
terraform destroy --auto-approve
```
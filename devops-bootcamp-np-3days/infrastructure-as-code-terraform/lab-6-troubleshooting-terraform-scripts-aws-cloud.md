# Lab 5: Building Reusable Components in Terraform on AWS

### Learning objectives

* Build reusable resources using `for_each`
* Build reusable parts of resources using `dynamic` blocks

### Prerequisite

This lab is continuation of [Lab 2: Provision AWS VPC](lab-2-provision-aws-vpc.md) and we will use the same code repository.

Make sure to complete lab 2 before continuing here&#x20;

### Target architecture

* 1 VPC
* Multiple subnets in the same VPC
* 1 EC2 instance per subnet
* 1 Security Group with multiple ingress rules using a `dynamic` block
* Public subnet(s) route to Internet Gateway
* Private subnet(s) do not have a default route to the internet

## Task 1: Create multiple subnets using `for_each`

### 1) In the variables.tf, add a subnet variable

Add a new variable called subnets in `variables.tf`:

```hcl

variable "subnets" {
  type = map(object({
    cidr_block = string
    az         = string
    public     = bool
  }))
  description = "Subnets to create in the VPC"
}
```

### 2) update terraform.tfvars with the subnets value as follow:

Update `terraform.tfvars` with the following subnets

```hcl

subnets = {
  public-subnet = {
    cidr_block = "10.0.2.0/24"
    az         = "us-east-1a"
    public     = true
  }

  private-subnet = {
    cidr_block = "10.0.3.0/24"
    az         = "us-east-1b"
    public     = false
  }
}
```

### 5) In subnet.tf, update subnet resource block with the following code:

Update `subnet.tf`:

```hcl
resource "aws_subnet" "subnet" {
  for_each = var.subnets

  vpc_id                  = aws_vpc.myVpc.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.az
  map_public_ip_on_launch = each.value.public

  tags = {
    Name = each.key
    Tier = each.value.public ? "public" : "private"
  }
}
```

#### Common issue you will hit (and why?)

The older code referenced a single subnet like:

```hcl
aws_subnet.subnet.id
```

Terraform will fail because `aws_subnet.subnet` is now a map of resources.

#### Temporary fix

Update the resources that use public subnet with this -

```hcl
aws_subnet.subnet["public-subnet"].id
```

and the resource blocks that use private subnet with this -

```
aws_subnet.subnet["private-subnet"].id
```

Later (Task 2 and 3) you will do the correct fix by selecting the subnet per instance using `each.value.subnet`.

Note - This will need update in ec2\_instance.tf, nacl.tf, and route.tf files. Make sure to update the subnet reference in all these places.

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

Update the public\_security\_group in `security_group.tf` with the following code.<br>

_Note - We are not updating the private\_security\_group, if you wish you can apply the same logic to update the private security group as well._

```hcl
resource "aws_security_group" "public_security_group" {
  name        = "allow_public_traffic"
  description = "Security group with dynamic ingress rules"
  vpc_id      = aws_vpc.myVpc.id

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
    Name = "allow_public_traffic"
  })
}
```

***

## Task 2.5: Route tables for public and private subnets

### routes.tf

Create `routes.tf`:

```hcl
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.myVpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "lab5-public-rt"
  }
}

resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.myVpc.id

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

Update `ec2_instance.tf`:

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
  key_name                    = aws_key_pair.ssh_key.key_name
  instance_type               = each.value.instance_type
  subnet_id                    = aws_subnet.subnet[each.value.subnet].id
  vpc_security_group_ids       = var.subnets[each.value.subnet].public ? [aws_security_group.public_security_group.id] : [aws_security_group.private_security_group.id]
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

Update `outputs.tf`:

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

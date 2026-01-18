# Lab 9:  (optional) Troubleshoot AWS Scripts

Repository (used for this lab): `hashicorp-education/learn-terraform-troubleshooting`&#x20;

#### Goal

Practice a repeatable troubleshooting workflow by running Terraform commands, reading the error output carefully, and applying targeted fixes until the configuration validates and applies successfully.

#### Learning objectives

By the end of this lab, you will be able to:

* Use `terraform fmt` and `terraform validate` to detect configuration language issues early
* Fix common Terraform failures:
  * HCL syntax and interpolation mistakes
  * Cyclic dependency errors
  * Incorrect `for_each` usage (wrong types, splat expressions)
  * Outputs that break when resources use `for_each`
* Apply and destroy infrastructure safely with Terraform

#### Prerequisites

* An AWS account
* AWS CLI installed and configured (`aws configure`)
* Terraform installed (use a modern Terraform 1.14.x+ recommended)&#x20;
* Git installed
* VS Code (or similar editor)

#### Recommended provider versions (latest as of mid Dec 2025)

* AWS provider: `hashicorp/aws` `6.27.0`&#x20;

### Step 1: Clone the repo and review files

1. Clone the repository:

```bash
git clone https://github.com/hashicorp-education/learn-terraform-troubleshooting.git
cd learn-terraform-troubleshooting
```

2. Open these files in your editor:

* `main.tf`
* `outputs.tf`
* `terraform.tfvars`

Do not edit yet. First, reproduce the errors.

### Scenario 1: Syntax and interpolation errors

#### What you will do

Run `terraform fmt`, read the exact line number, then fix the broken string interpolation.

#### Steps

1. Run formatting:

```bash
terraform fmt
```

You should see errors complaining about an invalid character and invalid expression in the EC2 instance tags, caused by a malformed variable reference like: `Name = $var.name-learn`&#x20;

2. Fix the interpolation in `main.tf` (inside the EC2 resource tags):

```hcl
tags = {
  Name = "${var.name}-learn"
}
```

This exact correction is what resolves the parse error.&#x20;

3. Re-run:

```bash
terraform fmt
```

### Scenario 2: Cyclic dependency error (security groups)

#### What you will do

Initialize Terraform, validate, then remove the circular dependency between two security groups by moving ingress rules into `aws_security_group_rule` resources.

#### Steps

1. Initialize:

```bash
terraform init
```

2. Validate:

```bash
terraform validate
```

You should see a cycle error similar to: `Cycle: aws_security_group.sg_ping, aws_security_group.sg_8080`&#x20;

3. Fix the cycle

**A. Remove the mutually-referencing `ingress` blocks** inside both `aws_security_group` resources (leave the groups without ingress defined in those blocks).&#x20;

**B. Add these independent rule resources to `main.tf`:**

```hcl
resource "aws_security_group_rule" "allow_ping" {
  type                     = "ingress"
  from_port                = -1
  to_port                  = -1
  protocol                 = "icmp"
  security_group_id        = aws_security_group.sg_ping.id
  source_security_group_id = aws_security_group.sg_8080.id
}

resource "aws_security_group_rule" "allow_8080" {
  type                     = "ingress"
  from_port                = 80
  to_port                  = 80
  protocol                 = "tcp"
  security_group_id        = aws_security_group.sg_8080.id
  source_security_group_id = aws_security_group.sg_ping.id
}
```

This is the recommended pattern to break the dependency cycle.&#x20;

4. Re-run:

```bash
terraform fmt
terraform validate
```

### Scenario 3: `for_each` invalid reference and invalid `each` usage

#### What you will do

Fix a broken `for_each` that uses a splat expression, and fix `each.id` to `each.value` by converting the security group list into a map using `locals`.

#### Steps

1. Run:

```bash
terraform validate
```

You should see errors like:

* Invalid reference: `for_each = aws_security_group.*.id`
* Invalid "each" attribute: `each.id` is not valid&#x20;

2. Fix the EC2 resource in `main.tf`

Update the instance resource as follows:

```hcl
resource "aws_instance" "web_app" {
  for_each               = local.security_groups
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t2.micro"
  vpc_security_group_ids = [each.value]

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y apache2
    sed -i -e 's/80/8080/' /etc/apache2/ports.conf
    echo "Hello World" > /var/www/html/index.html
    systemctl restart apache2
  EOF

  tags = {
    Name = "${var.name}-learn-${each.key}"
  }
}
```

This replaces the invalid splat-based `for_each`, fixes `each.id`, and ensures unique names per instance. (\[HashiCorp Developer]\[1])

3. Add the required `locals` block (in `main.tf`, near the top or bottom):

```hcl
locals {
  security_groups = {
    sg_ping = aws_security_group.sg_ping.id
    sg_8080 = aws_security_group.sg_8080.id
  }
}
```

This is the key change that makes `for_each` valid, because `for_each` needs a map or set of strings, not a list from a splat. (\[HashiCorp Developer]\[1])

4. Re-run:

```bash
terraform fmt
terraform validate
```

### Scenario 4: Outputs fail when a resource uses `for_each`

#### What you will do

Fix outputs so they return values for all instances created by `for_each`.

#### Steps

1. Run:

```bash
terraform validate
```

You should see: `Missing resource instance key` errors in `outputs.tf` because `aws_instance.web_app` is now a map.&#x20;

2. Fix `outputs.tf` using `for` expressions:

```hcl
output "instance_id" {
  description = "ID of the EC2 instances"
  value       = [for instance in aws_instance.web_app : instance.id]
}

output "instance_public_ip" {
  description = "Public IP addresses of the EC2 instances"
  value       = [for instance in aws_instance.web_app : instance.public_ip]
}

output "instance_name" {
  description = "Name tags of the EC2 instances"
  value       = [for instance in aws_instance.web_app : instance.tags.Name]
}
```

This is the recommended fix for outputs when resources use `for_each`.&#x20;

3. Re-run:

```bash
terraform fmt
terraform validate
```

You should now get: `Success! The configuration is valid.`&#x20;

### Run the full workflow (optional)

#### Apply

```bash
terraform plan
terraform apply
```

#### Verify

* Confirm Terraform created multiple EC2 instances (one per security group key)
* Confirm outputs show multiple instance IDs, public IPs, and names

#### Destroy

```bash
terraform destroy --auto-approve
```

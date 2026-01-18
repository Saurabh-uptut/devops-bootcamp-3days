# Lab 1: Provision Your First Infrastructure

## Goal

Provision a single **Amazon EC2 virtual machine** on AWS using **Terraform**, using variables and a `.tfvars` file for clean configuration.

## What you will build

* One EC2 instance in **us-east-1**
* Tagged with `Name = myWebServer`
* AMI and instance type controlled through Terraform variables

## Learning Objectives

By the end of this lab, you will be able to:

* Configure the AWS provider in Terraform
* Initialize a Terraform working directory and understand generated files
* Use input variables and a `terraform.tfvars` file
* Run the standard Terraform workflow: `fmt`, `validate`, `plan`, `apply`
* Verify the deployed EC2 instance in the AWS Console

***

## Prerequisites

* An AWS account (free tier or paid)
* Terraform installed
* AWS CLI installed
* An IAM user/role with permissions to create EC2 instances
* VS Code (or any IDE)
* AWS credentials configured locally using the AWS CLI

### Configure AWS CLI credentials

Run:

```bash
aws configure
```

Provide:

* AWS Access Key ID
* AWS Secret Access Key
* Default region name (use `us-east-1` for this lab)
* Default output format (optional, `json` is fine)

To confirm credentials are working:

```bash
aws sts get-caller-identity
```

## Lab Structure

You will create a Terraform root module folder:

```
myFirstVm/
  providers.tf
  variables.tf
  ec2_instance.tf
  terraform.tfvars
```

## Step 1: Create the project folder

1. Open VS Code
2. Create a new folder named:

```
myFirstVm
```

3. Open the folder in VS Code

## Step 2: Add provider configuration

Create a file named `providers.tf` and add:

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "6.27.0"
    }
  }
}

provider "aws" {
  # Configuration options
  region = "us-east-1"
}
```

### Description

* **terraform block**: Declares required providers and pins the provider version.
* **required\_providers**: Ensures Terraform downloads the correct AWS provider plugin.
* **provider "aws"**: Sets the AWS region where resources will be created.

## Step 3: Initialize Terraform

From the terminal inside `myFirstVm/`, run:

```bash
terraform init
```

### Expected outcome

Terraform downloads the AWS provider and creates:

* `.terraform/` directory (provider plugins and internal files)
* `.terraform.lock.hcl` (locks provider versions for consistent installs)

## Step 4: Define variables

Create a file named `variables.tf` and add:

```hcl
variable "ami_image_name" {
  type        = string
  description = "AMI ID used to create the EC2 instance"
  default     = "ami-0ecb62995f68bb549"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
}
```

### Notes

* `ami_image_name` has a default value for convenience.
* `instance_type` is required and will be provided via `terraform.tfvars`.

## Step 5: Create the EC2 instance resource

Create a file named `ec2_instance.tf` and add:

```hcl
resource "aws_instance" "web_server" {
  ami           = var.ami_image_name
  instance_type = var.instance_type

  tags = {
    Name = "myWebServer"
  }
}
```

### Description

This resource tells Terraform to create:

* An EC2 instance using the AMI ID from `var.ami_image_name`
* An instance size from `var.instance_type`
* A tag to identify the VM in the AWS console

## Step 6: Provide variable values using terraform.tfvars

Create a file named `terraform.tfvars` and add:

```hcl
ami_image_name = "ami-0ecb62995f68bb549"
instance_type  = "t2.micro"
```

### Notes

* AMI IDs vary by region. If you change `region`, you must update the AMI.
* `terraform.tfvars` automatically supplies variable values during `plan` and `apply`.

## Step 7: Format the Terraform code

Run:

```bash
terraform fmt
```

This formats all `.tf` files to standard Terraform style.

## Step 8: Validate the configuration

Run:

```bash
terraform validate
```

This checks for:

* syntax errors
* invalid references
* basic internal consistency

## Step 9: Preview what Terraform will create

Run:

```bash
terraform plan
```

### Expected outcome

Terraform will show:

* one EC2 instance being created
* the AMI, instance type, and tags

## Step 10: Deploy the infrastructure

Run:

```bash
terraform apply --auto-approve
```

### Expected outcome

Terraform will create the EC2 instance and display a completion message.

## Step 11: Verify in AWS Console

1. Open AWS Console
2. Go to **EC2**
3. Click **Instances**
4. Confirm you see an instance named:

`myWebServer`

***

## Troubleshooting Tips

### Invalid AMI error

If you see an error like “InvalidAMIID.NotFound”:

* Confirm your region is `us-east-1`
* Replace the AMI with a valid Amazon Linux 2 or Ubuntu AMI for that region

### Credentials not found or access denied

* Re-run `aws configure`
* Confirm `aws sts get-caller-identity` works
* Ensure IAM permissions include EC2 create permissions

## Cleanup

To avoid charges, destroy the instance after the lab:

```bash
terraform destroy --auto-approve
```

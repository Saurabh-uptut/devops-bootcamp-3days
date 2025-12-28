# Lab: Store Terraform State Remotely in AWS S3 with DynamoDB State Locking

## Goal

Move your Terraform state (`terraform.tfstate`) from local disk to:

* **Amazon S3** for centralized, durable remote state storage
* **Amazon DynamoDB** for **state locking** (prevents two people or pipelines from applying at the same time)

## Learning Objectives

By the end of this lab, you will be able to:

* Create an S3 bucket for Terraform remote state
* Create a DynamoDB table for state locking (partition key `LockID` as String)
* Configure the Terraform **S3 backend**
* Initialize Terraform with remote backend
* Apply infrastructure and confirm locking and remote state storage

---

## Prerequisites

* AWS account (free tier or paid)
* Terraform installed
* AWS CLI installed and configured (`aws configure`)
* IAM user/role that can create and manage:

  * S3 bucket (read/write, list)
  * DynamoDB table (read/write)
  * Any infrastructure you provision (example: EC2)
* VS Code (or any IDE)

Validate AWS access:

```bash
aws sts get-caller-identity
```

---

## Important concept

Terraform backend resources (S3 bucket, DynamoDB table) must exist **before** Terraform can use them as a backend.

So this lab has two stages:

1. Create S3 bucket and DynamoDB table (bootstrap)
2. Configure Terraform backend to use them and deploy an example resource

---

# Stage 1: Create S3 bucket and DynamoDB table

## Step 1: Choose names

Pick globally unique names:

* S3 bucket: `tfstate-saurabh-<unique-number>`
* DynamoDB table: `terraform-state-locks`

Example values used below:

* Bucket: `tfstate-saurabh-12345`
* Table: `terraform-state-locks`
* Region: `us-east-1`

---

## Step 2: Create S3 bucket (AWS CLI)

For `us-east-1`, create bucket like this:

```bash
aws s3api create-bucket \
  --bucket tfstate-saurabh-12345 \
  --region us-east-1
```

Enable versioning (recommended):

```bash
aws s3api put-bucket-versioning \
  --bucket tfstate-saurabh-12345 \
  --versioning-configuration Status=Enabled
```

Enable default encryption (recommended):

```bash
aws s3api put-bucket-encryption \
  --bucket tfstate-saurabh-12345 \
  --server-side-encryption-configuration '{
    "Rules": [
      { "ApplyServerSideEncryptionByDefault": { "SSEAlgorithm": "AES256" } }
    ]
  }'
```

---

## Step 3: Create DynamoDB table for locking (AWS CLI)

The table must have:

* Partition key: `LockID`
* Type: `S` (String)

```bash
aws dynamodb create-table \
  --table-name terraform-state-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

Wait for table to be active:

```bash
aws dynamodb describe-table \
  --table-name terraform-state-locks \
  --region us-east-1
```

---

# Stage 2: Configure Terraform to use the S3 backend and deploy a resource

## Step 4: Create project directory

Create a folder:

```
remote-state-example
```

Open it in VS Code.

---

## Step 5: Create `providers.tf` with backend configuration

Create `providers.tf`:

```hcl
terraform {
  required_version = ">= 1.6.0"

  backend "s3" {
    bucket         = "tfstate-saurabh-12345"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0.0, < 7.0.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

Update these values to match your environment:

* `bucket`
* `region`
* `dynamodb_table`

What this does:

* Stores state as an S3 object at: `s3://<bucket>/terraform.tfstate`
* Uses DynamoDB table to lock state during `plan` and `apply`

---

## Step 6: Create variables and tfvars for an example EC2

### Create `variables.tf`

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"
}
```

### Create `main.tf` (EC2 example)

Use a data source to fetch a recent Ubuntu image automatically (recommended over hardcoding AMIs):

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

resource "aws_instance" "web_server" {
  ami           = data.aws_ami.ubuntu_2404.id
  instance_type = var.instance_type

  tags = {
    Name = "myWebServer"
  }
}
```

### Optional: Create `outputs.tf`

```hcl
output "instance_public_ip" {
  value       = aws_instance.web_server.public_ip
  description = "Public IP of the EC2 instance (if one is assigned)"
}
```

---

## Step 7: Run Terraform workflow

From inside `remote-state-example`:

Format:

```bash
terraform fmt -recursive
```

Validate:

```bash
terraform validate
```

Initialize backend (this is where remote state wiring happens):

```bash
terraform init
```

Plan:

```bash
terraform plan
```

Apply:

```bash
terraform apply
```

---

## Step 8: Verify state locking and remote state file

### 1) Confirm locking message during apply

During `terraform apply`, Terraform should show messages related to acquiring a state lock (wording varies by version). This indicates DynamoDB locking is active.

### 2) Confirm state stored in S3

List the object:

```bash
aws s3 ls s3://tfstate-saurabh-12345/
```

You should see:

* `terraform.tfstate`

You can also verify in AWS Console:

* S3 bucket contains `terraform.tfstate`
* DynamoDB table exists and is used for locking

---

## Common Issues and Fixes

### Bucket does not exist

Backend initialization fails if the bucket is missing.
Fix: Create bucket first (Stage 1).

### DynamoDB table schema wrong

Locking requires partition key `LockID` (String).
Fix: Recreate table with the correct key.

### Access denied

IAM user lacks required permissions.
Fix: Ensure permissions for S3 and DynamoDB, plus the resources you are creating.

### Region mismatch

If backend region and provider region are inconsistent, it can confuse teams.
Fix: Keep `backend "s3" { region = ... }` and `provider "aws" { region = ... }` aligned unless you intentionally separate them.

---

## Cleanup

If you want to remove the EC2 instance:

```bash
terraform destroy
```

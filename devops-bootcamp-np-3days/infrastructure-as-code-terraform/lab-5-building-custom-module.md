# Challenge Lab: Multi-Region Payroll Application Deployment on AWS using Terraform

## Lab Type

Challenge Lab
No solution or reference implementation is provided. Learners are expected to design and implement the Terraform configuration independently.

---

## Scenario

An organization named **NOBLEPROG** has developed a prototype payroll application that must be deployed in **multiple countries**.

Each country must run its **own isolated instance** of the Payroll application, but all deployments must follow the **same architectural blueprint**.

You are responsible for creating a **reusable and scalable Terraform-based solution** that enables this multi-country deployment model.

---

## Architecture Overview

Each country deployment must include:

* One EC2 instance running the Payroll application

  * Uses a **custom AMI** provided by the organization
* One DynamoDB table

  * Stores employee and payroll financial data
* One S3 bucket

  * Stores tax documents and other payroll-related files
* Users access the application via the EC2 instance
* AWS **default VPC and default subnets** must be used
* No custom VPC creation is required

This is a deliberately simplified architecture to focus on **Terraform design, reuse, and scalability**, not networking complexity.

---

## Goal

Design and implement a Terraform solution that can deploy the Payroll application for **multiple countries** using the **same codebase**, while ensuring that:

* Resources are logically separated per country
* Naming collisions are avoided
* The solution is easy to extend to new countries

---

## Learning Objectives

By completing this challenge, you will demonstrate the ability to:

* Translate a real-world business requirement into Terraform infrastructure
* Use Terraform variables and data structures to support multi-country deployments
* Provision EC2, DynamoDB, and S3 resources using Terraform
* Design reusable and scalable Terraform configurations
* Apply consistent naming and tagging strategies
* Use the default AWS VPC and subnets correctly

---

## Constraints and Rules

You must adhere to the following rules:

1. Use **Terraform only** to provision infrastructure
2. Do not create a custom VPC or subnets
3. Use the **default VPC and default subnets**
4. Each country must have:

   * Its own EC2 instance
   * Its own DynamoDB table
   * Its own S3 bucket
5. All resources must be uniquely identifiable per country
6. The solution must allow adding a new country with minimal changes
7. No hardcoded country-specific logic scattered across files
8. No manual resource creation via AWS Console

---

## High-Level Tasks

You are expected to complete the following tasks independently:

### Task 1: Project Setup

* Create a new Terraform project directory
* Define provider configuration for AWS
* Ensure Terraform initialization works correctly

### Task 2: Country-Based Configuration Design

* Decide how countries will be represented in Terraform
* Ensure the design supports adding or removing countries easily
* Avoid code duplication

### Task 3: EC2 Provisioning

* Provision one EC2 instance per country
* Use the provided custom AMI
* Deploy the instance in the default VPC
* Ensure each instance is identifiable by country

### Task 4: DynamoDB Provisioning

* Provision one DynamoDB table per country
* Design a naming strategy that avoids collisions
* Ensure tables are isolated per country

### Task 5: S3 Bucket Provisioning

* Provision one S3 bucket per country
* Ensure bucket names are globally unique
* Apply a consistent naming convention

### Task 6: Access and Tagging

* Apply meaningful tags to all resources
* Tags must include at least:

  * Application name
  * Country
  * Environment

### Task 7: Validation and Deployment

* Validate Terraform configuration
* Generate a Terraform plan and review it
* Apply the configuration successfully
* Verify resources in the AWS Console

---

## Verification Criteria

Your solution will be considered complete when:

* All required resources are created for each country
* Resources are properly named and tagged
* Adding a new country requires minimal code change
* Terraform plan clearly shows per-country resource separation
* Infrastructure is deployed successfully without manual intervention

---

## Optional Extension Challenges

Attempt these only after completing the core lab:

* Add outputs to expose key resource identifiers per country
* Introduce environment separation (dev, staging, prod)
* Add basic IAM roles or policies for EC2 access to S3 and DynamoDB
* Convert the solution into a reusable Terraform module

---

## Deliverables

* Terraform configuration files
* Clear folder structure
* Successful Terraform apply output
* Ability to explain your design decisions

---

## Cleanup

After validation, destroy all created infrastructure using Terraform to avoid unnecessary AWS charges.

---
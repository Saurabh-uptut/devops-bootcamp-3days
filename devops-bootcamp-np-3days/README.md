# Lab: Environment Setup for AWS Infrastructure Automation

In this lab, you will prepare a local development environment to work with AWS infrastructure using Terraform, Kubernetes, and Jenkins.

### Prerequisites

* Laptop or desktop running Ubuntu 22.04 or 24.04
* Active AWS account
* Internet connectivity
* sudo access

### Step 1: Download and Install Terraform

Terraform is used to provision AWS infrastructure as code.

#### Download Terraform

Download Terraform from the official website:\
Terraform Download

#### Install Terraform

Follow the installation steps based on your OS.

#### Verify Installation

```bash
terraform --version
```

You should see the installed Terraform version.

### Step 2: Install Visual Studio Code Terraform Extension

Visual Studio Code will be used as the IDE.

#### Steps

1. Open Visual Studio Code
2. Go to the Extensions tab
3. Search for **Terraform**
4. Install **HashiCorp Terraform**

This enables syntax highlighting, validation, formatting, and IntelliSense.

### Step 3: Download and Install AWS CLI

AWS CLI is required to authenticate and interact with AWS.

#### Download AWS CLI

Download from:\
AWS CLI Download

#### Verify Installation

```bash
aws --version
```

### Step 4: Configure AWS Credentials

Configure AWS CLI:

```bash
aws configure
```

Provide:

* AWS Access Key ID
* AWS Secret Access Key
* Default region (example: us-east-1)
* Output format: json

Verify authentication:

```bash
aws sts get-caller-identity
```

### Step 5: Download and Install kubectl

kubectl is required to interact with Kubernetes clusters such as Amazon EKS.

#### Download kubectl

Download from:\
Download kubectl utility

#### Verify Installation

```bash
kubectl version --client
```

### Step 6: Install Jenkins on Ubuntu&#x20;

Jenkins will be used for CI/CD automation.

#### 6.1 Install Required Dependencies

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg openjdk-17-jdk
```

Verify Java:

```bash
java -version
```

Jenkins requires Java 17 on Ubuntu noble.

#### 6.2 Add Jenkins GPG Key (Modern and Secure)

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
| sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

#### 6.3 Add Jenkins Repository

```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ \
| sudo tee /etc/apt/sources.list.d/jenkins.list
```

#### 6.4 Install Jenkins

```bash
sudo apt update
sudo apt install -y jenkins
```

#### 6.5 Start and Enable Jenkins

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Verify status:

```bash
sudo systemctl status jenkins
```

#### 6.6 Access Jenkins UI

Open browser and navigate to:

```
http://localhost:8080
```

Retrieve initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Complete the setup wizard and install suggested plugins.

### Troubleshooting Section

#### Issue 1: GPG Error and Jenkins Not Found

**Error**

```
NO_PUBKEY 5BA31D57EF5975CA
Package jenkins has no installation candidate
```

**Cause**

* Jenkins repository key missing or outdated
* Ubuntu noble blocks unsigned repositories

**Fix**

Re-add Jenkins GPG key and repository using the signed-by method shown in Step 6.

#### Issue 2: jenkins.service Not Found

**Error**

```
Failed to start jenkins.service: Unit jenkins.service not found
```

**Cause**

* Jenkins package was never installed due to repo failure

**Fix**

Ensure `sudo apt install -y jenkins` completes successfully, then start the service.

#### Issue 3: Ignoring mozilla.source Warning

**Warning**

```
Ignoring file 'mozilla.source' ... invalid filename extension
```

**Fix (Optional Cleanup)**

```bash
sudo mv /etc/apt/sources.list.d/mozilla.source \
/etc/apt/sources.list.d/mozilla.sources
sudo apt update
```

This warning does not block Jenkins installation.

### Final Verification Checklist

Run all commands successfully:

```bash
terraform --version
aws --version
aws sts get-caller-identity
kubectl version --client
java -version
systemctl status jenkins
```

Jenkins UI should load at `http://localhost:8080`.

### Conclusion

Your AWS automation environment is now ready with:

* Terraform for Infrastructure as Code
* AWS CLI for cloud operations
* kubectl for Kubernetes management
* Jenkins for CI/CD automation
* VS Code with Terraform support

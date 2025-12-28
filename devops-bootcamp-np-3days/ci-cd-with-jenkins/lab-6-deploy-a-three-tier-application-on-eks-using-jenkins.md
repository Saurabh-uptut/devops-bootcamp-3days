# Lab 6: Deploy a Three Tier Application on AWS Kubernetes using Jenkins

## Lab Overview

In this lab, you will deploy a complete three tier application on Amazon EKS using Jenkins for CI/CD.

The pipeline will:

* Build frontend and backend Docker images
* Push images to Amazon ECR
* Authenticate to AWS
* Configure kubectl for the EKS cluster
* Deploy the application using Kubernetes manifests

## Application Architecture

Frontend (NGINX static UI)
-> Backend (Node.js API)
-> Database (PostgreSQL)

### Service exposure

* PostgreSQL: ClusterIP
* Backend API: LoadBalancer
* Frontend UI: LoadBalancer

## What You Will Build

You will:

* Configure Jenkins credentials for AWS and ECR
* Create a Jenkins Pipeline (Jenkinsfile)
* Build and push container images
* Deploy Kubernetes manifests to EKS
* Validate the application end to end

---

## Prerequisites

You must have:

* An EKS cluster already running
* kubectl installed and available (on Jenkins agent and optionally locally)
* AWS CLI installed on Jenkins agent
* Docker installed on Jenkins agent
* zip installed (optional, if you also package artifacts)
* Jenkins server with a Linux agent capable of building Docker images
* GitHub repository containing the application code and Kubernetes manifests

Recommended:

* eksctl installed (optional)
* AWS Load Balancer Controller installed on the EKS cluster (recommended for production, not required for this lab if you use Service type LoadBalancer)
* ECR repositories created for frontend and backend images

---

## Part 1: Prepare AWS IAM for Jenkins

You have two common ways to authenticate Jenkins to AWS:

1. IAM user access keys stored as Jenkins credentials (easy for labs)
2. IAM role attached to the Jenkins EC2 instance (better practice)

This lab uses IAM user access keys.

### Step 1: Create an IAM user for Jenkins

Create IAM user: `jenkins-eks-deployer`

Attach policies (lab friendly):

* `AmazonEC2ContainerRegistryPowerUser`
* `AmazonEKSClusterPolicy` (or ensure it has `eks:DescribeCluster`)
* `AmazonEKSServicePolicy` (optional, depends on org setup)

Also ensure it can call:

* `sts:GetCallerIdentity`

### Step 2: Create access keys

Generate access keys for this IAM user and store them securely.

---

## Part 2: Configure Jenkins Credentials

In Jenkins:
Manage Jenkins -> Credentials -> System -> Global credentials -> Add credentials

Add:

### Credential A: AWS keys

* Kind: Username with password
* ID: `aws-access`
* Username: AWS_ACCESS_KEY_ID
* Password: AWS_SECRET_ACCESS_KEY

### Credential B: AWS region

* Kind: Secret text
* ID: `aws-region`
* Secret: example `ap-south-1`

### Credential C: EKS cluster name

* Kind: Secret text
* ID: `eks-cluster-name`
* Secret: example `user-mgmt-eks`

If your GitHub repo is private:

* Add GitHub PAT or SSH key credential for checkout

---

## Part 3: Prepare Kubernetes Manifests for AWS

Your repository structure should look like:

.
├── api/
├── ui/
├── k8s_solution/
│   ├── namespace.yml
│   ├── db-secret.yml
│   ├── db-pvc.yml
│   ├── db-deploy.yml
│   ├── db-svc.yml
│   ├── api-deploy.yml
│   ├── api-svc-lb.yml
│   ├── ui-configmap.yml
│   ├── ui-deploy.yml
│   └── ui-svc-lb.yml
└── Jenkinsfile

### Update image references to ECR placeholders

Update images in:

* `k8s_solution/api-deploy.yml`
* `k8s_solution/ui-deploy.yml`

Use placeholders such as:

```yaml
image: YOUR_ECR_REGISTRY/backend-user-management:latest
image: YOUR_ECR_REGISTRY/frontend-user-management:latest
```

Later, Jenkins will replace these with the actual tag.

---

## Part 4: Create Amazon ECR Repositories

Create two repositories:

* backend-user-management
* frontend-user-management

Using AWS CLI:

```bash
aws ecr create-repository --repository-name backend-user-management
aws ecr create-repository --repository-name frontend-user-management
```

---

## Part 5: Create Jenkins Pipeline (Jenkinsfile)

Create `Jenkinsfile` at the repository root.

This pipeline triggers on SCM (push webhook) or manual builds, builds and pushes images, then deploys to EKS.

```groovy
pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
  }

  environment {
    APP_NS = 'user-management'
    ECR_REPO_BACKEND = 'backend-user-management'
    ECR_REPO_FRONTEND = 'frontend-user-management'
    MANIFEST_DIR = 'k8s_solution'
    RENDER_DIR = 'k8s_rendered'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Verify tools') {
      steps {
        sh '''
          set -e
          aws --version
          kubectl version --client=true
          docker --version
          zip -v || true
        '''
      }
    }

    stage('Resolve AWS Account and ECR') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'aws-access', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-region', variable: 'AWS_REGION')
        ]) {
          sh '''
            set -e
            AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
            ECR_REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

            echo "$AWS_ACCOUNT_ID" > aws_account_id.txt
            echo "$ECR_REGISTRY" > ecr_registry.txt

            echo "AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID"
            echo "ECR_REGISTRY=$ECR_REGISTRY"
          '''
        }
      }
    }

    stage('Login to ECR') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'aws-access', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-region', variable: 'AWS_REGION')
        ]) {
          sh '''
            set -e
            ECR_REGISTRY=$(cat ecr_registry.txt)
            aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ECR_REGISTRY"
          '''
        }
      }
    }

    stage('Build and push backend image') {
      steps {
        sh '''
          set -e
          ECR_REGISTRY=$(cat ecr_registry.txt)
          IMAGE_TAG="${GIT_COMMIT:0:7}-${BUILD_NUMBER}"

          docker build -t "$ECR_REGISTRY/$ECR_REPO_BACKEND:$IMAGE_TAG" ./api
          docker push "$ECR_REGISTRY/$ECR_REPO_BACKEND:$IMAGE_TAG"

          echo "$IMAGE_TAG" > image_tag.txt
          echo "Backend pushed with tag: $IMAGE_TAG"
        '''
      }
    }

    stage('Build and push frontend image') {
      steps {
        sh '''
          set -e
          ECR_REGISTRY=$(cat ecr_registry.txt)
          IMAGE_TAG=$(cat image_tag.txt)

          docker build -t "$ECR_REGISTRY/$ECR_REPO_FRONTEND:$IMAGE_TAG" ./ui
          docker push "$ECR_REGISTRY/$ECR_REPO_FRONTEND:$IMAGE_TAG"

          echo "Frontend pushed with tag: $IMAGE_TAG"
        '''
      }
    }

    stage('Configure kubectl for EKS') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'aws-access', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-region', variable: 'AWS_REGION'),
          string(credentialsId: 'eks-cluster-name', variable: 'EKS_CLUSTER')
        ]) {
          sh '''
            set -e
            aws eks update-kubeconfig --name "$EKS_CLUSTER" --region "$AWS_REGION"
            kubectl get nodes
          '''
        }
      }
    }

    stage('Render manifests with new image tags') {
      steps {
        sh '''
          set -e
          ECR_REGISTRY=$(cat ecr_registry.txt)
          IMAGE_TAG=$(cat image_tag.txt)

          rm -rf "$RENDER_DIR"
          mkdir -p "$RENDER_DIR"
          cp -r "$MANIFEST_DIR/"* "$RENDER_DIR/"

          # Update images in manifests
          sed -i "s|image:.*backend-user-management:latest|image: $ECR_REGISTRY/$ECR_REPO_BACKEND:$IMAGE_TAG|g" "$RENDER_DIR/api-deploy.yml"
          sed -i "s|image:.*frontend-user-management:latest|image: $ECR_REGISTRY/$ECR_REPO_FRONTEND:$IMAGE_TAG|g" "$RENDER_DIR/ui-deploy.yml"

          echo "Rendered manifests:"
          ls -la "$RENDER_DIR"
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          set -e

          kubectl apply -f k8s_rendered/namespace.yml
          kubectl apply -f k8s_rendered/db-secret.yml
          kubectl apply -f k8s_rendered/db-pvc.yml
          kubectl apply -f k8s_rendered/db-deploy.yml
          kubectl apply -f k8s_rendered/db-svc.yml
          kubectl apply -f k8s_rendered/api-deploy.yml
          kubectl apply -f k8s_rendered/api-svc-lb.yml
          kubectl apply -f k8s_rendered/ui-configmap.yml
          kubectl apply -f k8s_rendered/ui-deploy.yml
          kubectl apply -f k8s_rendered/ui-svc-lb.yml

          kubectl get pods -n user-management
          kubectl get svc -n user-management
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'aws_account_id.txt,ecr_registry.txt,image_tag.txt,k8s_rendered/**', allowEmptyArchive: true, fingerprint: true
    }
  }
}
```

Commit and push:

```bash
git add Jenkinsfile
git commit -m "Add Jenkins CI/CD pipeline for EKS"
git push origin main
```

---

## Part 6: Create Jenkins Job and Trigger the Pipeline

1. Jenkins -> New Item
2. Name: `three-tier-eks-deploy`
3. Type: Pipeline
4. Pipeline -> Definition: Pipeline script from SCM
5. Git repo URL -> your repo
6. Branch -> `*/main`
7. Save

Trigger options:

* Manual: Build Now
* Automatic: Configure GitHub webhook to trigger on push

---

## Part 7: Verify Deployment

From your local machine or Jenkins agent:

```bash
kubectl get pods -n user-management
kubectl get svc -n user-management
```

Get LoadBalancer DNS:

```bash
kubectl get svc -n user-management
```

Test backend:

```bash
curl http://<BACKEND_LB_DNS>/health
```

Open frontend:

```text
http://<FRONTEND_LB_DNS>
```

---

## Cleanup

```bash
kubectl delete namespace user-management
```

Optional:

* Delete ECR repositories
* Delete IAM user access keys
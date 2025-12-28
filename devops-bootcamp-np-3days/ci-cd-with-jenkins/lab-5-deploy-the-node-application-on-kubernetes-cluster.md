# Lab 5: Deploy the Node Application on Kubernetes Cluster

## Lab Overview

In this lab, you will deploy a complete three tier application on Amazon EKS using Jenkins for CI/CD.

The pipeline will:

* Build frontend and backend Docker images
* Push images to a container registry (Amazon ECR recommended)
* Authenticate to AWS
* Configure kubectl access to the EKS cluster
* Deploy the application using Kubernetes manifests
* Validate the application end to end

## Application Architecture

Frontend (NGINX static UI) -> Backend (Node.js API) -> Database (PostgreSQL)

### Service exposure

* PostgreSQL: ClusterIP
* Backend API: LoadBalancer
* Frontend UI: LoadBalancer

## What You Will Build

You will:

* Configure Jenkins credentials for AWS and ECR
* Create a Jenkins pipeline (Jenkinsfile)
* Build and push container images
* Deploy Kubernetes manifests to EKS
* Validate the application end to end

## Prerequisites

You must have:

* An EKS cluster already running
* kubectl available on the Jenkins agent (for verification and deployment)
* AWS CLI installed on the Jenkins agent
* eksctl installed (recommended but optional)
* Docker installed on the Jenkins agent
* A Jenkins server with a Linux agent that can run Docker builds
* A GitHub repository with the application code
* IAM permissions to push to ECR and deploy to EKS

Recommended AWS services:

* ECR for container images
* EKS for Kubernetes
* IAM for authentication

## Part 1: Prepare AWS IAM Access for Jenkins

You can authenticate Jenkins to AWS using either IAM user access keys or an IAM role (recommended on EC2). This lab uses IAM user access keys because it is simplest for training.

### Step 1: Create an IAM policy for Jenkins (minimum required)

Create a policy named `JenkinsEKSDeployPolicy` with permissions for:

* ECR login, push, and repository describe
* EKS cluster describe
* STS GetCallerIdentity

You can keep it broad for training, but in real projects you should tighten it.

Minimum set of AWS managed policies for a lab style setup:

* `AmazonEC2ContainerRegistryPowerUser`
* `AmazonEKSClusterPolicy` (for cluster operations, if needed)
* `AmazonEKSWorkerNodePolicy` is not needed for Jenkins itself

Also ensure the IAM user can call `eks:DescribeCluster`.

### Step 2: Create IAM user and access keys

1. IAM -> Users -> Create user: `jenkins-eks-deployer`
2. Attach the policies above
3. Create access keys (Access key ID, Secret access key)

Store them securely.

## Part 2: Configure Jenkins Credentials

In Jenkins:

1. Manage Jenkins -> Credentials -> System -> Global credentials -> Add Credentials
2. Add these credentials:

### Credential 1: AWS Access Keys

* Kind: Username with password
* ID: `aws-access`
* Username: AWS\_ACCESS\_KEY\_ID
* Password: AWS\_SECRET\_ACCESS\_KEY

### Credential 2: AWS Region

* Kind: Secret text
* ID: `aws-region`
* Secret: example `ap-south-1`

### Credential 3: EKS Cluster Name

* Kind: Secret text
* ID: `eks-cluster-name`
* Secret: example `user-mgmt-eks`

Optional:

* ECR registry URL as secret text (or calculate it in pipeline)
* GitHub credentials if repo is private

## Part 3: Prepare Kubernetes Manifests for AWS

Your repository structure should look like:

```
// Some code
```

### Update image references to ECR placeholders

Replace Docker Hub images with ECR images:

```yaml
image: <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/backend-user-management:latest
image: <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/frontend-user-management:latest
```

You can keep placeholders and replace them during the pipeline using sed, or commit actual values.

## Part 4: Create ECR Repositories

Create two ECR repositories:

* backend-user-management
* frontend-user-management

Using AWS CLI:

```bash
aws ecr create-repository --repository-name backend-user-management
aws ecr create-repository --repository-name frontend-user-management
```

Confirm:

```bash
aws ecr describe-repositories
```

## Part 5: Create Jenkins Pipeline (Jenkinsfile)

Create a `Jenkinsfile` in the repo root.

This pipeline will:

* Checkout code
* Login to ECR
* Build and push images
* Update kubeconfig for EKS
* Apply Kubernetes manifests

```groovy
pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
  }

  environment {
    APP_NS = "user-management"
    ECR_REPO_BACKEND = "backend-user-management"
    ECR_REPO_FRONTEND = "frontend-user-management"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Load AWS settings') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'aws-access', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-region', variable: 'AWS_REGION'),
          string(credentialsId: 'eks-cluster-name', variable: 'EKS_CLUSTER')
        ]) {
          sh '''
            set -e
            aws --version
            kubectl version --client=true
            docker --version
          '''
        }
      }
    }

    stage('Get AWS Account ID') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'aws-access', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-region', variable: 'AWS_REGION')
        ]) {
          sh '''
            set -e
            AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
            echo $AWS_ACCOUNT_ID > aws_account_id.txt
            echo "AWS account: $AWS_ACCOUNT_ID"
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
            AWS_ACCOUNT_ID=$(cat aws_account_id.txt)
            ECR_REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
            aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ECR_REGISTRY"
            echo "$ECR_REGISTRY" > ecr_registry.txt
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

    stage('Deploy to EKS') {
      steps {
        sh '''
          set -e
          ECR_REGISTRY=$(cat ecr_registry.txt)
          IMAGE_TAG=$(cat image_tag.txt)

          mkdir -p k8s_rendered
          cp -r k8s_solution/* k8s_rendered/

          # Replace placeholder images if needed
          # Adjust search patterns if your manifests are different
          sed -i "s|image:.*backend-user-management:latest|image: $ECR_REGISTRY/$ECR_REPO_BACKEND:$IMAGE_TAG|g" k8s_rendered/api-deploy.yml
          sed -i "s|image:.*frontend-user-management:latest|image: $ECR_REGISTRY/$ECR_REPO_FRONTEND:$IMAGE_TAG|g" k8s_rendered/ui-deploy.yml

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
      archiveArtifacts artifacts: 'aws_account_id.txt,ecr_registry.txt,image_tag.txt,k8s_rendered/**', allowEmptyArchive: true
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

## Part 6: Create Jenkins Job and Trigger Pipeline

1. Jenkins -> New Item
2. Name: `deploy-node-app-eks`
3. Select: Pipeline
4. Pipeline definition: Pipeline script from SCM
5. Provide Git repo URL and credentials (if private)
6. Save

Trigger:

* Click Build Now
* Or configure a GitHub webhook to trigger builds on push

## Part 7: Verify Deployment

On your machine (or from Jenkins console output):

```bash
kubectl get pods -n user-management
kubectl get svc -n user-management
```

Find LoadBalancer endpoints:

Frontend:

```bash
kubectl get svc -n user-management ui-service
```

Backend:

```bash
kubectl get svc -n user-management api-service
```

Test API:

```bash
curl http://<BACKEND_LB_DNS>/health
```

Open frontend in browser:

```
http://<FRONTEND_LB_DNS>
```

## Cleanup

Delete namespace:

```bash
kubectl delete namespace user-management
```

Optional cleanup:

* Delete ECR repositories
* Delete IAM user and access keys if this was a lab only setup

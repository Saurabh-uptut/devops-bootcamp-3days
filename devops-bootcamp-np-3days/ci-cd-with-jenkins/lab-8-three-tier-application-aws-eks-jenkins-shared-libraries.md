# Lab: Deploy a Three-Tier Application on AWS EKS using Jenkins Shared Libraries

## Lab Objective

In this lab, you will deploy a complete **three-tier application** on **Amazon EKS** using **Jenkins CI/CD with Shared Libraries**.

Instead of writing a long Jenkinsfile, you will:

* Move reusable pipeline logic into a **Jenkins Shared Library**
* Keep the Jenkinsfile clean and readable
* Reuse the same library across multiple applications

---

## Application Architecture

Frontend (NGINX static UI)
→ Backend (Node.js API)
→ Database (PostgreSQL)

### Kubernetes Service Exposure

* PostgreSQL: `ClusterIP`
* Backend API: `LoadBalancer`
* Frontend UI: `LoadBalancer`

---

## What You Will Build

You will:

* Create a Jenkins Shared Library for AWS + EKS pipelines
* Configure Jenkins credentials for AWS and ECR
* Build and push Docker images to Amazon ECR
* Configure kubectl for EKS from Jenkins
* Deploy Kubernetes manifests to EKS
* Validate the application end to end

---

## Prerequisites

You must already have:

* An **EKS cluster** running
* Jenkins server with a Linux agent
* Docker, AWS CLI, kubectl installed on the Jenkins agent
* Git repository with:

  * `api/`
  * `ui/`
  * `k8s_solution/`
* Amazon ECR repositories:

  * `backend-user-management`
  * `frontend-user-management`

---

## PART 1: Create Jenkins Shared Library

### Step 1.1: Create a Git Repository for the Shared Library

Create a repository named:

```
jenkins-eks-shared-library
```

### Step 1.2: Shared Library Structure

```
jenkins-eks-shared-library/
├── vars/
│   ├── awsLogin.groovy
│   ├── ecrLogin.groovy
│   ├── buildAndPushImage.groovy
│   ├── configureEksKubectl.groovy
│   ├── renderManifests.groovy
│   └── deployManifests.groovy
├── src/
│   └── org/
│       └── eks/
│           └── AwsHelper.groovy
└── README.md
```

---

## PART 2: Implement Shared Library Steps

### Step 2.1: AWS Account Resolution

`vars/awsLogin.groovy`

```groovy
def call() {
    sh '''
      set -e
      AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      echo "$AWS_ACCOUNT_ID" > aws_account_id.txt
      echo "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com" > ecr_registry.txt
    '''
}
```

---

### Step 2.2: Login to Amazon ECR

`vars/ecrLogin.groovy`

```groovy
def call() {
    sh '''
      set -e
      ECR_REGISTRY=$(cat ecr_registry.txt)
      aws ecr get-login-password --region "$AWS_REGION" \
        | docker login --username AWS --password-stdin "$ECR_REGISTRY"
    '''
}
```

---

### Step 2.3: Build and Push Docker Image

`vars/buildAndPushImage.groovy`

```groovy
def call(String appDir, String repoName) {
    sh """
      set -e
      ECR_REGISTRY=\$(cat ecr_registry.txt)
      IMAGE_TAG="\${GIT_COMMIT:0:7}-\${BUILD_NUMBER}"

      docker build -t "\$ECR_REGISTRY/$repoName:\$IMAGE_TAG" $appDir
      docker push "\$ECR_REGISTRY/$repoName:\$IMAGE_TAG"

      echo "\$IMAGE_TAG" > image_tag.txt
    """
}
```

---

### Step 2.4: Configure kubectl for EKS

`vars/configureEksKubectl.groovy`

```groovy
def call(String clusterName) {
    sh """
      set -e
      aws eks update-kubeconfig --name "$clusterName" --region "$AWS_REGION"
      kubectl get nodes
    """
}
```

---

### Step 2.5: Render Kubernetes Manifests

`vars/renderManifests.groovy`

```groovy
def call(String manifestDir, String renderDir) {
    sh """
      set -e
      ECR_REGISTRY=\$(cat ecr_registry.txt)
      IMAGE_TAG=\$(cat image_tag.txt)

      rm -rf $renderDir
      mkdir -p $renderDir
      cp -r $manifestDir/* $renderDir/

      sed -i "s|backend-user-management:latest|backend-user-management:\$IMAGE_TAG|g" $renderDir/api-deploy.yml
      sed -i "s|frontend-user-management:latest|frontend-user-management:\$IMAGE_TAG|g" $renderDir/ui-deploy.yml
    """
}
```

---

### Step 2.6: Deploy to EKS

`vars/deployManifests.groovy`

```groovy
def call(String renderDir) {
    sh """
      set -e
      kubectl apply -f $renderDir
      kubectl get pods -n user-management
      kubectl get svc -n user-management
    """
}
```

---

### Step 2.7: Push the Shared Library

```bash
git add .
git commit -m "Initial Jenkins shared library for EKS deployments"
git push origin main
```

---

## PART 3: Configure Shared Library in Jenkins

1. Jenkins Dashboard → Manage Jenkins → Configure System
2. Global Pipeline Libraries → Add

Configuration:

* Name: `eks-shared-lib`
* Default version: `main`
* Retrieval method: Modern SCM
* SCM: Git
* Repository URL: `<shared-library-repo-url>`
* Credentials: if required

Save.

---

## PART 4: Jenkinsfile (Thin and Clean)

Create `Jenkinsfile` in your application repository:

```groovy
@Library('eks-shared-lib') _

pipeline {
  agent any

  environment {
    AWS_REGION = credentials('aws-region')
    APP_NS = 'user-management'
    MANIFEST_DIR = 'k8s_solution'
    RENDER_DIR = 'k8s_rendered'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('AWS Init') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'aws-access',
            usernameVariable: 'AWS_ACCESS_KEY_ID',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
          )
        ]) {
          awsLogin()
          ecrLogin()
        }
      }
    }

    stage('Build & Push Images') {
      steps {
        buildAndPushImage('api', 'backend-user-management')
        buildAndPushImage('ui', 'frontend-user-management')
      }
    }

    stage('Configure EKS') {
      steps {
        withCredentials([
          string(credentialsId: 'eks-cluster-name', variable: 'EKS_CLUSTER')
        ]) {
          configureEksKubectl(EKS_CLUSTER)
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        renderManifests(MANIFEST_DIR, RENDER_DIR)
        deployManifests(RENDER_DIR)
      }
    }
  }
}
```

---

## PART 5: Run the Pipeline

1. Jenkins → New Item
2. Name: `three-tier-eks-sharedlib`
3. Type: Pipeline
4. Pipeline script from SCM
5. Repo URL → your application repo
6. Branch: `main`
7. Save → Build Now

---

## PART 6: Verify Deployment

```bash
kubectl get pods -n user-management
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

## Why Shared Libraries Matter (Key Learning)

* Jenkinsfile reduced from 300+ lines to ~50 lines
* CI/CD logic is reusable across teams
* Changes happen in one place
* Cleaner reviews and safer pipelines
* Enterprise-grade Jenkins design

---

## Lab Completion Criteria

You have successfully completed this lab if:

* Jenkins Shared Library is configured
* Jenkinsfile only orchestrates stages
* Images are pushed to ECR
* Kubernetes manifests deploy successfully
* Application is accessible via LoadBalancers

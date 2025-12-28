# Lab 2: Continuous Integration for a Node.js Application

Build, Test, Generate Reports, and Publish Artifacts Using Jenkins Pipeline

This lab walks you through creating a complete CI pipeline for a Node.js application using Jenkins. The pipeline will install dependencies, run tests, generate JUnit and coverage reports, and publish build artifacts including a deployment ready ZIP file.

## Pre-Requisites

* A GitHub account and a sample Node.js repo (provided by your instructor)
* Jenkins installed and running (VM, server, or Docker)
* A Jenkins agent that can run Node builds (Linux recommended)
* Tools available on the Jenkins agent:
  * Git
  * Node.js 18 and Node.js 20 (or ability to install via Jenkins tool configuration)
  * zip
* Jenkins plugins:
  * Pipeline
  * Git
  * NodeJS (recommended)
  * JUnit
  * HTML Publisher (recommended, for coverage reports)
  * Credentials Binding (recommended)

Note: If your tests produce `coverage/junit.xml` and coverage reports under `coverage/`, this lab will work as is. If your repo uses a different path, adjust in the Jenkinsfile.

## Learning Objectives

1. Understand what CI is and why teams use it.
2. Create a Jenkins Pipeline for a Node.js application.
3. Automatically build and test a Node.js app in Jenkins.
4. Publish JUnit test results and coverage reports in Jenkins.
5. Publish CI artifacts including a deployment ZIP package.
6. Run the pipeline across Node.js versions 18 and 20 (matrix style).

## Step 1: Fork and Clone the Application

1. Fork the sample application repository provided by your instructor.
2. Clone the repository to your local machine.

```bash
git clone <repository-url>
cd <repository-folder>
```

## Step 2: Prepare Jenkins for Node Builds

### Option A: Install NodeJS tools inside Jenkins (recommended)

1. Jenkins Dashboard -> Manage Jenkins -> Tools
2. Find NodeJS installations
3. Add:
   * `node-18`
   * `node-20`
4. If you have internet access from Jenkins, you can enable automatic install.

### Option B: Use system Node on the agent

Make sure these commands work on the agent:

```bash
node -v
npm -v
zip -v
```

If you only have one Node version installed, you can still run the pipeline, but the matrix part will not truly validate both versions.

## Step 3: Add a Jenkinsfile to the Repo

Create a file named `Jenkinsfile` in the repository root.

Use this Jenkins Pipeline. It runs builds for Node 18 and Node 20 in parallel, publishes JUnit test results, archives coverage and ZIP artifacts, and creates a deployment package similar to your GitHub Actions workflow.

```groovy
pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
  }

  parameters {
    choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Environment tag for the build')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build and Test Matrix') {
      parallel {
        stage('Node 18') {
          steps {
            runNodeCi('18')
          }
        }

        stage('Node 20') {
          steps {
            runNodeCi('20')
          }
        }
      }
    }
  }

  post {
    always {
      echo "Build completed with result: ${currentBuild.currentResult}"
    }
  }
}

/**
 * CI function for a specific Node major version.
 * Expects tests to produce coverage/junit.xml and coverage content.
 */
def runNodeCi(String nodeMajor) {
  // If you configured Jenkins NodeJS tools, it will use them.
  // Otherwise it will fallback to system node (you can remove NodeJS tool lines if not using plugin).
  def nodeToolName = "node-${nodeMajor}"

  stage("Setup Node ${nodeMajor}") {
    script {
      try {
        def nodeHome = tool name: nodeToolName, type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
        env.PATH = "${nodeHome}/bin:${env.PATH}"
      } catch (ignored) {
        echo "Found no Jenkins NodeJS tool named ${nodeToolName}. Using system Node on agent."
      }
    }

    sh '''
      set -e
      node -v || true
      npm -v || true
      zip -v  || true
    '''
  }

  stage("Install deps (Node ${nodeMajor})") {
    sh '''
      set -e
      npm ci
    '''
  }

  stage("Build (optional) (Node ${nodeMajor})") {
    sh '''
      set +e
      npm run build 2>/dev/null
      if [ $? -ne 0 ]; then
        echo "No build script found, skipping build step"
      fi
      set -e
    '''
  }

  stage("Test (Node ${nodeMajor})") {
    sh '''
      set -e
      mkdir -p coverage
      export JEST_JUNIT_OUTPUT=coverage/junit.xml
      npm run test:ci
    '''
  }

  stage("Publish test results (Node ${nodeMajor})") {
    // JUnit publisher reads XML and shows it in Jenkins UI.
    // allowEmptyResults helps if tests fail before report generation; adjust based on preference.
    junit testResults: 'coverage/junit.xml', allowEmptyResults: true
  }

  stage("Archive coverage (Node ${nodeMajor})") {
    // Store coverage reports and junit for download
    archiveArtifacts artifacts: 'coverage/**', allowEmptyArchive: true, fingerprint: true

    // Optional: Publish HTML coverage report if present
    script {
      if (fileExists('coverage/lcov-report/index.html')) {
        publishHTML(target: [
          allowMissing: false,
          alwaysLinkToLastBuild: true,
          keepAll: true,
          reportDir: 'coverage/lcov-report',
          reportFiles: 'index.html',
          reportName: "Coverage Report (Node ${nodeMajor})"
        ])
      } else {
        echo "No HTML coverage report found at coverage/lcov-report/index.html"
      }
    }
  }

  stage("Create deployment package (Node ${nodeMajor})") {
    sh """
      set -euo pipefail

      STAGING="\$WORKSPACE/deployment-package-node-${nodeMajor}"
      rm -rf "\$STAGING"
      mkdir -p "\$STAGING"

      [ -d public ] && cp -r public "\$STAGING/" || echo "No public directory found"
      [ -d node_modules ] && cp -r node_modules "\$STAGING/" || echo "No node_modules directory found"
      [ -f server.js ] && cp server.js "\$STAGING/" || echo "No server.js found"
      [ -f package.json ] && cp package.json "\$STAGING/" || echo "No package.json found"
      [ -f package-lock.json ] && cp package-lock.json "\$STAGING/" || echo "No package-lock.json found"
      [ -f README.md ] && cp README.md "\$STAGING/" || true
      [ -f Dockerfile ] && cp Dockerfile "\$STAGING/" || echo "No Dockerfile found"

      (cd "\$STAGING" && npm pkg delete devDependencies || true)

      TS="\$(date +%Y%m%d-%H%M%S)"
      SHORT_SHA="\$(git rev-parse --short HEAD)"
      ZIP_NAME="deployment-package-node-${nodeMajor}-\${SHORT_SHA}-\${TS}.zip"

      (cd "\$WORKSPACE" && zip -r "\$ZIP_NAME" "deployment-package-node-${nodeMajor}")

      echo "Created ZIP: \$ZIP_NAME"
    """
  }

  stage("Archive deployment ZIP (Node ${nodeMajor})") {
    archiveArtifacts artifacts: 'deployment-package-node-*.zip', allowEmptyArchive: false, fingerprint: true
  }
}
```

Commit and push:

```bash
git add Jenkinsfile
git commit -m "Add Jenkins CI pipeline"
git push
```

## Step 4: Create the Jenkins Pipeline Job

1. Jenkins -> New Item
2. Name: `nodejs-ci-pipeline`
3. Select: **Pipeline**
4. Click OK

### Configure

* Under **Pipeline**:
  * Definition: **Pipeline script from SCM**
  * SCM: **Git**
  * Repository URL: your fork URL
  * Branch: `*/main`
  * Credentials: add if private repo

Click Save.

## Step 5: Run the Pipeline

1. Open the job
2. Click **Build Now**
3. Open the build
4. Click **Console Output**

You will see two parallel branches:

* Node 18
* Node 20

You will also see:

* JUnit test report in Jenkins Test Results
* Coverage artifacts archived
* Deployment ZIP archived for download

## Step 6: Download and Run the Deployment Artifact

1. Open the Jenkins build page
2. Under **Artifacts**, download:
   * `deployment-package-node-18-<sha>-<timestamp>.zip` or
   * `deployment-package-node-20-<sha>-<timestamp>.zip`
3. Extract it
4. Go into the extracted folder:

```bash
cd deployment-package-node-18-*   # or node-20
npm start
```

5. Access the application in the browser using your machine or VM IP and exposed port.

## Understanding the Jenkins Pipeline

### What the pipeline does

* Checks out code from GitHub
* Runs CI in parallel for Node 18 and Node 20
* Installs dependencies using `npm ci`
* Runs optional build step
* Runs tests using `npm run test:ci` generating `coverage/junit.xml`
* Publishes JUnit results to Jenkins UI
* Archives coverage folder as an artifact
* Creates a deployment staging directory and produces a ZIP
* Archives the ZIP for download

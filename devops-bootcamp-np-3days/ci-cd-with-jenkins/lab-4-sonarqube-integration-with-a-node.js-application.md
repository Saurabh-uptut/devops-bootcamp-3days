# Lab 4: SonarQube Integration with a Node.js Application using Jenkins

In this lab, you will create a Jenkins Pipeline that builds, tests, and scans a Node.js application using SonarQube. You will also publish JUnit test results, coverage artifacts, and a deployment ready ZIP package from Jenkins.

---

## Pre-Requisites

* A GitHub account
* SonarQube set up and running (from the previous lab)
* Jenkins installed and running
* Jenkins agent (Linux recommended) with:

  * Git
  * Node.js 18
  * npm
  * zip
* Jenkins plugins:

  * Pipeline
  * Git
  * JUnit
  * Credentials Binding (recommended)
  * SonarQube Scanner for Jenkins (recommended)
  * NodeJS (optional, if you want Jenkins managed Node installs)
  * HTML Publisher (optional, for coverage HTML report)

---

## Learning Objectives

1. Create a Jenkins Pipeline for a Node.js CI process
2. Integrate SonarQube code scanning into a Jenkins pipeline
3. Publish test results and deployment artifacts in Jenkins
4. Store SonarQube credentials securely using Jenkins credentials
5. Understand how CI, testing, packaging, and code analysis work together

---

## Step 1: Prepare the Node.js Application

1. Fork the sample Node.js application repository provided by your instructor.
2. Clone it locally:

```bash
git clone <repository-url>
cd <repository-folder>
```

3. Create a `Jenkinsfile` in the repo root.

---

## Step 2: Configure SonarQube in Jenkins

### Step 2.1: Add SonarQube Server in Jenkins

1. Jenkins Dashboard -> Manage Jenkins -> System
2. Find **SonarQube servers**
3. Click **Add SonarQube**
4. Provide:

   * Name: `sonarqube-dev`
   * Server URL: your SonarQube URL (example: `http://sonarqube:9000`)
   * Server authentication token:

     * Add a Jenkins credential:

       * Kind: Secret text
       * ID: `sonar-token-dev`
       * Value: your SonarQube token

Save.

### Step 2.2: Install SonarScanner Tool in Jenkins

1. Manage Jenkins -> Tools
2. Find **SonarQube Scanner**
3. Add installation:

   * Name: `sonar-scanner`
   * Enable auto install if available, or configure a scanner path on agent

Save.

---

## Step 3: Add the Jenkinsfile (CI + Tests + Sonar + Artifacts)

Create `Jenkinsfile` with the following Pipeline. This mirrors your GitHub Actions workflow: checkout, Node setup, install, optional build, tests with JUnit, SonarQube scan, package ZIP, archive artifacts.

```groovy
pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
  }

  environment {
    NODE_ENV = 'test'
    // Sonar metadata
    SONAR_PROJECT_KEY = 'sample-node-app-saurabh'
    SONAR_ORG = 'sda'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Verify toolchain') {
      steps {
        sh '''
          set -e
          node -v
          npm -v
          zip -v
        '''
      }
    }

    stage('Install dependencies') {
      steps {
        sh '''
          set -e
          npm ci
        '''
      }
    }

    stage('Build (optional)') {
      steps {
        sh '''
          set +e
          npm run build 2>/dev/null
          if [ $? -ne 0 ]; then
            echo "No build script found, skipping build step"
          fi
          set -e
        '''
      }
    }

    stage('Test with coverage + JUnit') {
      steps {
        sh '''
          set -e
          mkdir -p coverage
          export JEST_JUNIT_OUTPUT=coverage/junit.xml
          npm run test:ci
        '''
      }
      post {
        always {
          junit testResults: 'coverage/junit.xml', allowEmptyResults: true
          archiveArtifacts artifacts: 'coverage/**', allowEmptyArchive: true, fingerprint: true

          script {
            if (fileExists('coverage/lcov-report/index.html')) {
              publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'coverage/lcov-report',
                reportFiles: 'index.html',
                reportName: 'Coverage Report'
              ])
            } else {
              echo 'No HTML coverage report found at coverage/lcov-report/index.html'
            }
          }
        }
      }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv('sonarqube-dev') {
          script {
            def scannerHome = tool 'sonar-scanner'
            sh """
              set -e

              ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                -Dsonar.organization=${env.SONAR_ORG} \
                -Dsonar.sources=. \
                -Dsonar.exclusions=node_modules/**,coverage/**,tests/**,**/*.test.js,**/*.spec.js \
                -Dsonar.tests=tests/ \
                -Dsonar.test.inclusions=**/*.test.js,**/*.spec.js \
                -Dsonar.coverage.exclusions=node_modules/**,coverage/**,tests/**,**/*.test.js,**/*.spec.js
            """
          }
        }
      }
    }

    stage('Quality Gate (optional)') {
      steps {
        script {
          // If your Jenkins is integrated with SonarQube webhooks, this can enforce quality gates.
          // If not configured, you can remove this stage.
          timeout(time: 10, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Pipeline failed due to SonarQube Quality Gate: ${qg.status}"
            }
          }
        }
      }
    }

    stage('Create deployment package') {
      steps {
        sh '''
          set -euo pipefail

          STAGING="$WORKSPACE/deployment-package"
          rm -rf "$STAGING"
          mkdir -p "$STAGING"

          [ -d public ] && cp -r public "$STAGING/" || echo "No public directory found"
          [ -d node_modules ] && cp -r node_modules "$STAGING/" || echo "No node_modules directory found"
          [ -f server.js ] && cp server.js "$STAGING/" || echo "No server.js found"
          [ -f package.json ] && cp package.json "$STAGING/" || echo "No package.json found"
          [ -f package-lock.json ] && cp package-lock.json "$STAGING/" || echo "No package-lock.json found"
          [ -f README.md ] && cp README.md "$STAGING/" || true
          [ -f Dockerfile ] && cp Dockerfile "$STAGING/" || echo "No Dockerfile found"

          (cd "$STAGING" && npm pkg delete devDependencies || true)

          TS="$(date +%Y%m%d-%H%M%S)"
          SHORT_SHA="$(git rev-parse --short HEAD)"
          ZIP_NAME="deployment-package-${SHORT_SHA}-${TS}.zip"

          (cd "$WORKSPACE" && zip -r "$ZIP_NAME" "deployment-package")
          echo "$ZIP_NAME" > deployment_zip_name.txt
          echo "Created ZIP: $ZIP_NAME"
        '''
      }
    }

    stage('Archive deployment artifact') {
      steps {
        sh 'cat deployment_zip_name.txt'
        archiveArtifacts artifacts: 'deployment-package-*.zip, deployment_zip_name.txt', fingerprint: true
      }
    }
  }

  post {
    always {
      echo "Result: ${currentBuild.currentResult}"
    }
  }
}
```

What this Jenkinsfile expects:

* `npm run test:ci` generates `coverage/junit.xml`
* Coverage output exists under `coverage/`
* You have a SonarQube server configured in Jenkins as `sonarqube-dev`
* You have a SonarScanner tool configured as `sonar-scanner`

If your repo uses different test commands or report paths, update the Jenkinsfile accordingly.

---

## Step 4: Create a Jenkins Pipeline Job

1. Jenkins -> New Item
2. Name: `nodejs-ci-sonar`
3. Type: **Pipeline**
4. Click OK

### Configure the job

* Under Pipeline:

  * Definition: Pipeline script from SCM
  * SCM: Git
  * Repository URL: your fork
  * Branch: `*/main`
  * Credentials: add if private

Save.

---

## Step 5: Run the Pipeline

1. Open the job
2. Click Build Now
3. Inspect:

   * Console Output
   * Test Result Trend
   * Coverage Report (if HTML published)
   * Artifacts section for ZIP download

---

## Step 6: Review the SonarQube Report

1. Log in to SonarQube
2. Go to Projects
3. Open: `sample-node-app-saurabh`
4. Review:

   * Bugs
   * Vulnerabilities
   * Code smells
   * Security hotspots
   * Coverage
   * Duplications

---

## Step 7: Download and Run the Deployment Artifact

1. From Jenkins build page, download the ZIP from Artifacts
2. Extract it
3. Run:

```bash
cd deployment-package
npm start
```

Open the app in your browser using your machine or VM IP.

---

## Completion

You have successfully completed the lab by:

* Building and testing a Node.js application in Jenkins
* Publishing JUnit test results and coverage artifacts
* Running a SonarQube scan from Jenkins
* Optionally enforcing a SonarQube quality gate
* Creating and archiving a deployment ready ZIP artifact for download and deployment
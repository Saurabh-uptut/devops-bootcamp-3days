# Lab 1: Your First Jenkins Pipeline

This lab introduces you to Jenkins and guides you through creating your first Pipeline job. You will learn Pipeline syntax, triggers, stages, agents, and common best practices. You will then extend your knowledge by working with parallel builds and scheduled, parameterized runs.

## Prerequisites

* Git installed on your machine
* A text editor such as Visual Studio Code
* A GitHub account with valid credentials (repo source)
* Jenkins installed and running (local VM, Docker, or server)
* Jenkins plugins:
  * Pipeline
  * Git
  * (Recommended) GitHub plugin
  * (Recommended) Credentials Binding

## Step 1: Create a New GitHub Repository

1. Log in to your GitHub account.
2. Create a new repository.
3. Enable the option to create a README.md file.
4. Add some text to README.md and commit.

This will serve as the initial content for your repository.

## Step 2: Create Your First Jenkins Pipeline (Jenkinsfile)

### Step 2.1: Add a Jenkinsfile to your repository

In your repository root, create a file named `Jenkinsfile` with this content:

```groovy
pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('One-line script') {
      steps {
        sh 'echo Hello, world!'
      }
    }

    stage('Multi-line script') {
      steps {
        sh '''
          echo Add other steps to build,
          echo test, and deploy your project.
        '''
      }
    }
  }
}
```

Commit and push:

```bash
git add Jenkinsfile
git commit -m "Add first Jenkins pipeline"
git push
```

### Step 2.2: Create a Pipeline job in Jenkins

1. In Jenkins, click **New Item**.
2. Enter a name: `first-jenkins-pipeline`
3. Select **Pipeline** and click **OK**.

### Step 2.3: Connect Jenkins to your GitHub repo

In the job configuration:

* Under **Pipeline**:
  * Definition: **Pipeline script from SCM**
  * SCM: **Git**
  * Repository URL: paste your repo URL
  * Branch: `*/main`
* If the repository is private:
  * Add **Credentials** (Username/Password or GitHub token).

Click **Save**.

### Step 2.4: Run and view results

1. Click **Build Now**.
2. Click the build number in the left pane.
3. Click **Console Output** to inspect logs and output.

You should see the echo output from both stages.

## Step 3: Modify the Pipeline to Run Only Manually

In GitHub Actions you removed push triggers. In Jenkins, the equivalent is: do not configure webhooks, polling, or scheduled triggers.

1. Go to Jenkins job → **Configure**
2. Ensure you DO NOT enable:
   * **GitHub hook trigger for GITScm polling**
   * **Poll SCM**
   * **Build periodically**

Save.

Now the job runs only when you click **Build Now**.

Tip: If you previously added a webhook in GitHub, remove it: Repo → Settings → Webhooks → delete Jenkins webhook (optional).

## Step 4: More Examples in Jenkins

### Example 1: Matrix-like Builds using Parallel Stages

Update your `Jenkinsfile` to this:

```groovy
pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Matrix Build (Parallel)') {
      parallel {
        stage('Ubuntu Node 18') {
          steps {
            sh '''
              echo "Simulating: Ubuntu + Node 18"
              node -v || true
              echo "Node 18 steps would run here"
            '''
          }
        }

        stage('Ubuntu Node 20') {
          steps {
            sh '''
              echo "Simulating: Ubuntu + Node 20"
              node -v || true
              echo "Node 20 steps would run here"
            '''
          }
        }
      }
    }

    stage('Print environment') {
      steps {
        sh '''
          echo "Jenkins Node Name: $NODE_NAME"
          echo "Workspace: $WORKSPACE"
        '''
      }
    }
  }
}
```

Commit and push, then run the job in Jenkins.

What you will observe:

* The “Matrix Build” runs multiple branches in parallel (similar to matrix jobs).

Note: If you truly want different OS runs (Linux vs Windows), you need Jenkins agents for each OS and use `agent { label 'windows' }` per stage.&#x20;

### Example 2: Scheduled and Parameterized Pipeline

In Jenkins, use:

* parameters (`parameters { ... }`)
* cron trigger (`triggers { cron('...') }`)

Create or replace your `Jenkinsfile` with:

```groovy
pipeline {
  agent any

  parameters {
    choice(
      name: 'ENVIRONMENT',
      choices: ['dev', 'staging', 'prod'],
      description: 'Which environment?'
    )
    string(
      name: 'MESSAGE',
      defaultValue: 'Hello from manual run',
      description: 'Message to print'
    )
  }

  triggers {
    cron('H 0 * * 1')
  }

  stages {
    stage('Print context') {
      steps {
        sh """
          echo "Build Cause (may include timer/manual):"
          echo "${currentBuild.rawBuild.getCauses()}"
          echo "Env input: ${params.ENVIRONMENT}"
          echo "Msg: ${params.MESSAGE}"
        """
      }
    }
  }
}
```

Commit and push.

#### Run manually with inputs

1. Jenkins job page: click **Build with Parameters**
2. Select:
   * ENVIRONMENT = `staging`
   * MESSAGE = `Testing manual run`
3. Click **Build**

#### What this pipeline does

* Runs automatically every Monday around 00:00 (Jenkins uses `H` to spread load)
* Also supports manual runs with inputs
* Prints the cause and parameter values

If you want it to be exactly Monday 00:00 without hashing:

* Use `cron('0 0 * * 1')`

## Completion

You have successfully:

1. Created your first Jenkins Pipeline using a Jenkinsfile
2. Learned core Pipeline structure: agent, stages, steps
3. Configured it to run only manually
4. Implemented matrix-style execution using parallel stages
5. Created scheduled and input-based pipeline runs with parameters

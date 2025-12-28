Below is a **clean, step-by-step lab** for **Jenkins Shared Libraries**, written in a clear instructional format and ready to be used in training or self-practice.

---

# Lab: Creating and Using Jenkins Shared Libraries

## Lab Objective

In this lab, you will create a **Jenkins Shared Library**, publish it to Git, configure it in Jenkins, and use it inside a Jenkins Pipeline to reuse common pipeline logic.

---

## What You Will Learn

By the end of this lab, you will be able to:

* Understand the structure of a Jenkins Shared Library
* Create reusable pipeline steps using the `vars/` directory
* Create helper classes using the `src/` directory
* Configure a shared library globally in Jenkins
* Import and use the shared library inside a Jenkins Pipeline

---

## Prerequisites

* Jenkins installed and running
* Access to **Manage Jenkins**
* A Git repository hosting service (GitHub, GitLab, Bitbucket, etc.)
* Basic knowledge of Jenkins Pipelines and Groovy
* Git installed locally

---

## Step 1: Understand the Jenkins Shared Library Structure

A Jenkins Shared Library is hosted in a Git repository and follows a **specific directory structure**.

### Standard Shared Library Layout

```
(root)
├── vars/
│   └── helloWorld.groovy
├── src/
│   └── org/
│       └── example/
│           └── MyHelperClass.groovy
├── resources/
├── README.md
```

### Purpose of Each Directory

* **vars/**

  * Contains globally accessible pipeline steps
  * Each `.groovy` file becomes a callable function
* **src/**

  * Contains helper classes and complex logic
  * Must follow standard Java package structure
* **resources/**

  * Holds static files such as templates, JSON, YAML, scripts
* **README.md**

  * Documents how the library is used

---

## Step 2: Create the Shared Library Repository

1. Create a new Git repository
   Example name:

   ```
   jenkins-shared-library
   ```

2. Clone the repository locally:

   ```bash
   git clone <repository-url>
   cd jenkins-shared-library
   ```

3. Create the directory structure:

   ```bash
   mkdir -p vars src/org/example resources
   touch README.md
   ```

---

## Step 3: Create a Custom Pipeline Step (vars directory)

### Create `vars/helloWorld.groovy`

```groovy
def call(String name = 'World') {
    echo "Hello, ${name}!"
}
```

### Explanation

* The `call()` method makes this file callable as a pipeline step
* You can invoke it directly as `helloWorld('Name')`
* The function is available globally to all pipelines using this library

---

## Step 4: Create a Helper Class (src directory)

### Create `src/org/example/MyHelperClass.groovy`

```groovy
package org.example

class MyHelperClass {
    static String greet(String name) {
        return "Hello, ${name}!"
    }
}
```

### Explanation

* Classes under `src/` are imported and used inside `script {}` blocks
* Logic here is reusable and testable
* Use helper classes for complex operations

---

## Step 5: Push the Shared Library to Git

```bash
git add .
git commit -m "Initial Jenkins shared library"
git branch -M main
git push origin main
```

Your shared library is now available in Git.

---

## Step 6: Configure the Shared Library in Jenkins

1. Open **Jenkins Dashboard**
2. Navigate to:

   ```
   Manage Jenkins → Configure System
   ```
3. Scroll to **Global Pipeline Libraries**
4. Click **Add**

### Configuration Values

* **Name**

  ```
  shared-library
  ```

* **Default Version**

  ```
  main
  ```

* **Retrieval Method**

  ```
  Modern SCM
  ```

* **SCM**

  * Select **Git**
  * Repository URL: `<your-shared-library-repo-url>`
  * Credentials: Select credentials if required

5. Click **Save**

---

## Step 7: Use the Shared Library in a Jenkins Pipeline

### Example Jenkinsfile

```groovy
@Library('shared-library') _

pipeline {
    agent any

    stages {
        stage('Hello World') {
            steps {
                helloWorld('Jenkins User')
            }
        }

        stage('Using Helper Class') {
            steps {
                script {
                    def message = org.example.MyHelperClass.greet('Jenkins User')
                    echo message
                }
            }
        }
    }
}
```

### Key Notes

* `@Library('shared-library') _` loads the library
* Steps from `vars/` are callable directly
* Classes from `src/` are accessed using their package name
* Helper classes must be used inside `script {}` blocks

---

## Step 8: Test the Shared Library

1. Create or update a Jenkins Pipeline job
2. Paste the Jenkinsfile or reference it from SCM
3. Run the pipeline

### Expected Output

```
Hello, Jenkins User!
Hello, Jenkins User!
```

This confirms:

* Shared library loaded successfully
* Custom step executed correctly
* Helper class imported and used properly

---

## Best Practices

* Keep business logic in `src/`, not in pipelines
* Keep `vars/` steps simple and readable
* Version your shared libraries using Git tags
* Use one shared library across multiple teams
* Document usage clearly in `README.md`

---

## Lab Completion Criteria

You have successfully completed this lab if:

* Jenkins Shared Library is configured globally
* Custom step from `vars/` executes successfully
* Helper class from `src/` works inside a pipeline
* The same library can be reused across pipelines
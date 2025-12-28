# Lab 8: Multi-Stage Build and Deployment of a Java Application as a Docker Container

In this lab, you will create a multi-stage Dockerfile, build a Docker image for a Java Spring Boot application, and deploy it as a container. You will also apply a security best practice by running the application as a non-root user in the final image.

Multi-stage builds allow you to compile your application in one image (which contains heavy build tools like Maven) and run it in a second, lightweight image containing only what is required at runtime. This approach reduces image size and increases security.

### Why Use Multi-Stage Images

* Smaller final images which improve deployment speed
* More secure because no build tools or compilers are included in production
* Better caching during builds
* Cleaner separation between build and run environments

### How Multi-Stage Builds Work

1. Stage 1: The build stage uses a full-featured image (for example Maven) to compile the source code.
2. Stage 2: The runtime stage uses a lightweight JRE-only image to run the application.
3. The compiled JAR file is copied from the build stage into the runtime stage.

### Pre-Requisite

A virtual machine or workstation with Docker installed.

### Learning Objectives

1. Create a multi-stage Dockerfile for a Java application.
2. Build a Docker image for a Spring Boot application using Maven.
3. Run a Java container from the built image.
4. Apply security best practices by running as a non-root user.

### Step 1: Clone the Application and Prepare the Project

Clone the sample Java Spring Boot repository from here - [Multistage Build](https://github.com/saurabhd2106/ecommerce-java-app-ih.git)

Switch into the project directory. The project contains a Spring Boot application using Maven.

If you are new to Maven, refer to the official quick start guide:

[https://maven.apache.org/guides/getting-started/](https://maven.apache.org/guides/getting-started/)

### Step 2: Create the Dockerfile

Inside the project directory, create a file named Dockerfile (no extension). Add the following content.

```
# Multi-stage build for Spring Boot application

# Stage 1: Build stage
FROM maven:3.9.9-eclipse-temurin-21 AS builder

LABEL maintainer="saurabh@uptut.com"

WORKDIR /app

# Copy the pom.xml first for layer caching
COPY pom.xml .

# Download dependencies
RUN mvn dependency:go-offline -B

# Copy the complete source code
COPY src ./src

# Build the application
RUN mvn package -DskipTests

# Stage 2: Runtime stage
FROM eclipse-temurin:21-jre-alpine AS runtime

LABEL maintainer="Saurabh"

# Create non-root user for security
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# Copy the JAR built in the previous stage
COPY --from=builder /app/target/ecommerce-app-1.1-SNAPSHOT.jar app.jar

# Fix ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose application port
EXPOSE 8080

# Start the application
CMD ["java", "-jar", "app.jar"]
```

### Step 3: Explanation of the Dockerfile

#### Stage 1: Build the Application

* Uses Maven with Java 21 for compiling the application.
* Copies only pom.xml initially so Docker can cache dependency layers.
* Downloads all dependencies.
* Copies the source code and compiles it into a JAR file stored in the target directory.

#### Stage 2: Run the Application

* Uses a minimal Java 21 JRE base image.
* Creates a non-root user named appuser to improve security.
* Copies the JAR from the build stage into the runtime image.
* Changes file ownership and switches to the non-root user.
* Exposes port 8080 and runs the application.

#### Benefits

* Final image is much smaller and more secure.
* Build tools are not included in production.
* Faster rebuilds due to layer caching.

#### Important Note

Ensure the JAR name ecommerce-app-1.1-SNAPSHOT.jar matches the actual JAR produced by Maven.\
If your JAR version changes frequently, you can use:

```
COPY --from=builder /app/target/*.jar app.jar
```

### Step 4: Build the Docker Image

Run the following command from the directory containing the Dockerfile:

```bash
docker build -t <dockerhub_username>/java-app-ms .
```

Example:

```bash
docker build -t saurabhd2106/java-app-ms .
```

Docker will execute both stages and produce a final optimized runtime image.

### Step 5: Run the Docker Container

Start the application using:

```bash
docker run -d -p 80:8080 <dockerhub_username>/java-app-ms
```

Example:

```bash
docker run -d -p 80:8080 saurabh/java-app-ms
```

Verify that the container is running:

```bash
docker ps
```

### Step 6: Test the Application

Open your browser and navigate to:

```
http://localhost
```

or use curl:

```bash
curl http://localhost
```

If the application responds, the deployment is successful.

### Completion

You have successfully:

1. Created a multi-stage Dockerfile for a Java Spring Boot application.
2. Built a lightweight and secure Docker image.
3. Deployed the application as a running Docker container.

# Lab 3 - Managing Containers and Images

In this hands-on lab, you will learn and practice essential Docker commands used to manage containers and images.\
These commands are foundational for all DevOps engineers, SREs, and Cloud engineers.

### **Pre-Requisite**

* A VM or system with **Docker installed**

## **Steps to Follow**

## **Step 1: Create a Test Container**

Run this command:

```bash
docker run -d -p 80:80 --name mycontainer nginx:latest
```

#### What this does:

* Downloads the **nginx:latest** image (if not already available)
* Creates and runs a container named **mycontainer**
* `-d` runs the container in detached mode (background)
* Maps **host port 80 → container port 80**

#### Expected Output:

A long container ID, meaning the container is successfully created and running.

## **Step 2: List Running Containers**

```bash
docker ps
```

#### What you will see:

A table showing:

* CONTAINER ID
* IMAGE
* COMMAND
* CREATED
* STATUS
* PORTS
* NAMES

You should see **mycontainer** in the list.

If nothing is running, the output will be empty.

## **Step 3: Stop a Running Container**

```bash
docker stop mycontainer
```

#### What happens:

* The container stops immediately.
* Run `docker ps` again : you will NOT see the container in the running list.

## **Step 4: List All Containers (Running + Stopped)**

```bash
docker ps -a
```

#### Purpose:

This displays _every_ container, including those in **Exited** state.

You should now see **mycontainer** listed as **Exited**.

## **Step 5: Inspect a Container**

```bash
docker inspect mycontainer
```

#### What this shows:

* JSON output describing the container in detail:
  * Config
  * Environment variables
  * Network settings
  * Mounts
  * Command executed
  * IP address
  * and more…

This is useful for debugging or retrieving metadata.

## **Step 6: Start a Stopped Container**

```bash
docker start mycontainer
```

#### What happens:

* The container restarts with the previous configuration.
* `docker ps` will show it running again.

## **Step 7: Restart a Running Container**

```bash
docker restart mycontainer
```

#### ✔ What this does:

* Stops the container
* Starts it again immediately\
  Useful for applying small changes or clearing temporary issues.

## **Step 8: Remove a Container**

A container must be stopped before removing.

```bash
docker stop mycontainer
docker rm mycontainer
```

#### What happens:

* Container will be permanently deleted.

## **Step 9: List All Docker Images**

```bash
docker images
```

#### Output shows:

* Repository name
* Tag (version)
* Image ID
* Created time
* Image size

## **Step 10: Inspect an Image**

Example:

```bash
docker inspect nginx:latest
```

#### What this shows:

Detailed JSON describing the image:

* Default command (Cmd)
* Entrypoint
* Environment variables
* Filesystem layers
* Image architecture

Useful for understanding how an image is built.

## **Step 11: Tag an Image**

```bash
docker tag nginx:latest mynginx:v1
```

#### What this does:

* Creates a new tag **mynginx:v1** for the same image.
* No duplicate storage — only a new pointer to the same Image ID.

## **Step 12: Verify Image Tags**

```bash
docker images
```

You should now see:

* `nginx:latest`
* `mynginx:v1`

Both with the **same Image ID**.

## **Step 13: Remove an Image Tag (or Image)**

```bash
docker rmi mynginx:v1
```

#### What happens:

* Tag is removed
* Image continues to exist because **nginx:latest** still points to it

## **Step 14: Clean Up Dangling Images**

Dangling images look like `<none>:<none>`.

```bash
docker image prune
```

You will be prompted:

```
This will remove all dangling images. Are you sure? [y/N]
```

Press:

```
y
```

## **Step 15: Deep Clean Docker (Unused Containers, Images, Networks)**

```bash
docker system prune -a
```

#### Warning:

This removes:

* All stopped containers
* All unused images
* All unused networks
* All dangling layers

Use this to free space.

Congratulations, you have learnt the most used commands in Docker

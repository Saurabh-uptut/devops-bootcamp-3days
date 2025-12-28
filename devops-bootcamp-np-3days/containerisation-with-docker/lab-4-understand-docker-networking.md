# Lab 4 - Understand Docker Networking

In this lab, you will learn how Docker networking works by using the default bridge network and the host network. You will inspect Docker networks, run containers in different network modes, test connectivity, and observe how network isolation works.

### Step 1: List Docker’s Default Networks

List all Docker networks:

```bash
docker network ls
```

Docker automatically creates several networks during installation. Two important ones are:

* bridge: The default network for containers. Containers receive private IP addresses inside an internal subnet, and traffic goes out through the docker0 interface using NAT.
* host: Removes Docker’s network isolation and allows the container to share the host’s network stack.

Inspect the default bridge network:

```bash
docker network inspect bridge
```

Look for the subnet and gateway. On most systems, Docker uses:

* Subnet: 172.17.0.0/16
* Gateway: 172.17.0.1

You will also see that no containers are attached yet. This confirms that the bridge network is a private internal network that uses NAT.

### Step 2: Use the Bridge Network (Default Docker Network)

The bridge driver is used automatically when you run a container without specifying a network.

#### Start a container on the bridge network

```bash
docker run -d --name webserver nginx
```

This runs an NGINX web server attached to the default bridge network. No ports are published to the host.

#### Try accessing NGINX through localhost

```bash
curl http://localhost
```

This request fails because the container’s port 80 is only accessible inside the bridge network. It is not mapped to the host.

#### Find the container’s internal IP address

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' webserver
```

You should see an IP address in the 172.17.x.x range. This is the private address assigned by Docker. The host can access this IP, but the container port will not be reachable externally unless it is published.

#### Expose the container to the host network

Remove the previous container:

```bash
docker rm -f webserver
```

Start NGINX again, this time with a published port:

```bash
docker run -d --name webserver2 -p 8080:80 nginx
```

Port 8080 on the host now forwards traffic to port 80 in the container.

Verify with:

```bash
docker ps
```

You should see a PORTS column showing:

```
0.0.0.0:8080->80/tcp
```

#### Test access through the published port

```bash
curl http://localhost:8080
```

You should receive the NGINX welcome page. This confirms that publishing a port on the bridge network makes the service available through the host.

### Step 3: Use the Host Network

In host network mode, Docker does not assign a private IP or separate network namespace. The container shares the host’s network directly.

#### Run NGINX using the host network

```bash
docker run -d --name hostweb --network host nginx
```

The container now binds directly to the host’s network interfaces.

#### Verify the container is running

```bash
docker ps
```

In host mode, the PORTS column will be empty or show the container’s port without mappings. This is expected because Docker does not create NAT rules in host mode.

#### Test the NGINX service

```bash
curl http://localhost
```

This succeeds because the NGINX server inside the container is bound to port 80 on the host’s network.

#### Inspect network settings

```bash
docker inspect -f '{{.NetworkSettings.IPAddress}}' hostweb
```

This will return an empty result. In host mode, the container does not receive a Docker-assigned IP address. It uses the host’s network namespace directly.

### Cleanup

Stop and remove all containers created during the lab:

```bash
docker container stop webserver2 hostweb
docker container rm webserver2 hostweb
```


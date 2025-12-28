# Lab6: Deploy a Two-Tier Node.js and PostgreSQL Application with Docker

This lab teaches you how to deploy a Node.js backend and PostgreSQL database as separate Docker containers, connect them using a custom Docker network, and persist database data using Docker volumes.&#x20;

By the end of the lab, you will have a fully working two-tier application running entirely inside Docker.

### Learning Objectives

1. Understand how Docker volumes allow database persistence beyond the lifecycle of a container.
2. Learn how to create and use a user-defined Docker network so containers can communicate by name.
3. Use Docker CLI commands such as docker run, docker volume, docker network, and docker build.
4. Configure containers using environment variables for database credentials and connection details.
5. Verify that the Node.js backend communicates correctly with PostgreSQL and can store and retrieve data.

### Step 1: Prerequisites and Setup

1. Ensure Docker is installed and running on your machine.
2. Clone the backend application repository from here - [Backend Application](https://github.com/saurabhd2106/sample-backend-app-ih)
3. Navigate into the project folder.

### Step 2: Create a Docker Volume for PostgreSQL Data

A Docker volume stores data outside of the container filesystem. This ensures data persists even if the container is removed.

Create the volume:

```bash
docker volume create postgres-data
```

Check that it exists:

```bash
docker volume ls
```

You should see postgres-data in the list.

### Step 3: Create a User-Defined Docker Network

Containers on the default bridge network do not support automatic DNS resolution by container name. A user-defined network allows containers to communicate using names such as postgres-db.

Create the network:

```bash
docker network create my-app-network
```

Verify the network:

```bash
docker network ls
```

### Step 4: Run the PostgreSQL Database Container

Start PostgreSQL using a named container, the user-defined network, and the volume created earlier.

```bash
docker run -d \
  --name postgres-db \
  --network my-app-network \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_USER=myappuser \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=myappdb \
  postgres:latest
```

Explanation of key flags:

* \--name postgres-db: Names the container so other containers can reference it.
* \--network my-app-network: Ensures the app container can reach this database container.
* -v postgres-data:/var/lib/postgresql/data: Persists database data to the named volume.
* POSTGRES\_USER, POSTGRES\_PASSWORD, POSTGRES\_DB: Credentials and database initialization settings.

Verify that PostgreSQL is running:

```bash
docker ps
```

### Step 5: Build and Run the Node.js Backend Application Container

Move inside the backend application directory and create a Dockerfile with the following content:

```
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3001

CMD ["npm", "run", "dev"]
```

Build the Node.js application image:

```bash
docker build -t node-backend-app .
```

Now run the container:

```bash
docker run -d \
  --name node-backend-app \
  --network my-app-network \
  -p 3001:3001 \
  -e DATABASE_URL=postgresql://myappuser:mysecretpassword@postgres-db:5432/myappdb \
  node-backend-app
```

Explanation of key flags:

* \--network my-app-network: Ensures the backend container can reach the PostgreSQL container.
* -p 3001:3001: Exposes the application on your host at [http://localhost:3001](http://localhost:3001/).
* DATABASE\_URL: Connection string that tells the Node.js application how to reach PostgreSQL. The hostname postgres-db works because both containers are on the same user-defined network.

### Step 6: Test the Two-Tier Application

#### A. Add a note using the Node.js API

```bash
curl -X POST http://localhost:3001/notes \
  -H "Content-Type: application/json" \
  -d '{"title": "My Note", "content": "This is the content"}'
```

#### B. Retrieve all notes

```bash
curl http://localhost:3001/notes
```

Or open this in your browser:

```
http://localhost:3001/notes
```

You should see the data stored in PostgreSQL.

### Step 7: Verify Data Persistence

The goal is to confirm that PostgreSQL data remains intact even if the database container is removed.

1. Remove the PostgreSQL container:

```bash
docker rm -f postgres-db
```

This deletes the container but not the postgres-data volume.

2. Recreate the PostgreSQL container using the same command from Step 4:

```bash
docker run -d \
  --name postgres-db \
  --network my-app-network \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_USER=myappuser \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=myappdb \
  postgres:latest
```

3. Test the API again:

```bash
curl http://localhost:3001/notes
```

The previously inserted notes should still be present. This confirms that the data was stored in the volume and not inside the container.


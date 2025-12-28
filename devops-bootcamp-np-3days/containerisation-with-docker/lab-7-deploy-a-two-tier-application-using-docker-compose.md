# Lab 7: Deploy a Two-Tier Application Using Docker Compose

In this lab, you will deploy a simple two-tier application using Docker Compose. The application consists of a Node.js backend and a PostgreSQL database. You will run each component in its own container, configure them to communicate over a custom Docker network, and persist database data using Docker volumes.

By the end of this lab, you will have a working Node.js application connected to a PostgreSQL database, both running inside Docker containers, with data preserved across restarts.

### Learning Objectives

1. Understand how to manage multi-container applications using Docker Compose.
2. Learn how to define services, volumes, networks, and environment variables in a docker-compose.yml file.
3. Build and run a Node.js application container and a PostgreSQL container using a single Compose configuration.
4. Persist database data using Docker volumes.
5. Verify that the backend successfully connects to the database.

### Pre-Requisites

* Docker installed and running on your system
* Docker Compose installed (included with Docker Desktop)
* Clone the backend application repository from here - [Backend Application](https://github.com/saurabhd2106/sample-backend-app-ih)

### Step 1: Create a Dockerfile for the Backend Application

Inside the backend application directory, create a file named Dockerfile with the following content:

```
FROM node:latest

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3001

CMD ["npm", "run", "dev"]
```

This Dockerfile builds an image for the Node.js backend service.

***

### Step 2: Create a docker-compose.yml File

In the same directory, create a file named docker-compose.yml and add the following content:

```
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: notes_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

  backend:
    build: ./
    ports:
      - "3001:3001"
    environment:
      PORT: 3001
      DATABASE_URL: postgresql://postgres:password@postgres:5432/notes_db
      NODE_ENV: development
    depends_on:
      - postgres
    volumes:
      - ./:/app
      - /app/node_modules
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

### Step 3: Understand the docker-compose.yml Structure

#### 1. PostgreSQL Service (postgres)

* **image: postgres:15**\
  Uses the official PostgreSQL version 15 image.
* **environment variables**
  * POSTGRES\_DB: notes\_db
  * POSTGRES\_USER: postgres
  * POSTGRES\_PASSWORD: password\
    These values initialize the database at startup.
* **ports**
  * "5432:5432"\
    Makes PostgreSQL available on your host machine.
* **volume**
  * postgres\_data:/var/lib/postgresql/data\
    Stores database files persistently.
* **network**\
  Connected to app-network so other services can reach it by the hostname postgres.

#### 2. Backend Service (backend)

* **build: ./**\
  Builds the Node.js image using the Dockerfile created earlier.
* **ports**
  * "3001:3001"\
    Exposes the backend API on [http://localhost:3001](http://localhost:3001/).
* **environment variables**
  * PORT: 3001
  * DATABASE\_URL: postgresql://postgres:password@postgres:5432/notes\_db\
    Connection string that points the backend to the postgres service.
* **depends\_on**\
  Ensures PostgreSQL container starts before the backend.
* **volumes**
  * ./:/app (mounts source code for live updates)
  * /app/node\_modules (prevents overwriting installed modules)
* **network**\
  Connected to app-network, allowing communication with postgres.

#### 3. Volumes

```
volumes:
  postgres_data:
```

Stores database data persistently so that removing the container does not remove the stored data.

#### 4. Networks

```
networks:
  app-network:
    driver: bridge
```

Creates a private bridge network for internal communication between postgres and backend.

### Step 4: Run the Application Using Docker Compose

Execute the command:

```bash
docker compose up -d
```

Docker Compose will:

1. Build the backend image
2. Create both service containers
3. Create the app-network
4. Create the postgres\_data volume
5. Connect both containers to the network

Verify the containers are running:

```bash
docker compose ps
```

### Step 5: Test the Backend Application

Open your browser or use curl.

#### Add a note:

```bash
curl -X POST http://localhost:3001/notes \
  -H "Content-Type: application/json" \
  -d '{"title": "Test Note", "content": "Sample content"}'
```

#### Retrieve notes:

```bash
curl http://localhost:3001/notes
```

If the application responds with stored data, the backend is communicating with PostgreSQL correctly.

### Step 6: Verify Data Persistence

1. Stop and remove all containers:

```bash
docker compose down
```

This stops containers but does not remove volumes.

2. Bring everything back up:

```bash
docker compose up -d
```

3. Query notes again:

```bash
curl http://localhost:3001/notes
```

Your previously inserted data should still be present. This confirms that the postgres\_data volume is persisting your database files


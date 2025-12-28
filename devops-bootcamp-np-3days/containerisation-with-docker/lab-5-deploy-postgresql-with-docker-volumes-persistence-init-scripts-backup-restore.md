# Lab 5 - Deploy PostgreSQL With Docker Volumes (Persistence, Init Scripts, Backup/Restore)

In this lab, you will deploy PostgreSQL using Docker, attach persistent storage through volumes, verify that data survives container deletion, seed a database automatically, and perform backup/restore operations.

## **Learning Objectives**

By the end of this lab, you will be able to:

* Understand the difference between **Docker named volumes** and **bind mounts**
* Run PostgreSQL using a **named volume** for persistent data
* Create and query tables using **psql**
* Demonstrate persistence by deleting and recreating the container
* Seed a database using **init scripts**
* Take a **logical backup** and restore it into a new container
* Use an **environment file** to store PostgreSQL configuration

## **Prerequisites**

* Docker installed and running (Linux, macOS, or Windows)
* A working terminal (bash, zsh, PowerShell)
* Port **5432** must be free on your host system

## **Step-by-Step Instructions**

## **Step 1: Pull the PostgreSQL Image (Pin a Version)**

```bash
docker pull postgres:16
```

Pinning versions ensures reliable, reproducible environments.

## **Step 2: Create a Named Volume for Persistent Data**

```bash
docker volume create pgdata
docker volume ls
```

**Named volumes** are managed by Docker and ideal for production-like persistence.

## **Step 3: Start PostgreSQL Using the Named Volume**

Run PostgreSQL using Docker:

```bash
docker run -d --name pg-db \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=StrongP@ssw0rd \
  -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  --health-cmd="pg_isready -U appuser -d appdb" \
  --health-interval=10s --health-timeout=5s --health-retries=5 \
  postgres:16
```

#### What this does:

* Creates user `appuser`, database `appdb`
* Mounts the named volume **pgdata**
* Exposes PostgreSQL on host port **5432**
* Adds a **healthcheck** that waits for PostgreSQL readiness

#### Check health status:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

```bash
docker inspect --format='{{json .State.Health.Status}}' pg-db
```

Wait until the container is reported as **"healthy"**.

## **Step 4: Connect to PostgreSQL and Insert Data**

Connect using `psql` inside the container:

```bash
docker exec -it pg-db psql -U appuser -d appdb
```

At the `appdb=#` prompt, run:

```sql
CREATE TABLE notes(id serial PRIMARY KEY, msg text NOT NULL);

INSERT INTO notes(msg)
VALUES ('hello, docker volumes'),
       ('persists across restarts');

SELECT * FROM notes;
```

Exit psql:

```
\q
```

## **Step 5: Prove Persistence : Delete and Recreate the Container**

#### Stop and delete the container (volume is preserved):

```bash
docker rm -f pg-db
```

#### Start a fresh PostgreSQL container using the SAME volume:

```bash
docker run -d --name pg-db \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=StrongP@ssw0rd \
  -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

#### Query the data again:

```bash
docker exec -it pg-db psql -U appuser -d appdb -c "SELECT * FROM notes;"
```

You should see the same rows : **persistence confirmed!**

## **Step 6: Inspect Where Docker Stores the Volume**

```bash
docker volume inspect pgdata
```

Look at the `"Mountpoint"` field: this is where Docker stores the data on the host.

## **Optional: Use a Bind Mount Instead of a Named Volume**

Bind mounts allow you to see and manipulate the files directly on your host.

```bash
mkdir -p ./pgdata
```

On Linux, if permissions are an issue:

```bash
sudo chown -R 999:999 ./pgdata
```

(`postgres` runs as UID **999** inside the container.)

#### Start PostgreSQL with a bind mount:

```bash
docker rm -f pg-db

docker run -d --name pg-db \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=StrongP@ssw0rd \
  -e POSTGRES_DB=appdb \
  -v "$(pwd)/pgdata:/var/lib/postgresql/data" \
  -p 5432:5432 \
  postgres:16
```

***

## **Section B: Seed the Database Automatically Using Init Scripts**

Files placed in `/docker-entrypoint-initdb.d` run **once** when the data directory is empty.

#### Create init script:

```bash
mkdir -p initdb

cat > initdb/001_init.sql <<'SQL'
CREATE TABLE todos(id serial PRIMARY KEY, task text NOT NULL);
INSERT INTO todos(task) VALUES ('learn docker volumes'), ('write a great lab');
SQL
```

#### Reset and reinitialize volume:

```bash
docker rm -f pg-db
docker volume rm pgdata
docker volume create pgdata
```

#### Run PostgreSQL with init scripts:

```bash
docker run -d --name pg-db \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=StrongP@ssw0rd \
  -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data \
  -v "$(pwd)/initdb:/docker-entrypoint-initdb.d:ro" \
  -p 5432:5432 \
  postgres:16
```

#### Verify automatic seeding:

```bash
docker exec -it pg-db psql -U appuser -d appdb -c "SELECT * FROM todos;"
```

## **Section C: Quick Backup and Restore**

#### Create a backup folder:

```bash
mkdir -p backups
```

#### Take a logical backup:

```bash
docker exec pg-db pg_dump -U appuser appdb > backups/appdb_$(date +%F).sql
```

#### Restore into a new PostgreSQL container:

```bash
docker run -d --name pg-restore \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=StrongP@ssw0rd \
  -e POSTGRES_DB=restored \
  -v pgdata_restore:/var/lib/postgresql/data \
  -p 5433:5432 \
  postgres:16
```

#### Load the backup:

```bash
cat backups/appdb_*.sql | docker exec -i pg-restore psql -U appuser -d restored
```

#### Verify tables:

```bash
docker exec -it pg-restore psql -U appuser -d restored -c "\dt"
```

## **Section D: Use an Environment File for Cleaner Config**

Create a `.env` file:

```
POSTGRES_USER=appuser
POSTGRES_PASSWORD=StrongP@ssw0rd
POSTGRES_DB=appdb
```

Run PostgreSQL using the env file:

```bash
docker rm -f pg-db

docker run -d --name pg-db --env-file ./.env \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

> In real production, use a **secrets manager** and **TLS** secure connections.

## **Troubleshooting Guide**

| Issue                                       | Resolution                                                     |
| ------------------------------------------- | -------------------------------------------------------------- |
| **Port 5432 already in use**                | Change to `-p 5433:5432` or stop other services                |
| **Permission denied on bind mount (Linux)** | `sudo chown -R 999:999 ./pgdata`                               |
| _POSTGRES\_ variables not applied_\*        | The DB initializes **only on first run** → recreate the volume |
| **Healthcheck stuck at "starting"**         | Run `docker logs pg-db` : usually wrong password or busy port  |

## **Cleanup Resources**

```bash
docker rm -f pg-db pg-restore 2>/dev/null || true
docker volume rm pgdata pgdata_restore 2>/dev/null || true
rm -rf ./pgdata ./initdb ./backups 2>/dev/null || true
```


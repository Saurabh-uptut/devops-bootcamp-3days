# Lab: Deploy a Three Tier Application on a Remote Server using Ansible Roles

## What You Will Build

In this lab, you will deploy a **three tier application** on a remote Linux VM using **Ansible roles** and **Docker**.

The application consists of:

* **Frontend (UI)**: Static HTML and CSS served via Nginx on port 80
* **Backend (API)**: Node.js application running on port 3000
* **Database (DB)**: PostgreSQL running as a Docker container
* **Networking**: All containers connected via a custom Docker bridge network

---

## Why Use Ansible Roles

Roles provide a **clean, reusable structure** for automation by separating logic into:

* Tasks
* Handlers
* Variables
* Templates (optional)

With roles, you avoid copying large YAML files and can reuse components across projects.

---

## Learning Objectives

By the end of this lab, you will be able to:

* Design and use Ansible roles to deploy multi container applications
* Build and run Docker images and containers using Ansible
* Orchestrate deployment order using a single playbook
* Safely clean up containers, images, networks, and temporary files

---

## Prerequisites

* Control node with Ansible installed
* Managed Linux VM reachable via SSH using key based authentication
* Docker installed on the managed node
* SSH user added to the docker group
* Firewall or security group allows:

  * TCP 80 for frontend
  * TCP 3000 for backend

---

## Application Overview

| Layer | Technology | Port            |
| ----- | ---------- | --------------- |
| UI    | Nginx      | 80              |
| API   | Node.js    | 3000            |
| DB    | PostgreSQL | 5432 (internal) |

All containers communicate over a custom Docker network.

---

## Step 0: Get the Application Code

Clone the repository on the **control node**.

```bash
git clone https://github.com/saurabhd2106/usermanagement-lab-ih.git
cd usermanagement-lab-ih
```

### Repository Structure

```text
usermanagement-lab-ih/
├── api/        # Node.js backend (uses .env)
└── ui/         # Frontend (uses config.json)
```

Important files:

* `api/.env` contains database connection details
* `ui/config.json` contains backend API URL

---

## Step 1: Add Dockerfiles

### UI Dockerfile

Create `ui/Dockerfile`.

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

### API Dockerfile

Create `api/Dockerfile`.

```dockerfile
FROM node:latest
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

---

## Step 2: Ansible Inventory and Configuration

### Create Ansible working directory

```bash
mkdir -p ansible_roles
cd ansible_roles
```

---

### Create inventory.ini

Replace placeholders with your VM details.

```ini
[webservers]
managed_vm ansible_host=<MANAGED_VM_PUBLIC_IP> ansible_user=<SSH_USER> ansible_ssh_private_key_file=./key.pem ansible_python_interpreter=/usr/bin/python3
```

---

### Create ansible.cfg

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
```

---

### Verify connectivity

```bash
ansible webservers -m ping
```

Expected result is a successful `pong`.

---

## Step 3: Create Role Skeleton

Create required role directories.

```bash
mkdir -p roles/application/tasks roles/application/handlers \
         roles/docker_network/tasks \
         roles/postgres_db/tasks roles/postgres_db/handlers \
         roles/cleanup/tasks
```

Create `main.yml` files.

```bash
touch roles/application/tasks/main.yml \
      roles/application/handlers/main.yml \
      roles/docker_network/tasks/main.yml \
      roles/postgres_db/tasks/main.yml \
      roles/postgres_db/handlers/main.yml \
      roles/cleanup/tasks/main.yml
```

---

## Step 4: Docker Network Role

File: `roles/docker_network/tasks/main.yml`

```yaml
- name: Create a custom Docker network
  community.docker.docker_network:
    name: "{{ docker_network_name }}"
    driver: "{{ docker_network_driver }}"
    state: present
```

Purpose: Ensures a shared Docker network exists for container communication.

---

## Step 5: Application Role (UI and API)

### Tasks

File: `roles/application/tasks/main.yml`

```yaml
- name: Create application directory
  ansible.builtin.file:
    path: "{{ app_dest_dir }}"
    state: directory
    mode: "0755"

- name: Copy application source
  ansible.builtin.copy:
    src: "{{ app_src_dir }}/"
    dest: "{{ app_dest_dir }}/"

- name: Build Docker image
  community.docker.docker_image:
    name: "{{ app_image }}"
    build:
      path: "{{ app_dest_dir }}"
    source: build
  notify: restart app container

- name: Run application container
  community.docker.docker_container:
    name: "{{ app_container }}"
    image: "{{ app_image }}"
    state: started
    restart_policy: always
    networks:
      - name: "{{ docker_network_name }}"
    ports:
      - "{{ exposed_port }}:{{ container_port }}"
```

---

### Handler

File: `roles/application/handlers/main.yml`

```yaml
- name: restart app container
  community.docker.docker_container:
    name: "{{ app_container }}"
    state: started
```

---

## Step 6: PostgreSQL Role

### Tasks

File: `roles/postgres_db/tasks/main.yml`

```yaml
- name: Pull PostgreSQL image
  community.docker.docker_image:
    name: "{{ postgres_image }}"
    source: pull
  notify: restart postgres container

- name: Run PostgreSQL container
  community.docker.docker_container:
    name: "{{ postgres_container_name }}"
    image: "{{ postgres_image }}"
    state: started
    restart_policy: always
    networks:
      - name: "{{ docker_network_name }}"
    ports:
      - "{{ host_port }}:{{ container_port }}"
    env:
      POSTGRES_USER: "{{ postgres_user }}"
      POSTGRES_PASSWORD: "{{ postgres_password }}"
      POSTGRES_DB: "{{ postgres_db }}"
```

---

### Handler

File: `roles/postgres_db/handlers/main.yml`

```yaml
- name: restart postgres container
  community.docker.docker_container:
    name: "{{ postgres_container_name }}"
    state: started
```

---

## Step 7: Cleanup Role

File: `roles/cleanup/tasks/main.yml`

```yaml
- name: Remove containers
  community.docker.docker_container:
    name: "{{ item }}"
    state: absent
  loop: "{{ docker_containers }}"
  ignore_errors: yes

- name: Remove images
  community.docker.docker_image:
    name: "{{ item }}"
    state: absent
  loop: "{{ docker_images }}"
  ignore_errors: yes

- name: Remove Docker network
  community.docker.docker_network:
    name: "{{ docker_network_name }}"
    state: absent
  ignore_errors: yes

- name: Remove temporary directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop: "{{ folders_to_delete }}"
```

---

## Step 8: Configure the Application

### Frontend configuration

Edit `ui/config.json`.

```json
{
  "API_URL": "http://<MANAGED_VM_PUBLIC_IP>:3000/"
}
```

---

### Backend configuration

Edit `api/.env`.

```env
DB_USER=postgres
DB_PASSWORD=postgres
DB_SERVER=postgres_db
DB_NAME=postgresdb
```

Note: `postgres_db` matches the container name and resolves via Docker internal DNS.

---

## Step 9: Create the Playbook

File: `ansible_roles/app.yml`

```yaml
---
- hosts: webservers
  become: true

  roles:
    - role: cleanup
      vars:
        docker_containers:
          - frontendapp
          - backendapp
          - postgres_db
        docker_images:
          - frontendapp
          - backendapp
          - postgres:latest
        folders_to_delete:
          - /tmp/frontendapp
          - /tmp/backendapp
        docker_network_name: custom_network

    - role: docker_network
      vars:
        docker_network_name: custom_network
        docker_network_driver: bridge

    - role: postgres_db
      vars:
        postgres_container_name: postgres_db
        postgres_image: postgres:latest
        postgres_user: postgres
        postgres_password: postgres
        postgres_db: postgresdb
        host_port: 5432
        container_port: 5432
        docker_network_name: custom_network

    - role: application
      vars:
        app_image: frontendapp
        app_container: frontendapp
        app_src_dir: ../ui
        app_dest_dir: /tmp/frontendapp
        exposed_port: 80
        container_port: 80
        docker_network_name: custom_network

    - role: application
      vars:
        app_image: backendapp
        app_container: backendapp
        app_src_dir: ../api
        app_dest_dir: /tmp/backendapp
        exposed_port: 3000
        container_port: 3000
        docker_network_name: custom_network
```

---

## Step 10: Run the Deployment

From the `ansible_roles` directory:

```bash
ansible-playbook app.yml
```

---

## Step 11: Verify Deployment

### UI

Open in browser:

```text
http://<MANAGED_VM_PUBLIC_IP>
```

---

### API

```bash
curl http://<MANAGED_VM_PUBLIC_IP>:3000
```

---

### Containers and Network

```bash
ssh <SSH_USER>@<MANAGED_VM_PUBLIC_IP>
docker ps
docker network ls
docker inspect backendapp | grep custom_network
```

---

## Success Criteria

* Frontend loads on port 80
* Frontend communicates with backend on port 3000
* Backend connects to PostgreSQL without errors
* All containers are running and attached to `custom_network`

---

## Troubleshooting

* UI errors: Verify `ui/config.json`, rebuild frontend
* API DB errors: Verify `DB_SERVER=postgres_db`
* Network issues: Confirm containers share the same Docker network
* Docker module missing:

  ```bash
  ansible-galaxy collection install community.docker
  ```
* Docker permission denied:

  ```bash
  sudo usermod -aG docker <SSH_USER> && exit
  ```

---

## Cleanup

Re run the playbook or clean manually:

```bash
docker rm -f frontendapp backendapp postgres_db || true
docker rmi -f frontendapp backendapp postgres:latest || true
docker network rm custom_network || true
sudo rm -rf /tmp/frontendapp /tmp/backendapp
```

# Lab: Deploy a Three Tier Application on a Remote Server Using Ansible Playbooks

## Lab Objective

In this lab, you will deploy a **three tier application** on a remote server using **Ansible Playbooks and Docker**.

You will use Ansible to:

* Build Docker images
* Create a Docker network
* Deploy a PostgreSQL database
* Deploy a backend API
* Deploy a frontend UI
* Orchestrate everything using a single master playbook

This lab demonstrates how Ansible can be used as a **full application deployment tool**, not just for configuration.

---

## Learning Objective

* Write and execute Ansible playbooks to deploy applications as Docker containers

---

## Prerequisites

* A control node with Ansible installed
* One managed node (AWS EC2 Ubuntu)
* SSH access from control node to managed node
* Docker installed on the managed node
* Ports **80** and **3000** open in the EC2 security group

---

## Step 1: Get the Application Code

Clone the application repository on the **control node**.

```bash
git clone https://github.com/saurabhd2106/usermanagement-lab-ih.git
cd usermanagement-lab-ih
```

This repository contains a sample **three tier user management application**.

### Application Structure

* **ui/**
  Frontend static application served via Nginx

* **api/**
  Backend Node.js REST API

* **Database**
  PostgreSQL container

### Important Configuration Files

* `api/.env`
  Database connection configuration

* `ui/config.json`
  Backend API URL configuration

---

## Step 2: Add Dockerfiles

### Frontend Dockerfile

Create `ui/Dockerfile`.

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

### Backend Dockerfile

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

## Step 3: Create the Ansible Project Structure

Inside the repository root, create an **ansible/** directory.

```bash
mkdir ansible
cd ansible
```

Create the following files:

```text
ansible/
├── inventory.ini
├── ansible.cfg
├── create-custom-network.yml
├── db-app.yml
├── backend-app.yml
├── frontend-app.yml
├── final-app.yml
├── cleanup.yml
```

---

## Step 4: Inventory and Configuration

### Inventory File: inventory.ini

Replace the IP with your **AWS EC2 public IP**.

```ini
[webservers]
managed_vm ansible_host=54.xxx.xxx.xxx ansible_user=ubuntu ansible_ssh_private_key_file=./key.pem ansible_python_interpreter=/usr/bin/python3
```

---

### Ansible Configuration: ansible.cfg

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
```

---

### Verify Connectivity

```bash
ansible webservers -m ping
```

You should see a successful `pong`.

---

## Step 5: Create a Custom Docker Network

File: `create-custom-network.yml`

```yaml
---
- name: Create a custom Docker network
  hosts: all
  become: true

  vars:
    docker_network_name: custom_network

  tasks:
    - name: Create Docker network
      community.docker.docker_network:
        name: "{{ docker_network_name }}"
        driver: bridge
        state: present
```

This allows all containers to communicate internally using container names.

---

## Step 6: Deploy the Database

File: `db-app.yml`

```yaml
---
- name: Deploy PostgreSQL database
  hosts: all
  become: true

  vars:
    postgres_container_name: postgres_db
    postgres_image: postgres:latest
    postgres_user: postgres
    postgres_password: postgres
    postgres_db: postgresdb
    docker_network_name: custom_network

  tasks:
    - name: Pull PostgreSQL image
      community.docker.docker_image:
        name: "{{ postgres_image }}"
        source: pull

    - name: Run PostgreSQL container
      community.docker.docker_container:
        name: "{{ postgres_container_name }}"
        image: "{{ postgres_image }}"
        state: started
        restart_policy: always
        networks:
          - name: "{{ docker_network_name }}"
        ports:
          - "5432:5432"
        env:
          POSTGRES_USER: "{{ postgres_user }}"
          POSTGRES_PASSWORD: "{{ postgres_password }}"
          POSTGRES_DB: "{{ postgres_db }}"
```

---

### Update Backend Environment File

Edit `api/.env`:

```env
DB_USER=postgres
DB_PASSWORD=postgres
DB_SERVER=postgres_db
DB_NAME=postgresdb
```

Note: `postgres_db` is the container name and acts as the hostname inside Docker network.

---

## Step 7: Deploy the Backend API

File: `backend-app.yml`

```yaml
---
- name: Build and run backend application
  hosts: all
  become: true

  vars:
    app_image: backendapp
    app_container: backendapp
    app_src_dir: ../api
    app_dest_dir: /tmp/backendapp
    docker_network_name: custom_network

  tasks:
    - name: Create backend directory
      ansible.builtin.file:
        path: "{{ app_dest_dir }}"
        state: directory
        mode: "0755"

    - name: Copy backend source code
      ansible.builtin.copy:
        src: "{{ app_src_dir }}/"
        dest: "{{ app_dest_dir }}/"

    - name: Build backend Docker image
      community.docker.docker_image:
        name: "{{ app_image }}"
        build:
          path: "{{ app_dest_dir }}"
        source: build

    - name: Run backend container
      community.docker.docker_container:
        name: "{{ app_container }}"
        image: "{{ app_image }}"
        state: started
        restart_policy: always
        networks:
          - name: "{{ docker_network_name }}"
        ports:
          - "3000:3000"
```

---

### Update Frontend Configuration

Edit `ui/config.json`:

```json
{
  "API_URL": "http://<EC2_PUBLIC_IP>:3000/"
}
```

---

## Step 8: Deploy the Frontend UI

File: `frontend-app.yml`

```yaml
---
- name: Build and run frontend application
  hosts: all
  become: true

  vars:
    app_image: frontendapp
    app_container: frontendapp
    app_src_dir: ../ui
    app_dest_dir: /tmp/frontendapp

  tasks:
    - name: Create frontend directory
      ansible.builtin.file:
        path: "{{ app_dest_dir }}"
        state: directory
        mode: "0755"

    - name: Copy frontend source code
      ansible.builtin.copy:
        src: "{{ app_src_dir }}/"
        dest: "{{ app_dest_dir }}/"

    - name: Build frontend Docker image
      community.docker.docker_image:
        name: "{{ app_image }}"
        build:
          path: "{{ app_dest_dir }}"
        source: build

    - name: Run frontend container
      community.docker.docker_container:
        name: "{{ app_container }}"
        image: "{{ app_image }}"
        state: started
        restart_policy: always
        ports:
          - "80:80"
```

---

## Step 9: Cleanup Playbook

File: `cleanup.yml`

```yaml
---
- name: Cleanup containers, images, and folders
  hosts: all
  become: true

  vars:
    docker_containers:
      - frontendapp
      - backendapp
      - postgres_db
    docker_images:
      - frontendapp
      - backendapp
      - postgres:latest
    folders:
      - /tmp/backendapp
      - /tmp/frontendapp

  tasks:
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

    - name: Remove folders
      ansible.builtin.file:
        path: "{{ item }}"
        state: absent
      loop: "{{ folders }}"
```

---

## Step 10: Combine All Playbooks

File: `final-app.yml`

```yaml
---
- import_playbook: cleanup.yml
- import_playbook: create-custom-network.yml
- import_playbook: db-app.yml
- import_playbook: backend-app.yml
- import_playbook: frontend-app.yml
```

---

## Step 11: Run the Full Deployment

```bash
ansible-playbook final-app.yml
```

---

## Step 12: Verify the Application

Open the frontend in your browser:

```text
http://<EC2_PUBLIC_IP>
```

You should see:

* Frontend UI running on port 80
* Backend API responding on port 3000
* Data persisted in PostgreSQL

---

## Optional Cleanup

To remove everything:

```bash
ansible-playbook cleanup.yml
```
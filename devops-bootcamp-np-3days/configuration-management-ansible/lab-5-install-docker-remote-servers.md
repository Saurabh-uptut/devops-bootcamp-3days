# Lab: Create an Ansible Playbook to Install Docker on Remote Servers (AWS)

## Lab Objective

In this lab, you will create and run an **Ansible playbook** to install Docker on a remote AWS EC2 instance.

This lab demonstrates how Ansible playbooks automate **multi step installations** in a **consistent, repeatable, and idempotent** way, which is a key advantage over manual shell scripts.

---

## Learning Objectives

By the end of this lab, you will be able to:

* Write a basic Ansible playbook
* Automate Docker installation on AWS EC2 instances
* Run playbooks using Ansible inventory and configuration files
* Verify Docker installation on remote servers

---

## Prerequisites

* A control node with Ansible installed
* At least one AWS EC2 instance running Ubuntu
* SSH access from the control node to the EC2 instance
* An AWS key pair (.pem file)

Tip: Always SSH manually into each managed node at least once to populate the `known_hosts` file.

---

## Step 1: Create the Inventory File

### Step 1.1: Create a working directory

On the control node:

```bash
mkdir install_docker
cd install_docker
```

---

### Step 1.2: Copy and secure the AWS SSH key

Copy your EC2 key pair into this directory and secure it.

```bash
chmod 400 key.pem
```

---

### Step 1.3: Create an INI inventory file

Create a file named `inventory.ini`.

Replace the IP address with your EC2 public IP.

```ini
[webservers]
backend_vm ansible_host=54.182.xxx.xxx ansible_user=ubuntu ansible_ssh_private_key_file=./key.pem ansible_python_interpreter=/usr/bin/python3
```

Explanation:

* `ansible_user=ubuntu` is the default user for Ubuntu EC2 instances
* `ansible_python_interpreter` ensures Ansible uses Python 3

---

### Alternative: YAML inventory file

You may also use YAML format.

Create `inventory.yml`:

```yaml
all:
  children:
    webservers:
      hosts:
        backend_vm:
          ansible_host: 54.182.xxx.xxx
          ansible_user: ubuntu
          ansible_ssh_private_key_file: "./key.pem"
          ansible_python_interpreter: /usr/bin/python3
```

---

### Step 1.4: Verify the inventory

```bash
ansible-inventory -i inventory.ini --list
```

Or for YAML:

```bash
ansible-inventory -i inventory.yml --list
```

You should see structured output showing the `webservers` group.

---

## Step 2: Create an Ansible Configuration File

Instead of passing the inventory file on every command, configure Ansible to use it by default.

### Step 2.1: Create ansible.cfg

In the project directory, create `ansible.cfg`.

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
```

This:

* Sets the default inventory
* Avoids SSH host key prompts during labs

---

### Step 2.2: Verify configuration

```bash
ansible-inventory --list
```

---

### Step 2.3: Test SSH connectivity

```bash
ansible webservers -m ping
```

Expected output:

```text
backend_vm | SUCCESS => {"ping": "pong"}
```

---

## Step 3: Create the Playbook to Install Docker

### Step 3.1: Create the playbook file

Create a file named `install_docker.yml`.

---

### Step 3.2: Add the playbook content

```yaml
---
- name: Install Docker on AWS EC2 instances
  hosts: all
  become: true

  vars:
    docker_release: "{{ ansible_distribution_release | lower }}"

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install required packages
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present

    - name: Create keyrings directory
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: "0755"

    - name: Add Docker GPG key
      ansible.builtin.apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present
        keyring: /etc/apt/keyrings/docker.gpg

    - name: Add Docker repository
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ docker_release }} stable"
        state: present
        filename: docker

    - name: Update apt cache after adding Docker repo
      ansible.builtin.apt:
        update_cache: yes

    - name: Install Docker packages
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present

    - name: Enable and start Docker service
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    - name: Add user to docker group
      ansible.builtin.user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes
```

---

## Step 4: Understand What the Playbook Does

This playbook performs the following actions in order:

1. Updates the system package index
2. Installs required dependencies for Docker
3. Creates a secure keyrings directory
4. Adds Docker’s official GPG key
5. Adds Docker’s official APT repository
6. Installs Docker Engine, CLI, and containerd
7. Starts and enables Docker on boot
8. Adds the EC2 user to the Docker group for non sudo usage

---

## Step 5: Run the Playbook

Execute the playbook from the project directory.

```bash
ansible-playbook install_docker.yml
```

You should see tasks executing with `ok` and `changed` statuses.

---

## Step 6: Verify the Docker Installation

### Step 6.1: SSH into the EC2 instance

```bash
ssh -i key.pem ubuntu@<EC2_PUBLIC_IP>
```

---

### Step 6.2: Verify Docker version

```bash
docker --version
```

Expected output example:

```text
Docker version 26.x.x, build xxxxxxx
```

---

### Step 6.3: Test Docker without sudo

Log out and log back in, then run:

```bash
docker ps
```

This confirms Docker group membership is working.

---

## Lab Summary

In this lab, you:

* Created an Ansible inventory for AWS EC2 instances
* Configured Ansible using ansible.cfg
* Wrote an Ansible playbook to install Docker
* Executed the playbook on a remote server
* Verified Docker installation and configuration

---

## Next Lab

* Writing idempotent playbooks
* Installing Docker Compose using Ansible
* Deploying a containerized application with Ansible
* Converting ad hoc commands into playbooks

If you want, I can also:

* Convert this into a challenge lab
* Add Amazon Linux support
* Extend the playbook to install Docker Compose and Docker Buildx
* Add handlers and conditional logic for production readiness

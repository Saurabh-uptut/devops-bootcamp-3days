# Lab: Create Your First Ansible Playbook

## Lab Objective

In this lab, you will write and execute your **first Ansible playbook**.
The goal is to understand how a playbook is structured and how Ansible executes tasks on remote servers.

You will use the **ping module**, which is the simplest way to verify connectivity and Python availability on managed nodes.

---

## Learning Objectives

By the end of this lab, you will be able to:

* Understand the structure and syntax of an Ansible playbook
* Write a simple playbook using the ping module
* Execute a playbook on remote servers
* Read and interpret Ansible playbook output

---

## Prerequisites

* A control node with Ansible installed
* Two AWS EC2 instances running Ubuntu
* SSH access from the control node to both EC2 instances
* An AWS key pair (.pem file)

Note: Always SSH manually into each managed node at least once to populate the `known_hosts` file.

---

## Step 1: Create an Inventory File

### Step 1.1: Create a working directory

On the control node:

```bash
mkdir ansible_playbook
cd ansible_playbook
```

---

### Step 1.2: Copy and secure the SSH key

Copy your AWS EC2 private key into this directory and set proper permissions.

```bash
chmod 400 key.pem
```

---

### Step 1.3: Create an inventory file in INI format

Create a file named `inventory.ini`.

Replace the IP addresses with your EC2 public IPs.

```ini
[webservers]
backend_vm ansible_host=54.182.xxx.xxx ansible_user=ubuntu ansible_ssh_private_key_file=./key.pem ansible_python_interpreter=/usr/bin/python3
frontend_vm ansible_host=3.110.xxx.xxx ansible_user=ubuntu ansible_ssh_private_key_file=./key.pem ansible_python_interpreter=/usr/bin/python3
```

Explanation:

* `[webservers]` is a host group
* `backend_vm` and `frontend_vm` are host aliases
* `ansible_user=ubuntu` is the default user for Ubuntu EC2
* `ansible_python_interpreter` ensures Python 3 is used

---

### Alternative: Inventory in YAML format

Create `inventory.yml` if you prefer YAML.

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
        frontend_vm:
          ansible_host: 3.110.xxx.xxx
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

You should see both hosts listed under the `webservers` group.

---

## Step 2: Create Your First Playbook

### Step 2.1: Create the playbook file

Create a file named `ping.yml`.

---

### Step 2.2: Add the playbook content

```yaml
---
- name: My first Ansible play
  hosts: webservers

  tasks:
    - name: Ping all hosts
      ansible.builtin.ping:
```

Explanation:

* `name` describes the play and the task
* `hosts` specifies the target group from the inventory
* `tasks` defines the actions Ansible will run
* `ansible.builtin.ping` checks connectivity and Python availability

---

## Step 3: Check the Playbook Syntax

Before executing any playbook, always validate the syntax.

```bash
ansible-playbook ping.yml -i inventory.ini --syntax-check
```

Expected output:

```text
playbook: ping.yml
```

This confirms the YAML syntax is valid.

---

## Step 4: Run the Playbook

Execute the playbook against the managed nodes.

```bash
ansible-playbook ping.yml -i inventory.ini
```

---

## Step 5: Review the Output

While the playbook runs, observe the following:

* **Play name and task name** printed clearly
* **Gathering Facts** step, where Ansible collects system details
* **Task status**, such as:

  * `ok` for success
  * `changed` when changes are made
  * `failed` if something goes wrong

Expected output example:

```text
backend_vm | SUCCESS => {"ping": "pong"}
frontend_vm | SUCCESS => {"ping": "pong"}
```

### Play recap section

At the end, you will see a summary like:

```text
PLAY RECAP
backend_vm  : ok=1 changed=0 unreachable=0 failed=0
frontend_vm : ok=1 changed=0 unreachable=0 failed=0
```

This shows the task executed successfully on both hosts.

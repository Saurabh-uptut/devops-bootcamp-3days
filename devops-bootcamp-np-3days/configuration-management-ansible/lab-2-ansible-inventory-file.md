# Lab: Ansible Inventory File

## Lab Objective

In this lab, you will learn how to create and use **Ansible inventory files** to define and manage multiple target systems.

An inventory file tells Ansible:

* Which hosts to manage
* How to connect to them
* How to group them logically

With a single Ansible command, you can run tasks across many servers.

---

## Learning Objectives

By the end of this lab, you will be able to:

* Create Ansible inventory files
* Understand inventory syntax in INI and YAML formats
* Use host aliases for readability
* Configure SSH authentication using keys or passwords
* Verify connectivity using Ansible commands

---

## Prerequisites

* Ansible installed on a control node
* Basic familiarity with INI and YAML syntax
* At least two Linux VMs with SSH enabled
* Network access from the control node to the managed nodes

Tip: If you know Terraform, you can provision the VMs using Terraform.

---

## Task 1: Inventory File in INI Format

### Step 1: Verify SSH access to managed nodes

From your control node, manually SSH into each VM at least once.

```bash
ssh azureuser@<vm-ip-address>
```

This ensures:

* SSH connectivity works
* Host keys are added to `known_hosts`

Repeat for all managed nodes.

---

### Step 2: Create a project directory

```bash
mkdir ansible_project
cd ansible_project
```

---

### Step 3: Create an inventory file in INI format

Create a file named `inventory.ini`.

```ini
[webservers]
20.71.160.229
20.71.160.223
```

Note:

* Replace the IP addresses with your VM IPs
* `[webservers]` defines a host group
* Each line below the group is a managed host

---

### Step 4: Verify the inventory file

```bash
ansible-inventory -i inventory.ini --list
```

You should see structured JSON output showing the `webservers` group and its hosts.

---

## Task 2: Inventory File in YAML Format

### Step 1: Create a YAML inventory file

Create a file named `inventory.yml`.

```yaml
all:
  children:
    webservers:
      hosts:
        20.71.160.229:
        20.71.160.223:
```

Explanation:

* `all` is the top-level inventory group
* `children` defines subgroups
* `webservers` is a logical group
* `hosts` lists individual machines

---

### Step 2: Verify the YAML inventory

```bash
ansible-inventory -i inventory.yml --list
```

The output should match the structure of your inventory.

---

## Task 3: Using Host Aliases

IP addresses are hard to remember. Host aliases improve readability.

---

### INI inventory with aliases

Update `inventory.ini`:

```ini
[webservers]
backend_vm ansible_host=20.71.160.229
frontend_vm ansible_host=20.71.160.223
```

---

### YAML inventory with aliases

Update `inventory.yml`:

```yaml
all:
  children:
    webservers:
      hosts:
        backend_vm:
          ansible_host: 20.71.160.229
        frontend_vm:
          ansible_host: 20.71.160.223
```

---

### Verify aliases

```bash
ansible-inventory -i inventory.ini --list
```

You should now see logical hostnames instead of raw IPs.

---

## Task 4: SSH Authentication Using Key Pair

### Step 1: Copy your SSH private key to the project directory

If required, copy the key using SCP.

```bash
scp key.pem user@control-node:/path/to/ansible_project/
```

---

### Step 2: Secure the private key

```bash
chmod 400 key.pem
```

---

### Step 3: Update INI inventory with SSH key details

```ini
[webservers]
backend_vm ansible_host=20.71.160.229 ansible_user=azureuser ansible_ssh_private_key_file=./key.pem
frontend_vm ansible_host=20.71.160.223 ansible_user=azureuser ansible_ssh_private_key_file=./key.pem
```

---

### Step 4: Update YAML inventory with SSH key details

```yaml
all:
  children:
    webservers:
      hosts:
        backend_vm:
          ansible_host: 20.71.160.229
          ansible_user: azureuser
          ansible_ssh_private_key_file: "./key.pem"
        frontend_vm:
          ansible_host: 20.71.160.223
          ansible_user: azureuser
          ansible_ssh_private_key_file: "./key.pem"
```

---

### Step 5: Verify connectivity

```bash
ansible webservers -m ping -i inventory.ini
```

Expected output:

```text
backend_vm | SUCCESS => {"ping": "pong"}
frontend_vm | SUCCESS => {"ping": "pong"}
```

---

### Step 6: Fix Python interpreter warnings if needed

If you see Python related warnings, explicitly define the interpreter.

INI example:

```ini
backend_vm ansible_host=20.71.160.229 ansible_user=azureuser ansible_ssh_private_key_file=./key.pem ansible_python_interpreter=/usr/bin/python3.12
```

---

## Task 5: SSH Authentication Using Username and Password

Password based authentication is supported but not recommended for production.

---

### INI inventory with password authentication

```ini
[webservers]
backend_vm ansible_host=20.71.160.229 ansible_user=azureuser ansible_ssh_pass=yourpassword ansible_python_interpreter=/usr/bin/python3.12
frontend_vm ansible_host=20.71.160.223 ansible_user=azureuser ansible_ssh_pass=yourpassword ansible_python_interpreter=/usr/bin/python3.12
```

---

### YAML inventory with password authentication

```yaml
all:
  children:
    webservers:
      hosts:
        backend_vm:
          ansible_host: 20.71.160.229
          ansible_user: azureuser
          ansible_ssh_pass: yourpassword
        frontend_vm:
          ansible_host: 20.71.160.223
          ansible_user: azureuser
          ansible_ssh_pass: yourpassword
```

---

### Verify connectivity

```bash
ansible webservers -m ping -i inventory.yml
```

---

## Lab Summary

In this lab, you learned:

* How to define Ansible inventory files using INI and YAML
* How to group hosts logically
* How to use host aliases for readability
* How to configure SSH authentication using keys and passwords
* How to verify inventory structure and host connectivity

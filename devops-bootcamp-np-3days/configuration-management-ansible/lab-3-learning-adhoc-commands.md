# Lab: Learning Ad hoc Commands in Ansible

## Lab Objective

In this lab, you will learn how to use **Ansible ad hoc commands** to perform quick, one time tasks on remote servers.

Ad hoc commands are executed directly from the command line. They are not reusable like playbooks, but they help you understand:

* How Ansible works internally
* How modules behave
* How inventory, authentication, and privilege escalation work together

Everything you practice here directly applies when you later write playbooks.

---

## Learning Objectives

By the end of this lab, you will be able to:

* Understand what ad hoc commands are in Ansible
* Use common Ansible modules such as `ping`, `copy`, `file`, `apt`, and `service`
* Run ad hoc commands to:

  * Test connectivity
  * Transfer files
  * Change file permissions
  * Create directories
  * Install packages
  * Start and stop services

---

## Prerequisites

* Ansible installed on a control node
* At least two Linux managed nodes with SSH enabled
* Working SSH connectivity from the control node to the managed nodes
* Python 3 installed on the managed nodes

---

## Ad hoc Command Syntax

```bash
ansible [pattern] -m [module] -a "[module options]"
```

Where:

* `[pattern]` is a host or group name from the inventory
* `-m` specifies the Ansible module
* `-a` provides arguments to the module

---

## Step 1: Create the Inventory File

### Step 1.1: Connect to the control node

SSH into the machine where Ansible is installed.

---

### Step 1.2: Create a working directory

```bash
mkdir ansible_adhoc
cd ansible_adhoc
```

---

### Step 1.3: Prepare SSH key authentication

If you are using key based authentication, copy your private key into this directory and secure it.

```bash
chmod 400 key.pem
```

For Azure VMs, you can transfer the key using `scp`.

---

### Step 1.4: Create an INI inventory file

Create `inventory.ini` and add the following content.

Replace the IP addresses with your own VM IPs.

```ini
[webservers]
backend_vm ansible_host=20.71.160.229 ansible_user=azureuser ansible_ssh_private_key_file=./key.pem ansible_python_interpreter=/usr/bin/python3.12
frontend_vm ansible_host=20.71.160.223 ansible_user=azureuser ansible_ssh_private_key_file=./key.pem ansible_python_interpreter=/usr/bin/python3.12
```

---

### Alternative: YAML inventory file

You can also create `inventory.yml`.

```yaml
all:
  children:
    webservers:
      hosts:
        backend_vm:
          ansible_host: 20.71.160.229
          ansible_user: azureuser
          ansible_ssh_private_key_file: "./key.pem"
          ansible_python_interpreter: /usr/bin/python3.12
        frontend_vm:
          ansible_host: 20.71.160.223
          ansible_user: azureuser
          ansible_ssh_private_key_file: "./key.pem"
          ansible_python_interpreter: /usr/bin/python3.12
```

---

### Step 1.5: Verify the inventory

```bash
ansible-inventory -i inventory.ini --list
```

Or for YAML:

```bash
ansible-inventory -i inventory.yml --list
```

You should see structured output showing the `webservers` group and its hosts.

---

## Step 2: Test SSH Connectivity

Run a ping test to confirm Ansible can reach the managed nodes.

```bash
ansible webservers -m ping -i inventory.ini
```

Or:

```bash
ansible webservers -m ping -i inventory.yml
```

Expected output:

```text
backend_vm | SUCCESS => {"ping": "pong"}
frontend_vm | SUCCESS => {"ping": "pong"}
```

Note:

* The `ping` module does not use ICMP
* It verifies Python availability and SSH connectivity

---

## Step 3: Transferring Files

### Step 3.1: Create a sample file

```bash
echo "Sample file" > sample_file.txt
```

---

### Step 3.2: Copy the file to all webservers

```bash
ansible webservers -m ansible.builtin.copy -a "src=./sample_file.txt dest=/tmp/sample_file.txt" -i inventory.ini
```

---

### Step 3.3: Verify on a managed node

Manually SSH into one of the VMs and verify.

```bash
ls -lrt /tmp/sample_file.txt
cat /tmp/sample_file.txt
```

---

## Step 4: Changing File Permissions

Use the `file` module to modify permissions.

```bash
ansible webservers -m ansible.builtin.file -a "dest=/tmp/sample_file.txt mode=600" -i inventory.ini
```

Verify on the VM:

```bash
ls -l /tmp/sample_file.txt
```

---

## Step 5: Creating Directories

Create a directory with ownership and permissions.

```bash
ansible webservers -m ansible.builtin.file -a "dest=/tmp/new_folder state=directory mode=755 owner=azureuser group=azureuser" -i inventory.ini
```

Verify:

```bash
ls -ld /tmp/new_folder
```

---

## Step 6: Managing Packages

Use the `apt` module to install packages.

### Install Apache

```bash
ansible webservers -m ansible.builtin.apt -a "name=apache2 state=present" --become -i inventory.ini
```

Explanation:

* `state=present` ensures the package is installed
* `--become` runs the command with root privileges

---

## Step 7: Managing Services

### Start Apache

```bash
ansible webservers -m ansible.builtin.service -a "name=apache2 state=started" --become -i inventory.ini
```

---

### Verify Apache

Ensure port 80 is open in the VM firewall or security group.

Open in a browser:

```text
http://<backend_vm_ip>
http://<frontend_vm_ip>
```

You should see the Apache default page.

---

### Stop Apache

```bash
ansible webservers -m ansible.builtin.service -a "name=apache2 state=stopped" --become -i inventory.ini
```

---

## Lab Summary

In this lab, you learned how to:

* Define and verify an Ansible inventory
* Run ad hoc Ansible commands
* Test connectivity using the ping module
* Transfer files using the copy module
* Change permissions and create directories using the file module
* Install packages using apt
* Start and stop services using the service module

---

## Next Lab

* Using variables in Ansible
* Group variables and host variables
* Writing your first Ansible playbook

If you want, I can now:

* Convert this into a challenge lab without hints
* Add troubleshooting scenarios
* Create a comparison table of ad hoc commands vs playbooks
* Turn this into a GitHub ready README

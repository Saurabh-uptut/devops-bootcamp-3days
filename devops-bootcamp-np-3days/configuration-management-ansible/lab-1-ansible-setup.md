# Lab 1: Installing and Setting Up Ansible

## Lab Objective

In this lab, you will install and configure Ansible on different operating systems and set up your local environment for Ansible development.

Ansible is an **agentless automation tool** that runs from a single machine called the **control node**. This control node manages other machines remotely using SSH.

---

## Prerequisites

* Internet connectivity
* Terminal access
* Administrator or sudo privileges
* VS Code installed for editor setup

---

## Task 1: Install Ansible on macOS

### Step 1: Install Homebrew (if not already installed)

Verify Homebrew:

```bash
brew --version
```

If not installed, install Homebrew from [https://brew.sh](https://brew.sh)

### Step 2: Install Ansible using Homebrew

```bash
brew install ansible
```

### Step 3: Verify Ansible installation

```bash
ansible --version
```

Expected output includes:

* Ansible version
* Python version
* Configuration file path

---

## Task 2: Install Ansible on Ubuntu Linux

### Step 1: Connect to your Ubuntu VM

```bash
ssh <username>@<vm-ip-address>
```

### Step 2: Update package list and install prerequisites

```bash
sudo apt update
sudo apt install software-properties-common -y
```

### Step 3: Add the official Ansible repository

```bash
sudo add-apt-repository --yes --update ppa:ansible/ansible
```

### Step 4: Install Ansible

```bash
sudo apt install ansible -y
```

### Step 5: Verify installation

```bash
ansible --version
```

---

## Task 3: Install Ansible Using Python pip

This method works on macOS and Linux systems.

### Step 1: Verify Python 3 installation

```bash
python3 --version
```

If Python is missing, install it from [https://www.python.org](https://www.python.org)

### Step 2: Verify pip availability

```bash
python3 -m pip -V
```

Expected output example:

```text
pip 21.0.1 from /usr/lib/python3.9/site-packages/pip (python 3.9)
```

If pip is missing, install it:

```bash
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py --user
```

### Step 3: Install Ansible using pip

```bash
python3 -m pip install --user ansible
```

### Step 4: Update PATH if required

If you see a PATH warning, run:

```bash
export PATH="$PATH:/home/<your-username>/.local/bin"
source ~/.bashrc
```

For macOS users:

```bash
export PATH="$PATH:$HOME/Library/Python/3.x/bin"
source ~/.zshrc
```

### Step 5: Verify Ansible installation

```bash
ansible --version
```

---

## Task 4: Ansible on Windows

* Windows **cannot act as an Ansible control node**
* You must use a Linux based system

### Recommended approaches:

* Provision a Linux VM on Azure, AWS, or GCP
* Use WSL2 with Ubuntu
* Use a remote Linux server as control node

### Best practice:

* Write Ansible playbooks locally
* Push them to GitHub
* Clone the repository on the Linux control node
* Execute Ansible from the control node

---

## Task 5: Install Ansible Extension for VS Code

![Image](https://raw.githubusercontent.com/wiki/ansible/vscode-ansible/images/activate-extension.gif)

![Image](https://www.golinuxcloud.com/wp-content/uploads/visual_studio.jpg)

### Step 1: Open VS Code

### Step 2: Open Extensions panel

* Click the Extensions icon
* Or press `Ctrl + Shift + X`

### Step 3: Search for Ansible

Search for:

```
Ansible
```

### Step 4: Install the official Ansible extension

This provides:

* YAML syntax highlighting
* Ansible linting
* Playbook autocompletion
* Role and inventory validation

---

## Lab Completion Checklist

* Ansible installed and verified
* Python and pip available
* PATH configured correctly
* VS Code Ansible extension installed
* Control node ready for automation
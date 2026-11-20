# Automated Nginx Web Server Deployment on AWS EC2 with Ansible

![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=flat&logo=amazon-aws&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-Roles_%26_Vault-EE0000?style=flat&logo=ansible&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-Web_Server-009639?style=flat&logo=nginx&logoColor=white)
![Amazon Linux](https://img.shields.io/badge/Amazon_Linux-2023-232F3E?style=flat&logo=amazon-aws&logoColor=white)

## Overview

This project demonstrates production-style cloud configuration management by pairing **Amazon EC2** with **Ansible**. It uses a modular **Ansible Role**, externalized **variables and Jinja2 templating**, **Ansible Vault** for secret encryption, and a **multi-play playbook with `block`/`rescue`/`always` error handling** to deploy and verify an Nginx web server.

The architecture emphasizes **separation of concerns** (cloud compute provisioning vs. software configuration), **secure-by-design networking**, and **idempotency** to eliminate configuration drift.

---

## Project Structure

```text
aws-ec2-ansible-nginx/
├── .gitignore                         # Excludes private SSH keys (.pem) and Vault password files
├── inventory.ini                      # Target EC2 host definition and SSH connection parameters
├── playbook.yml                       # Multi-play orchestration with error handling & health checks
├── group_vars/
│   ├── webservers.yml                 # Non-sensitive environment & application variables
│   └── vault.yml                      # AES256 Ansible Vault encrypted secrets
├── roles/
│   └── nginx/
│       ├── defaults/main.yml          # Default role variables (packages, paths, permissions)
│       ├── handlers/main.yml          # Service restart handler triggered on config/template changes
│       ├── tasks/main.yml             # Modular Nginx installation, service, and template tasks
│       └── templates/index.html.j2    # Dynamic Jinja2 HTML template
├── screenshots/
│   └── nginx-live.png                 # Verification screenshot of the live web page
└── README.md                          # Project documentation
```

---

## Industry Relevance & DevOps Principles

| Principle | How It Is Applied Here | Why It Matters in Production |
| :--- | :--- | :--- |
| **Modular Ansible Roles** | Nginx tasks, handlers, defaults, and templates are isolated in `roles/nginx/`. | Keeps automation reusable, testable, and easy to share across teams. |
| **Variable Separation** | All paths, package names, and page content live in `defaults/` and `group_vars/`. | Eliminates hardcoded values so the same code works across Dev, Staging, and Prod. |
| **Secret Management** | Sensitive values are encrypted at rest in `group_vars/vault.yml` using `ansible-vault`. | Prevents plaintext credentials from leaking into Git history. |
| **Error Handling & Health Checks** | `playbook.yml` uses 3 plays with `block`/`rescue`/`always` and an HTTP `200` check. | Catches failures gracefully and verifies the service is actually responding after deployment. |
| **Idempotency** | Re-running the playbook yields `changed=0` when the server is already in the desired state. | Eliminates configuration drift and avoids unnecessary service restarts. |
| **Secure-by-Design** | Port 22 (SSH) is restricted to a single `/32` IP; key-based authentication replaces passwords. | Minimizes the attack surface against automated SSH brute-force bots while keeping Port 80 open for web traffic. |

---

## Prerequisites

* An active **AWS Account** with permissions to launch EC2 instances and manage Security Groups.
* An **Amazon Linux** EC2 instance (`t2.micro` or `t3.micro`).
* **Security Group** inbound rules allowing **SSH (TCP 22)** from `My IP` (`/32`) and **HTTP (TCP 80)** from `0.0.0.0/0`.
* **Ansible** installed on your local control node (`sudo apt install ansible -y`).
* An **RSA SSH Key Pair** (`.pem`) generated in AWS.

---

## Implementation & Usage

### 1. Secure SSH Access
Move the downloaded private key to `~/.ssh/` and restrict its permissions so OpenSSH accepts it:
```bash
mkdir -p ~/.ssh && mv ~/Downloads/ansible-key.pem ~/.ssh/
chmod 400 ~/.ssh/ansible-key.pem
```

### 2. Configure the Ansible Inventory
Update `inventory.ini` with your EC2 instance's Public IPv4 address, default Amazon Linux user (`ec2-user`), and key path:
```ini
[webservers]
<EC2_PUBLIC_IP> ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/ansible-key.pem
```

### 3. Verify Connectivity & Playbook Syntax
Test SSH reachability and validate the playbook syntax (enter your Vault password when prompted):
```bash
ansible webservers -i inventory.ini -m ping
ansible-playbook -i inventory.ini playbook.yml --syntax-check --ask-vault-pass
```

### 4. Execute the Multi-Play Playbook
Run the playbook against your EC2 instance:
```bash
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass
```

This executes three sequential plays:
1. **System Preparation:** Updates OS packages via `dnf` inside a `block`/`rescue`/`always` error handler.
2. **Nginx Role Deployment:** Decrypts `group_vars/vault.yml`, installs and enables Nginx, and renders `index.html.j2` to `/usr/share/nginx/html/index.html`.
3. **Post-Deployment Health Check:** Sends an HTTP request to `http://localhost` to confirm a `200 OK` response.

Run the playbook a second time to verify idempotency (`changed=0`, `failed=0`).

### 5. Validation
Navigate to `http://<EC2_PUBLIC_IP>` in a web browser to confirm the live deployment.

![Nginx Live Deployment](screenshots/nginx-live.png)

---

## Troubleshooting

* **`Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`:** Amazon Linux uses `ec2-user` rather than `ubuntu`. Set `ansible_user=ec2-user` in `inventory.ini`.
* **`Connection timed out` on Port 22:** Dynamic ISP IP changes can cause the `/32` Security Group rule to block SSH. Update the Port 22 inbound rule to your current **My IP** in the AWS Console.
* **`UNPROTECTED PRIVATE KEY FILE!` error:** Restrict key permissions with `chmod 400 ~/.ssh/ansible-key.pem`.
* **Vault Decryption Error:** Always include `--ask-vault-pass` when running `ansible-playbook` or `--syntax-check`.
* **Browser fails to load the EC2 IP:** Ensure the URL uses `http://` (Port 80) rather than `https://` (Port 443).

---

## Cleanup

Terminate the EC2 instance in the AWS Management Console after verification to avoid ongoing charges.

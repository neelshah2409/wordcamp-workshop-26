# 🚀 From On-Demand to Cloud: Automate WordPress Installations Like a Pro

> **WordCamp Asia · 90-minute Workshop**  
> Speaker: Neel Shah — Developer Advocate, StackGen  
> Stack: Terraform · AWS EC2 · Ubuntu 22.04 · Apache2 · libapache2-mod-php · MySQL · WP-CLI

---

## 📋 Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [One-Time Local Setup](#2-one-time-local-setup)
3. [AWS Credentials](#3-aws-credentials)
4. [Create EC2 Key Pair](#4-create-ec2-key-pair)
5. [Configure tfvars](#5-configure-tfvars)
6. [Terraform: Provision Infrastructure](#6-terraform-provision-infrastructure)
7. [Connect to Your Server](#7-connect-to-your-server)
8. [Watch the Bootstrap Log](#8-watch-the-bootstrap-log)
---

## 1. Prerequisites

Install these tools before the workshop:

| Tool | Version | Install |
|------|---------|---------|
| Terraform | ≥ 1.6 | https://developer.hashicorp.com/terraform/install |
| AWS CLI | ≥ 2 | https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html |
| Git | any | https://git-scm.com/downloads |

Verify installs:

```bash
terraform -version
aws --version
git --version
```

---

## 2. One-Time Local Setup

```bash
# Clone the workshop repo
git clone https://github.com/neelshah2409/wp-aws-terraform.git
cd wp-aws-terraform

# Confirm the structure looks right
ls -la
```

Expected output:
```
main.tf
variables.tf
outputs.tf
terraform.tfvars.example
.gitignore
README.md
scripts/
modules/
```

---

## 3. AWS Credentials

You need an IAM user with **AmazonEC2FullAccess** and **AmazonVPCFullAccess** policies attached.

### Option A — Environment variables (recommended for workshops)

```bash
export AWS_ACCESS_KEY_ID=""
export AWS_SECRET_ACCESS_KEY=""
export AWS_DEFAULT_REGION=""
```

### Option B — AWS CLI configure

```bash
aws configure
# Prompts:
# AWS Access Key ID:      → paste your key
# AWS Secret Access Key:  → paste your secret
# Default region name:    → ap-southeast-1
# Default output format:  → json
```

### Verify credentials work

```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AIDIODR4TAW7CSEXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

> ⚠️ If you see `partial credentials` or `UnauthorizedOperation` — see [Troubleshooting](#13-troubleshooting)

---

## 4. Create EC2 Key Pair

```bash
# Create the .ssh directory if it doesn't exist
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Delete any existing key pair with the same name (if re-running)
aws ec2 delete-key-pair --key-name wp-workshop-key

# Remove any existing local key file
sudo rm -f ~/.ssh/wp-workshop-key.pem

# Create a fresh key pair and save locally
aws ec2 create-key-pair \
  --key-name wp-workshop-key \
  --query 'KeyMaterial' \
  --output text > wp-workshop-key.pem

sudo mv wp-workshop-key.pem ~/.ssh/wp-workshop-key.pem
sudo chown $(whoami) ~/.ssh/wp-workshop-key.pem

# Lock down permissions — SSH refuses keys that are too open
chmod 400 ~/.ssh/wp-workshop-key.pem

# Verify the file has content (~1700 bytes)
wc -c ~/.ssh/wp-workshop-key.pem

# Verify the key starts correctly
head -1 ~/.ssh/wp-workshop-key.pem
# Expected: -----BEGIN RSA PRIVATE KEY-----
```

---

## 5. Configure tfvars

```bash
# Copy the example file
cp terraform.tfvars.example terraform.tfvars
```

Find your public IP:

```bash
curl https://checkip.amazonaws.com
# → 203.0.113.42
```

Edit `terraform.tfvars`:

```bash
nano terraform.tfvars
```

Fill in your values:

```hcl
project      = "wp-workshop"
environment  = "dev"
aws_region   = "ap-south-1"

# Your public IP — format: x.x.x.x/32
your_ip      = "203.0.113.42/32"

# Must match the key pair name you created above
key_name     = "wp-workshop-key"

# EC2 sizing — t3.micro is free-tier eligible
instance_type    = "t3.micro"
root_volume_size = 20

# Ubuntu 22.04 LTS in ap-southeast-1
ami_id = "ami-05d2d839d4f73aafb"
```

> ⚠️ `terraform.tfvars` is in `.gitignore` — never commit it (it contains your IP)

---

## 6. Terraform: Provision Infrastructure

### Step 1 — Initialise

```bash
terraform init
```

Expected output:
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...

Terraform has been successfully initialized!
```

### Step 2 — Preview the plan

```bash
terraform plan
```

Expected output (look for this line at the bottom):
```
Plan: 7 to add, 0 to change, 0 to destroy.
```

The 7 resources are:
- `aws_vpc.main`
- `aws_subnet.public`
- `aws_internet_gateway.main`
- `aws_route_table.public` + `aws_route_table_association.public`
- `aws_security_group.web`
- `aws_instance.wordpress`
- `aws_eip.wordpress`

### Step 3 — Apply

```bash
terraform apply
# Type 'yes' when prompted
# Takes ~60 seconds
```

Expected final output:
```
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

instance_public_ip = "x.x.x.x"
instance_id        = "i-0abc123def456789"
ssh_command        = "ssh -i ~/.ssh/wp-workshop-key.pem ubuntu@x.x.x.x"
site_url           = "http://x.x.x.x"
wp_admin_url       = "http://x.x.x.x/wp-admin"
vpc_id             = "vpc-0abc123"
```

### Print outputs at any time

```bash
# Show all outputs
terraform output

# Get just the IP
terraform output -raw instance_public_ip

# Get the ready-to-run SSH command
terraform output -raw ssh_command
```

---

## 7. Connect to Your Server

```bash
# SSH using the Terraform output directly
ssh -i ~/.ssh/wp-workshop-key.pem ubuntu@$(terraform output -raw instance_public_ip)
```

> ⏳ Wait ~30 seconds after `terraform apply` before SSHing — the instance needs time to pass AWS health checks. If you get `Connection refused`, wait 15 seconds and retry.

---

## 8. Watch the Bootstrap Log

Once SSH'd into the server:

```bash
# Watch the LEMP stack install in real time
sudo tail -f /var/log/bootstrap.log
```

You'll see Nginx, PHP 8.2, MySQL, and WP-CLI install one step at a time.

Wait for this line before moving on:

```
===== Bootstrap completed at <timestamp> =====
LEMP stack is ready. Next step: run scripts/provision-wordpress.sh
```

Press `Ctrl+C` to stop tailing, then exit:

```bash
exit
```
---
Once AWS infra is set up, create an IAM rule named github_actions with the following policies:
- AmazonEC2FullAccess
- AmazonSSMFullAccess
- AmazonSSMManagedInstanceCore

Go to the instance, navigate to Networking, modify the IAM role, and add the github_actions role to it.

Google Doc Link: https://docs.google.com/document/d/1z5emoGWLfU0HCTDZXoXy8nuEUomIuArqQsXOUMbJoVE/edit?usp=sharing
---


## 9. Setup the EC2 instance 

```bash
#!/bin/bash

# Exit on error
set -e

echo "Updating system..."
sudo apt update -y
sudo apt upgrade -y

echo "Installing Apache..."
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2

echo "Installing PHP and extensions..."
sudo apt install -y php php-mysql php-cli php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip libapache2-mod-php

echo "Restarting Apache..."
sudo systemctl restart apache2

echo "Installing MySQL Server..."
sudo apt install mysql-server -y
sudo systemctl enable mysql
sudo systemctl start mysql

# -----------------------------
# MySQL Configuration
# -----------------------------

DB_NAME="wordpress_db"
DB_USER="wp_user"
DB_PASSWORD="StrongPassword123!"

echo "Configuring MySQL..."

sudo mysql <<EOF
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'RootPassword123!';
FLUSH PRIVILEGES;

CREATE DATABASE ${DB_NAME} DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;

CREATE USER '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASSWORD}';
GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'localhost';

FLUSH PRIVILEGES;
EXIT;
EOF

echo "MySQL setup completed."

# -----------------------------
# Apache Configuration for WordPress
# -----------------------------

echo "Enabling Apache rewrite module..."
sudo a2enmod rewrite

echo "Updating Apache config..."
sudo sed -i 's/AllowOverride None/AllowOverride All/g' /etc/apache2/apache2.conf

sudo systemctl restart apache2

# -----------------------------
# Permissions
# -----------------------------

echo "Setting permissions..."
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html

echo "--------------------------------------"
echo "Installation Completed!"
echo "--------------------------------------"
echo "Database Name: ${DB_NAME}"
echo "Database User: ${DB_USER}"
echo "Database Password: ${DB_PASSWORD}"
echo "MySQL Root Password: RootPassword123!"
echo "--------------------------------------"
```
Run the script:

```bash
chmod +x setup.sh

./setup.sh
```
## 10. Github Actions

```
#!/bin/bash

sudo apt update -y
sudo apt install -y git

sudo rm -rf /var/www/html/*

sudo git clone https://github.com/Jigneshyp/gitpipeline_wordpress.git /var/www/html/

sudo systemctl restart apache2

echo "Done!"
```
```
chmod +x gitclone.sh
./gitclone.sh
```
## 11. Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `partial credentials found` | Secret key missing from AWS config | Run `aws configure`, re-enter all four values |
| `UnauthorizedOperation` | IAM user lacks EC2/VPC permissions | Attach `AmazonEC2FullAccess` + `AmazonVPCFullAccess` in AWS Console → IAM |
| `Permission denied: ~/.ssh/*.pem` | File already exists and is locked | `sudo rm -f ~/.ssh/wp-workshop-key.pem` then recreate |
| `InvalidKeyPair.Duplicate` | Key pair already exists in AWS | `aws ec2 delete-key-pair --key-name wp-workshop-key` then recreate |
| SSH `Connection refused` | Instance still booting | Wait 30 seconds and retry |
| SSH `Permission denied (publickey)` | Wrong key or bad permissions | `chmod 400 ~/.ssh/wp-workshop-key.pem` |
| `502 Bad Gateway` / blank page | Apache or PHP mod not running | `sudo systemctl restart apache2` |
| `No valid credential sources found` | Terraform can't see AWS creds | `export AWS_ACCESS_KEY_ID=...` and `export AWS_SECRET_ACCESS_KEY=...` |
| `wc -c` shows 0 bytes on .pem | Key creation failed (creds error) | Fix creds first, then delete and recreate the key pair |
| WP install page instead of site | Provision script hasn't run yet | SSH in and run `sudo bash ~/provision-wordpress.sh` |

### Fix partial credentials step by step

```bash
# 1. Reconfigure
aws configure

# 2. Verify it worked
aws sts get-caller-identity

# 3. Check the credentials file looks complete
cat ~/.aws/credentials
# Should show BOTH aws_access_key_id AND aws_secret_access_key

# 4. Re-run Terraform
terraform plan
```

---

## 14. Repo Structure

```
wp-aws-terraform/
│
├── main.tf                          # Entry point — wires all modules together
├── variables.tf                     # All input variable definitions
├── outputs.tf                       # Values printed after terraform apply
├── terraform.tfvars.example         # Template — copy to terraform.tfvars
├── .gitignore                       # Keeps .pem and tfvars out of git
├── README.md                        # This file
│
├── scripts/
│   └── provision-wordpress.sh       # Run on server: downloads + configures WordPress
│
└── modules/
    ├── networking/
    │   ├── main.tf                  # VPC, public subnet, IGW, route table
    │   ├── variables.tf
    │   └── outputs.tf
    ├── security/
    │   ├── main.tf                  # Security group: SSH (your IP), HTTP/HTTPS (world)
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2/
        ├── main.tf                  # EC2 instance + Elastic IP
        ├── variables.tf
        ├── outputs.tf
        └── scripts/
            └── bootstrap.sh         # Runs on first boot: installs Apache2, libapache2-mod-php, MySQL
```

---

## ⚡ Quick Reference Card

```bash
# ── ONE-TIME SETUP ────────────────────────────────────────────────────────────
mkdir -p ~/.ssh && chmod 700 ~/.ssh
aws configure
aws ec2 delete-key-pair --key-name wp-workshop-key
sudo rm -f ~/.ssh/wp-workshop-key.pem
aws ec2 create-key-pair --key-name wp-workshop-key \
  --query 'KeyMaterial' --output text > ~/.ssh/wp-workshop-key.pem
chmod 400 ~/.ssh/wp-workshop-key.pem
cp terraform.tfvars.example terraform.tfvars
nano terraform.tfvars   # fill in your_ip and key_name

# ── PROVISION INFRASTRUCTURE ─────────────────────────────────────────────────
terraform init
terraform plan
terraform apply          # type 'yes'

# ── WATCH BOOTSTRAP ──────────────────────────────────────────────────────────
ssh -i ~/.ssh/wp-workshop-key.pem ubuntu@$(terraform output -raw instance_public_ip)
sudo tail -f /var/log/bootstrap.log   # wait for "Bootstrap completed"
exit

# ── DEPLOY WORDPRESS ─────────────────────────────────────────────────────────
scp -i ~/.ssh/wp-workshop-key.pem scripts/provision-wordpress.sh \
  ubuntu@$(terraform output -raw instance_public_ip):~
ssh -i ~/.ssh/wp-workshop-key.pem ubuntu@$(terraform output -raw instance_public_ip) \
  "sudo bash ~/provision-wordpress.sh"

# ── OPEN YOUR SITE ───────────────────────────────────────────────────────────
open $(terraform output -raw site_url)          # macOS
xdg-open $(terraform output -raw site_url)      # Linux

# ── TEAR DOWN ────────────────────────────────────────────────────────────────
terraform destroy        # type 'yes'
```

---

*WordCamp Asia · Neel Shah · github.com/neelshah2409/wp-aws-terraform*

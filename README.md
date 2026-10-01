# Startup Company AWS Cloud Infrastructure

A cloud infrastructure project designed for a startup company using **Amazon Web Services (AWS)** to provide a secure, scalable, and highly available environment for internal applications, web services, shared storage, and an ERP system.

The infrastructure is divided into separate environments and uses AWS networking, load balancing, auto scaling, shared storage, managed databases, and Linux servers.

---

## 🏢 Project Overview

The startup requires an infrastructure that can support:

- Internal company applications
- A company ERP system
- Web applications
- Shared files and documents
- Scalable web services
- Centralized database infrastructure
- Secure communication between environments
- Backup and recovery capabilities
- Network isolation between different environments

The architecture is built around **three VPCs**:

```text
                    AWS Cloud
                 Startup Company
                       │
              ┌────────┴────────┐
              │ Transit Gateway │
              └────────┬────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
 ┌────────────┐  ┌────────────┐  ┌────────────┐
 │ Admin VPC  │  │  Dev VPC   │  │  Test VPC  │
 │ 50.0.0.0/16│  │ 60.0.0.0/16│  │ 70.0.0.0/16│
 └────────────┘  └────────────┘  └────────────┘
```

---

# ☁️ AWS Architecture

## VPCs

| Environment | CIDR | Purpose |
|---|---|---|
| Admin | `50.0.0.0/16` | Administration, ERP, and database infrastructure |
| Dev | `60.0.0.0/16` | Development and bastion access |
| Test | `70.0.0.0/16` | Internal web infrastructure and testing |

The environments are separated into independent VPCs and connected through an **AWS Transit Gateway**.

This provides network-level separation while allowing controlled communication between environments.

---

# 🌐 Network Architecture

The project uses AWS networking components including:

- Amazon VPC
- Subnets
- Route Tables
- Internet Gateway where required
- NAT Gateway where required
- Transit Gateway
- Transit Gateway VPC attachments
- Security Groups

The architecture uses private networking for internal application resources.

### High-Level Network

```text
                         AWS
                          │
                  ┌───────┴───────┐
                  │ Transit       │
                  │ Gateway       │
                  └───────┬───────┘
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
      Admin VPC         Dev VPC         Test VPC
      50.0.0.0/16      60.0.0.0/16     70.0.0.0/16
```

---

# 🔐 Network Security

Security Groups are used to control communication between application components.

The general principle is to allow only the required traffic between specific resources.

For example:

```text
Odoo EC2
   │
   │ TCP 5432
   ▼
RDS PostgreSQL
```

The RDS Security Group allows PostgreSQL traffic from the appropriate application Security Group rather than exposing the database publicly.

---

# 🖥️ Bastion Host

A Bastion Host is deployed in the **Dev VPC** and is used as a controlled administration point for accessing private infrastructure.

```text
                    Administrator
                         │
                         │ SSH
                         ▼
                  ┌──────────────┐
                  │ Bastion Host │
                  │    Dev VPC   │
                  └───────┬──────┘
                          │
                          │ Private Network
                          ▼
                 Internal AWS Resources
```

The Bastion Host can be used to reach private resources that should not be directly accessible from the Internet.

The Dev VPC communicates with the other environments through the Transit Gateway.

---

# 🌍 Internal Web Infrastructure

The Test VPC contains an internal web infrastructure designed to provide load balancing and scalability.

The architecture contains:

- 3 existing EC2 web servers
- Internal Application Load Balancer
- Target Group
- Auto Scaling Group
- EFS shared storage
- Nginx

### Architecture

```text
                         TEST VPC
                       70.0.0.0/16
                              │
                              │
                 Internal Company Users
                          │
                          │ HTTP
                          ▼
             ┌─────────────────────────┐
             │      Internal ALB       │
             │ Application Load Balancer│
             └────────────┬────────────┘
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
         ┌───────┐    ┌───────┐    ┌───────┐
         │ Web 1 │    │ Web 2 │    │ Web 3 │
         │  EC2  │    │  EC2  │    │  EC2  │
         └───┬───┘    └───┬───┘    └───┬───┘
             │            │            │
             └────────────┼────────────┘
                          │
                          ▼
                     ┌─────────┐
                     │   EFS   │
                     │ Shared  │
                     │ Storage │
                     └─────────┘

                          │
                          │ Additional Capacity
                          ▼
                 ┌──────────────────┐
                 │ Auto Scaling     │
                 │ Group            │
                 │                  │
                 │ Up to 3 Extra    │
                 │ EC2 Instances    │
                 └────────┬─────────┘
                          │
                          ▼
                     ┌─────────┐
                     │   EFS   │
                     │ Shared  │
                     │ Storage │
                     └─────────┘
```

All components shown above — **Internal ALB, the three baseline web servers, Auto Scaling Group, additional EC2 instances, and EFS — are deployed within the Test VPC (`70.0.0.0/16`)**.
---

# ⚖️ Internal Application Load Balancer

The web application is exposed through an **internal Application Load Balancer**.

This means the load balancer is intended for internal company traffic rather than direct public Internet access.

### Target Group

The target group contains the existing web servers.

Health checks use:

```text
Protocol: HTTP
Port: 80
Path: /
```

Only healthy instances receive traffic.

---

# 🖥️ Web Servers

The baseline infrastructure contains three existing EC2 web servers.

Each server runs:

```text
Amazon Linux
Nginx
```

The servers can be individually identified for load-balancing testing.

Example:

```text
SERVER 1 - hostname
SERVER 2 - hostname
SERVER 3 - hostname
```

Testing the load balancer:

```bash
for i in {1..10}; do
    curl -s http://<INTERNAL-ALB-DNS>
    echo
done
```

This allows the responses from different backend servers to be observed.

---

# 📈 Auto Scaling

The project also includes an Auto Scaling Group for additional capacity.

The architecture separates:

### Baseline capacity

```text
3 existing EC2 servers
```

from:

### Additional capacity

```text
Auto Scaling Group
```

The Auto Scaling Group can launch additional EC2 instances when scaling conditions are met.

The configured maximum additional capacity is:

```text
3 instances
```

Therefore, the architecture can support the three baseline servers plus additional automatically launched servers.

---

# 🚀 Launch Template

New Auto Scaling instances are configured using a Launch Template.

The Launch Template automatically prepares the server.

The initialization process includes:

1. Install Nginx
2. Start Nginx
3. Install NFS utilities
4. Create the EFS mount point
5. Mount EFS
6. Create the web page
7. Display the server hostname

Example User Data:

```bash
#!/bin/bash

yum install -y nginx nfs-utils

systemctl enable --now nginx

mkdir -p /efs

mount -t nfs4 \
-o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
70.0.2.24:/ /efs

echo "AUTO SCALED SERVER - $(hostname)" \
> /usr/share/nginx/html/index.html
```

> The EFS mount target address above is environment-specific. For a multi-AZ implementation, the mount configuration should account for the appropriate EFS mount target in each Availability Zone.

---

# 📂 Shared Storage — Amazon EFS

Amazon EFS provides shared storage for the web servers.

Instead of maintaining separate copies of files on every EC2 instance:

```text
Web Server 1 ─┐
Web Server 2 ─┼──► EFS
Web Server 3 ─┘
```

all servers can access the same shared filesystem.

This is useful for:

- Shared documents
- Application files
- Common content
- Data that must be available across multiple web servers

---

# 🏢 ERP System

The startup company uses an **ERP system** as one of its major business applications.

The ERP implementation uses:

**Odoo 19 Enterprise**

Odoo is therefore a component of the overall company infrastructure rather than the purpose of the entire project.

---

# 🖥️ Odoo ERP Architecture

The Odoo ERP application is deployed in the **Admin VPC**.

The Odoo application runs on an Ubuntu EC2 instance and uses Amazon RDS PostgreSQL as its database.

```text
                    Company Users
                          │
                          │ HTTPS
                          ▼
                     ┌─────────┐
                     │  Nginx  │
                     └────┬────┘
                          │
                          │ HTTP :8069
                          ▼
                  ┌─────────────────┐
                  │   Odoo 19       │
                  │   EC2 Server    │
                  │   Admin VPC     │
                  └────────┬────────┘
                           │
                           │ PostgreSQL :5432
                           ▼
                  ┌─────────────────┐
                  │ Amazon RDS      │
                  │ PostgreSQL      │
                  │ Admin VPC       │
                  └─────────────────┘
```

---

# 🖥️ Odoo Server

The Odoo application runs on:

```text
Ubuntu Linux
Odoo 19.0
```

Main installation directory:

```text
/opt/odoo
```

The installation contains:

```text
/opt/odoo/
├── addons/
├── odoo/
└── enterprise/
```

---

# 🏢 Odoo Enterprise

The Enterprise addons are maintained in a private Git repository.

The Enterprise directory is:

```text
/opt/odoo/enterprise
```

The Odoo configuration includes the Enterprise addons path:

```ini
addons_path = /opt/odoo/addons,/opt/odoo/odoo/addons,/opt/odoo/enterprise
```

This allows Odoo to load:

- Core Odoo modules
- Community addons
- Enterprise addons

> **Tokens and credentials must never be committed to this repository.**

---

# 🗃️ ERP Database

The ERP database is hosted on **Amazon RDS for PostgreSQL** in the **Admin VPC**.

The Odoo EC2 server connects to RDS over the private AWS network.

```text
Admin VPC

┌─────────────────┐
│ Odoo EC2        │
└────────┬────────┘
         │
         │ TCP 5432
         ▼
┌─────────────────┐
│ RDS PostgreSQL  │
└─────────────────┘
```

The database is not directly exposed to the public Internet.

---

# 🌐 Nginx and HTTPS

Nginx is used as a reverse proxy for the Odoo application.

```text
HTTPS :443
     │
     ▼
  Nginx
     │
     │ HTTP :8069
     ▼
  Odoo
```

This separates web traffic handling from the Odoo application process.

SSL certificates and private keys must not be committed to GitHub.

---

# 💽 Storage and EBS

The Odoo EC2 server uses an EBS root volume.

The volume was expanded as additional storage was required by the Odoo Enterprise source.

The process used:

```bash
lsblk
```

Expand the partition:

```bash
sudo growpart /dev/nvme0n1 1
```

Expand the filesystem:

```bash
sudo resize2fs /dev/nvme0n1p1
```

Verify available space:

```bash
df -h /
```

---

# 🔒 Security Architecture

Security is implemented through multiple layers.

### Network Isolation

Separate VPCs provide environment isolation.

### Transit Gateway

The Transit Gateway provides controlled connectivity between the three VPCs.

### Security Groups

Security Groups control which resources can communicate.

For example:

```text
Odoo EC2 ──TCP 5432──► RDS PostgreSQL
```

The RDS Security Group should allow PostgreSQL traffic only from the required application Security Group.

### Internal Load Balancer

The internal web platform uses an internal ALB instead of exposing the application directly to the Internet.

### Private Database

The PostgreSQL database is hosted on RDS and is accessed through private network connectivity.

### Bastion Host

Administrative access to private resources can be performed through the Bastion Host in the Dev VPC.

### Secrets

Credentials must be stored securely and never committed to Git.

---

# 🔑 Secrets Management

The following must **never** be stored in this GitHub repository:

```text
AWS Access Keys
AWS Secret Keys
GitHub Personal Access Tokens
PostgreSQL Passwords
Odoo Master Password
SSH Private Keys
SSL Private Keys
Database Credentials
```

Use placeholders:

```text
<DB_PASSWORD>
<RDS_ENDPOINT>
<GITHUB_TOKEN>
<ODOO_MASTER_PASSWORD>
```

If a credential is accidentally exposed:

1. Revoke or rotate it immediately.
2. Remove it from the repository.
3. Check Git history if required.
4. Replace it with a new credential.

---

# 🧪 Testing

## Test the Internal Load Balancer

From an EC2 instance that has network access to the internal ALB:

```bash
curl http://<INTERNAL-ALB-DNS>
```

Run multiple requests:

```bash
for i in {1..10}; do
    curl -s http://<INTERNAL-ALB-DNS>
    echo
done
```

Different backend responses demonstrate traffic distribution.

---

## Test EFS

Check the mount:

```bash
df -h /efs
```

Create a test file:

```bash
echo "EFS TEST" | sudo tee /efs/test.txt
```

Read it:

```bash
cat /efs/test.txt
```

The same file should be accessible from other servers using the same EFS filesystem.

---

## Test Odoo Connectivity

The Odoo EC2 server should be able to communicate with the RDS PostgreSQL instance through the private network.

Example:

```bash
PGPASSWORD='<PASSWORD>' psql \
-h <RDS_ENDPOINT> \
-U <DB_USER> \
-d <DATABASE> \
-c "SELECT version();"
```

---

# 📊 Monitoring and Operations

The infrastructure can be monitored through AWS services and Linux tools.

Useful AWS components include:

- Amazon CloudWatch
- EC2 monitoring
- Auto Scaling metrics
- ALB health checks
- RDS monitoring
- CloudWatch alarms

Linux diagnostics include:

```bash
df -h
```

```bash
free -h
```

```bash
uptime
```

```bash
sudo ss -lntp
```

---

# 🛡️ Backup and Recovery

Important application and database data should be protected using appropriate AWS backup mechanisms.

Potential recovery mechanisms include:

- RDS automated backups
- RDS snapshots
- EBS snapshots
- EFS backup mechanisms
- Application-level backups

The backup strategy should be tested through restoration procedures rather than relying only on successful backup creation.

---

# 🧭 Project Architecture Summary

The complete architecture is:

```text
                              AWS CLOUD
                                  │
                    ┌─────────────┴─────────────┐
                    │      Transit Gateway      │
                    └─────────────┬─────────────┘
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
             ▼                    ▼                    ▼
      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
      │  ADMIN VPC  │      │   DEV VPC   │      │  TEST VPC   │
      │ 50.0.0.0/16 │      │ 60.0.0.0/16 │      │ 70.0.0.0/16 │
      └──────┬──────┘      └──────┬──────┘      └──────┬──────┘
             │                    │                    │
       ┌─────┴─────┐              │          ┌─────────┴──────────┐
       │           │              │          │                    │
       ▼           ▼              ▼          ▼                    ▼
  ┌────────┐ ┌──────────┐  ┌─────────┐ ┌────────────┐        ┌───────┐
  │ Odoo   │ │   RDS    │  │ Bastion │ │ Internal   │        │  EFS  │
  │  EC2   │ │PostgreSQL│  │  Host   │ │    ALB     │        │       │
  └────────┘ └──────────┘  └─────────┘ └─────┬──────┘        └───┬───┘
                                             │                   │
                                  ┌──────────┼──────────┐        │
                                  │          │          │        │
                                  ▼          ▼          ▼        │
                               ┌──────┐   ┌──────┐   ┌──────┐   │
                               │ Web1 │   │ Web2 │   │ Web3 │───┤
                               └──────┘   └──────┘   └──────┘   │
                                  │          │          │        │
                                  └──────────┼──────────┘        │
                                             │                   │
                                             ▼                   │
                                      Auto Scaling               │
                                        Capacity                 │
                                             │                   │
                                             └───────────────────┘
```

### VPC Responsibilities

```text
ADMIN VPC — 50.0.0.0/16
│
├── Odoo 19 EC2
└── RDS PostgreSQL
```

```text
DEV VPC — 60.0.0.0/16
│
└── Bastion Host
```

```text
TEST VPC — 70.0.0.0/16
│
├── Internal Application Load Balancer
├── Web Server 1
├── Web Server 2
├── Web Server 3
├── Auto Scaling Group
└── EFS Shared Storage
```

The **Transit Gateway** provides connectivity between the three VPCs while keeping each environment logically separated.

---

# 🎯 Project Objectives

### 1. Environment Isolation

Separate Admin, Development, and Test environments using independent VPCs.

### 2. Secure Connectivity

Provide controlled communication between environments through the AWS Transit Gateway.

### 3. Internal Application Availability

Use an internal Application Load Balancer to distribute traffic across multiple web servers.

### 4. Scalability

Use Auto Scaling to provide additional web server capacity when required.

### 5. Shared Storage

Use Amazon EFS so multiple web servers can access common files.

### 6. Managed Database

Use Amazon RDS PostgreSQL instead of maintaining the database directly on the application server.

### 7. ERP Integration

Deploy Odoo 19 Enterprise as the company's ERP platform inside the Admin VPC.

### 8. Secure Administration

Use a Bastion Host in the Dev VPC as a controlled access point for private infrastructure.

### 9. Network Security

Control communication between resources using VPC isolation, Transit Gateway routing, and Security Groups.

---

# 🧰 Technologies Used

| Category | Technology |
|---|---|
| Cloud | Amazon Web Services |
| Compute | Amazon EC2 |
| Networking | Amazon VPC |
| VPC Connectivity | AWS Transit Gateway |
| Load Balancing | Application Load Balancer |
| Scaling | EC2 Auto Scaling |
| Shared Storage | Amazon EFS |
| Block Storage | Amazon EBS |
| Database | Amazon RDS PostgreSQL |
| Web Server | Nginx |
| ERP | Odoo 19 Enterprise |
| Operating Systems | Ubuntu / Amazon Linux |
| Database Client | PostgreSQL |
| Source Control | Git / GitHub |
| Monitoring | Amazon CloudWatch |

---

# 📁 Repository Structure

```text
startup-company-aws/
│
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── networking.md
│   ├── security.md
│   ├── load-balancer.md
│   ├── auto-scaling.md
│   ├── efs.md
│   ├── erp.md
│   ├── database.md
│   ├── monitoring.md
│   └── disaster-recovery.md
│
├── scripts/
│   ├── health-check.sh
│   └── database-check.sh
│
├── config/
│   └── examples/
│
└── .gitignore
```

---

# 🚫 Files That Should Not Be Committed

Example `.gitignore`:

```gitignore
# Secrets
.env
.env.*
*.pem
*.key
*.p12
*.pfx

# Odoo secrets
odoo.conf

# SSH
id_rsa
id_rsa.pub

# Python
__pycache__/
*.py[cod]
.venv/
venv/

# Logs
*.log
logs/

# IDE
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db
```

---

# 🚀 Current Project Status

### AWS Networking

- [x] Admin VPC
- [x] Dev VPC
- [x] Test VPC
- [x] Transit Gateway
- [x] VPC attachments
- [x] Route configuration

### Administration

- [x] Bastion Host in Dev VPC
- [x] Private network connectivity between VPCs

### Internal Web Infrastructure

- [x] EC2 web servers
- [x] Nginx
- [x] Internal Application Load Balancer
- [x] Target Group
- [x] Health checks
- [x] Load-balancing testing
- [x] Launch Template
- [x] Auto Scaling Group
- [x] EFS
- [x] Shared filesystem testing

### ERP

- [x] Odoo 19
- [x] Ubuntu EC2
- [x] Admin VPC deployment
- [x] Amazon RDS PostgreSQL
- [x] Odoo → RDS connectivity
- [x] Odoo Enterprise addons
- [x] Nginx
- [x] HTTPS architecture

### Security

- [x] Security Groups
- [x] Private database connectivity
- [x] Internal load balancing
- [x] Bastion-based administration
- [x] Secrets excluded from documentation

---

# 👨‍💻 Author

**Mohsen Mohamed**

Odoo Developer | AWS / Cloud Infrastructure

This project combines:

```text
Cloud Infrastructure
AWS
Linux
Networking
Odoo ERP
PostgreSQL
Web Infrastructure
Load Balancing
Auto Scaling
Shared Storage
Security
```

---

# 📜 Note

This project is an infrastructure and cloud architecture implementation for a hypothetical startup-company environment.

Odoo Enterprise is used as the company's ERP platform. Odoo and Odoo Enterprise are products of **Odoo S.A.** and are subject to their respective licensing terms.

No proprietary Enterprise source code, credentials, tokens, private keys, or sensitive infrastructure secrets should be published in this repository.

# 🌐 Highly Available Multi-Tier Web Application on AWS

> **AWS Personal Lab Project** | EC2 · ALB · Auto Scaling · VPC · CloudWatch · SSM · IAM

---

## 📋 Project Overview

This project deploys a production-grade, highly available web application on AWS using a multi-tier architecture. The design ensures fault tolerance, automatic scaling, and zero direct SSH access — mirroring what you'd expect in a real enterprise cloud environment.

Everything was built from scratch: network, compute, security, scaling, and monitoring — then validated end-to-end.

---

## 🏗️ Architecture Diagram

```
                        ┌─────────────────────────────────┐
          Users ───────▶│   Application Load Balancer      │
                        │   (Public-facing, Multi-AZ)      │
                        └──────────┬──────────────────────┘
                                   │  Routes to Target Group
                    ┌──────────────▼──────────────────┐
                    │         AUTO SCALING GROUP        │
                    │   Min: 2  |  Max: 6  |  Desired: 2│
                    │                                   │
                    │  ┌─────────────┐ ┌─────────────┐  │
                    │  │  EC2 Web    │ │  EC2 Web    │  │
                    │  │  Server     │ │  Server     │  │
                    │  │  AZ: 1a     │ │  AZ: 1b     │  │
                    │  └─────────────┘ └─────────────┘  │
                    └───────────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │         PRIVATE SUBNET            │
                    │   App / Database Tier             │
                    │   (No internet exposure)          │
                    └───────────────────────────────────┘

         Monitoring: CloudWatch Alarms → SNS → Scale Out/In
         Access:     No SSH — AWS SSM Session Manager only
         Security:   Least-privilege IAM + scoped Security Groups
```

---

## 🔧 AWS Services Used

| Service | Role in Architecture |
|---|---|
| **VPC** | Isolated network with public/private subnets across 2 AZs |
| **EC2** | Web server instances (Amazon Linux 2, Apache/Nginx) |
| **Application Load Balancer** | Distributes traffic across healthy instances in multiple AZs |
| **Auto Scaling Group** | Maintains minimum 2 instances; scales to 6 on CPU demand |
| **Launch Template** | Defines AMI, instance type, SG, user data for zero-touch provisioning |
| **CloudWatch** | CPU, network, and health alarms; custom dashboard |
| **IAM Role** | EC2 instance profile with SSM + S3 permissions (no static keys) |
| **SSM Session Manager** | Secure shell access with no open ports and full audit trail |
| **S3** | Static asset storage (images, CSS, JS) |

---

## 🚀 Deployment Steps

### Step 1: Network Foundation
```bash
# Deploy VPC, subnets (2 public, 2 private), IGW, NAT, route tables
bash scripts/01-create-network.sh
```

### Step 2: Security Configuration
```bash
# Create ALB security group (80/443 from internet)
# Create EC2 security group (80 from ALB-SG only — not internet)
bash scripts/02-create-security-groups.sh
```

### Step 3: IAM Role for EC2
```bash
# Create instance profile with AmazonSSMManagedInstanceCore
# + S3 read access for static assets
bash scripts/03-create-iam-role.sh
```

### Step 4: Launch Template
```bash
# Creates launch template with:
# - Amazon Linux 2 AMI
# - t3.micro instance type
# - User data script: installs Apache, pulls static files from S3
# - SSM instance profile attached
bash scripts/04-create-launch-template.sh
```

### Step 5: Target Group & ALB
```bash
# Creates target group (HTTP, health check on /health)
# Creates ALB in public subnets
# Creates listener: port 80 → target group
bash scripts/05-create-alb.sh
```

### Step 6: Auto Scaling Group
```bash
# Creates ASG: min=2, max=6, desired=2
# Attaches to ALB target group
# Adds CPU scale-out policy at 70%
bash scripts/06-create-asg.sh
```

### Step 7: CloudWatch Monitoring
```bash
# Creates CPU alarm → triggers scale-out
# Creates dashboard: ALB requests, EC2 health, CPU
bash scripts/07-create-monitoring.sh
```

---

## 📄 User Data Bootstrap Script

This script runs automatically on every new EC2 instance — no manual configuration needed:

```bash
#!/bin/bash
# user-data.sh — runs at EC2 boot
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Health check endpoint for ALB
echo "OK" > /var/www/html/health

# Pull website content from S3
aws s3 sync s3://YOUR-BUCKET-NAME/website/ /var/www/html/

# Show instance metadata on homepage
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>Hello from $INSTANCE_ID in $AZ</h1>" >> /var/www/html/index.html
```

---

## 🛡️ Security Design

| Control | Implementation |
|---|---|
| No open SSH | SSM Session Manager — agentless, no port 22 |
| No static credentials | IAM instance profile with least-privilege policies |
| ALB-only inbound | EC2 security group allows port 80 from ALB-SG only |
| Private app tier | App/DB instances in private subnets — no public IPs |
| Audit trail | CloudTrail logs all SSM sessions and API calls |

---

## 📊 CloudWatch Dashboard

The monitoring dashboard tracks:
- **ALB**: Request count, HTTP 5xx errors, healthy host count
- **Auto Scaling**: In-service instances, scale events
- **EC2**: CPU utilization across all instances
- **Alarms**: CPU > 70% for 2 minutes → scale out

---

## 📁 Repository Structure

```
02-ha-multitier-webapp/
├── README.md
├── scripts/
│   ├── 01-create-network.sh
│   ├── 02-create-security-groups.sh
│   ├── 03-create-iam-role.sh
│   ├── 04-create-launch-template.sh
│   ├── 05-create-alb.sh
│   ├── 06-create-asg.sh
│   └── 07-create-monitoring.sh
├── user-data/
│   └── bootstrap.sh              # EC2 user data — auto-runs at launch
├── iam/
│   └── ec2-instance-policy.json  # IAM policy document for EC2 role
└── docs/
    └── architecture-notes.md     # Design decisions and lessons learned
```

---

## 💡 Key Takeaways

1. **ALB health checks** are critical — instances must pass the check before receiving traffic
2. **Cooldown periods** in ASG prevent flapping — set scaling policies with realistic warm-up times
3. **User data runs as root** — scripts execute once at first boot, perfect for bootstrap automation
4. **SSM Session Manager** requires the instance profile AND the SSM agent — Amazon Linux 2 includes it by default
5. **Multi-AZ = fault tolerance** — always spread ASG across at least 2 AZs

---

## 🔗 Related Projects

- [01 - Secure VPC Architecture](../01-secure-vpc-architecture)
- [03 - Auto Scaling & Load Balancer](../03-auto-scaling-load-balancer)
- [07 - CloudWatch Monitoring Dashboard](../07-cloudwatch-monitoring-dashboard)

---

*Built by Abdul Nazir Sadaat | AWS Cloud Engineer | Rancho Cordova, CA*

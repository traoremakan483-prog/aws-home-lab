# ☁️ AWS Home Lab

![Status](https://img.shields.io/badge/status-%E2%9C%85%20Complete-brightgreen)
![AWS](https://img.shields.io/badge/cloud-AWS-orange)
![Region](https://img.shields.io/badge/region-ap--southeast--1-blue)
![Skill Level](https://img.shields.io/badge/level-intermediate-informational)

> A hands-on AWS Home Lab built from scratch to demonstrate core cloud engineering skills: networking, compute, storage, databases, and identity management — all wired together following AWS best practices.

---

## 📖 Overview

This project is a **full AWS environment** designed, deployed, and documented end-to-end as part of my Cloud Engineering studies. The goal was to go beyond theory and prove — with a running, reproducible setup — that I understand how real AWS workloads are architected and secured.

The lab is deployed in the **`ap-southeast-1` (Singapore)** region and includes a custom-built **VPC** with public and private subnets, a **web server** running on EC2, a **static website** hosted on S3, a **private MySQL database** on RDS, and a dedicated **IAM** identity (`makan-admin`) used to operate the environment safely (no root usage).

Every component has been configured manually through the AWS Console and CLI to reinforce a solid understanding of the underlying primitives, rather than abstracting them away with Terraform or CloudFormation from day one.

---

## 🏗️ Architecture

```
                              ┌─────────────────────────────┐
                              │         Internet            │
                              └──────────────┬──────────────┘
                                             │
                                 ┌───────────▼────────────┐
                                 │  Internet Gateway (IGW)│
                                 └───────────┬────────────┘
                                             │
┌────────────────────────────────────────────┼────────────────────────────────────────────┐
│  VPC  10.0.0.0/16                          │                                            │
│                                            │                                            │
│   ┌──────────────────────────────┐    ┌────▼──────────────────────┐                     │
│   │ Public Subnet 10.0.1.0/24    │    │  Route Table (public)     │                     │
│   │                              │    │  0.0.0.0/0  →  IGW        │                     │
│   │   ┌──────────────────────┐   │    └───────────────────────────┘                     │
│   │   │  EC2 (Amazon Linux)  │   │                                                      │
│   │   │  Apache HTTPd        │◄──┼── SG: 22 (my IP), 80 (0.0.0.0/0)                     │
│   │   └──────────┬───────────┘   │                                                      │
│   └──────────────┼───────────────┘                                                      │
│                  │                                                                      │
│                  │ 3306 (MySQL, SG-restricted)                                          │
│                  ▼                                                                      │
│   ┌──────────────────────────────┐    ┌───────────────────────────┐                     │
│   │ Private Subnet 10.0.2.0/24   │    │  Route Table (private)    │                     │
│   │                              │    │  local only — no IGW      │                     │
│   │   ┌──────────────────────┐   │    └───────────────────────────┘                     │
│   │   │  RDS MySQL (private) │   │                                                      │
│   │   └──────────────────────┘   │                                                      │
│   └──────────────────────────────┘                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────┘

     ┌──────────────────────────────┐       ┌──────────────────────────────┐
     │  S3 (Static Website Hosting) │       │  IAM                         │
     │  Public bucket policy        │       │  Dedicated admin user        │
     │  GET object only             │       │  MFA-ready, no root use      │
     └──────────────────────────────┘       └──────────────────────────────┘
```

A more detailed walkthrough lives in [architecture/architecture-description.md](architecture/architecture-description.md).

---

## 🧰 Services Used

| Service | Purpose | Configuration |
|---|---|---|
| **VPC** | Isolated virtual network for the lab | `10.0.0.0/16` with public (`10.0.1.0/24`) and private (`10.0.2.0/24`) subnets in `ap-southeast-1a` |
| **Internet Gateway** | Public egress/ingress for the public subnet | Attached to the VPC, referenced in the public route table |
| **Route Tables** | Traffic routing per subnet | Public: `0.0.0.0/0 → IGW` · Private: local only |
| **EC2** | Web server hosting a live page | `makan-web-server`, Amazon Linux 2023, `t3.micro`, Apache HTTPd, public IP |
| **Security Groups** | Stateful firewall at the ENI level | SSH (22) from my IP only · HTTP (80) from anywhere · MySQL (3306) from EC2 SG only |
| **S3** | Static website hosting | `makan-lab-storage-2026-…` bucket, static hosting enabled + public-read policy on `s3:GetObject` |
| **RDS** | Managed MySQL database | `makan-lab-db`, MySQL Community, `db.t4g.micro`, private subnet, no public access, SG-restricted |
| **IAM** | Identity & access management | `makan-admin` user — root locked away, MFA-ready |

> **Note on instance types:** `t2.micro` (the classic Free Tier default) is **not offered** in `ap-southeast-1` for new accounts; Free Tier in Singapore maps to `t3.micro` for EC2 and `db.t4g.micro` (Graviton/ARM) for RDS. The lab uses those two instead, which is the current AWS-recommended path for Free Tier in this region.

---

## 🎯 Key Concepts Demonstrated

- ✅ **VPC design & CIDR planning** — sized the VPC and subnets intentionally, not by default wizard
- ✅ **Public vs. private subnet segregation** — DB is physically unreachable from the internet by routing, not just by firewall rules
- ✅ **Defense in depth** — security groups + private routing + IAM separation all reinforce each other
- ✅ **Least-privilege networking** — MySQL port is only reachable from the EC2 security group, not a CIDR range
- ✅ **Stateless vs. stateful layers** — web tier on EC2, state on RDS, static assets on S3
- ✅ **Managed vs. self-managed tradeoffs** — RDS for the database, self-managed Apache on EC2 for learning purposes
- ✅ **Public hosting via S3** — bucket policy scoped to `s3:GetObject` only
- ✅ **IAM hygiene** — root account is not used for day-to-day operations
- ✅ **Infrastructure documentation** — every service has its own notes file explaining *why*, not just *what*

---

## ✅ Prerequisites

To reproduce this lab you will need:

- An **AWS account** with billing enabled (Free Tier covers everything in this lab)
- An **IAM user** with admin permissions (do not use root) — mine is `makan-admin`
- The **AWS CLI** installed and configured (`aws configure`)
- An **SSH key pair** created in your target region
- Your **public IP address** (used to scope the SSH security group rule)
- A region — I used **`ap-southeast-1` (Asia Pacific — Singapore)** for everything in this lab

Optional but recommended:
- MFA enabled on both the root account and the IAM admin user
- Billing alerts configured in CloudWatch / Billing preferences

---

## 📂 Project Structure

```
aws-home-lab/
├── README.md                          ← you are here
├── architecture/
│   └── architecture-description.md    ← written walkthrough of the design
├── vpc/
│   └── setup-notes.md                 ← CIDR, subnets, IGW, route tables
├── ec2/
│   └── apache-setup.md                ← EC2 provisioning + Apache install
├── s3/
│   ├── bucket-policy.json             ← public-read policy template
│   └── setup-notes.md                 ← static website hosting steps
├── rds/
│   └── setup-notes.md                 ← private MySQL DB on RDS
├── iam/
│   └── setup-notes.md                 ← admin user, root hardening
└── screenshots/
    └── .gitkeep                       ← console screenshots go here
```

---

## 🚀 What's Next

This lab is the foundation. Planned follow-ups:

- Re-deploy everything as **Terraform** to lock in idempotency
- Add a **NAT Gateway** so the RDS-hosting private subnet can reach the internet for patching
- Put **CloudFront + ACM** in front of the S3 website for HTTPS + CDN
- Move the Apache stack behind an **Application Load Balancer** with an **Auto Scaling Group**
- Pipe EC2 and RDS logs into **CloudWatch Logs** with alarms

---

## 👤 Author

**Built by Makan Traoré — Cloud Engineering Student @ APU Malaysia**

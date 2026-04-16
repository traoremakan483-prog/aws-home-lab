# 🏗️ Architecture Description

This document describes the AWS Home Lab architecture in plain language, explaining **how the pieces fit together** and **why** each decision was made.

---

## 1. The Big Picture

The lab is deployed in the **`ap-southeast-1` (Singapore)** region and forms a small but realistic three-tier-style environment built inside a **custom VPC**:

- A **web tier** (EC2 + Apache) that users reach over HTTP from the public internet
- A **data tier** (RDS MySQL) that lives on a private subnet and is unreachable from the internet
- A **static assets / public site** layer (S3 with static website hosting) that operates outside the VPC
- An **identity layer** (IAM) that ensures no operation is ever done with the root account

All the network components (VPC, subnets, route tables, IGW, security groups) are owned and configured by me rather than inherited from the default VPC. The point was to understand every primitive.

---

## 2. Networking Layout

### VPC

- **CIDR:** `10.0.0.0/16` — gives 65,536 addresses, more than enough headroom for future subnets (NAT GW, extra AZs, a bastion, etc.) without re-plumbing anything
- **Tenancy:** default
- **DNS hostnames / DNS resolution:** enabled, so public-facing EC2 instances get a usable public DNS name

### Subnets

Two subnets, both anchored on `ap-southeast-1a` for this lab (multi-AZ is a planned follow-up):

| Subnet | CIDR | AZ | Type | Purpose |
|---|---|---|---|---|
| Public | `10.0.1.0/24` | `ap-southeast-1a` | Public (routes to IGW) | Hosts the EC2 web server |
| Private | `10.0.2.0/24` | `ap-southeast-1a` | Private (no IGW route) | Hosts the RDS database |

The distinction between "public" and "private" in AWS is **routing**, not anything about the subnet itself. A subnet is public because its route table has a `0.0.0.0/0 → IGW` entry.

### Internet Gateway

- One IGW attached to the VPC
- Referenced in the **public** route table only
- The private subnet never sees it, which is what makes the database unreachable from the internet by construction — not just by firewall rules

### Route Tables

- **Public route table:** `10.0.0.0/16 → local`, `0.0.0.0/0 → igw-xxxx`, associated with the public subnet
- **Private route table:** `10.0.0.0/16 → local` only, associated with the private subnet

---

## 3. Compute Tier — EC2

- **Instance name:** `makan-web-server`
- **AMI:** Amazon Linux 2023 (rolling, patched, minimal)
- **Instance type:** `t3.micro` (Free Tier eligible in `ap-southeast-1` — `t2.micro` is not offered in Singapore for new accounts)
- **Placement:** public subnet, `ap-southeast-1a`
- **Public IP:** auto-assigned
- **User:** `ec2-user`
- **Software:** Apache HTTPd serving a simple static page

The web server is intentionally stateless — everything persistent lives either on S3 or RDS.

---

## 4. Data Tier — RDS

- **DB identifier:** `makan-lab-db`
- **Engine:** MySQL Community
- **Instance class:** `db.t4g.micro` (Graviton/ARM — the Free Tier path for RDS in `ap-southeast-1`; `db.t3.micro` is not Free-Tier-eligible here)
- **Placement:** private subnet only (via a DB subnet group)
- **Public access:** disabled
- **Reachability:** only from the EC2 security group on port `3306`
- **Backups:** automated backups enabled (default retention)

Because the private subnet has no `0.0.0.0/0` route, even if the RDS security group were misconfigured, there is no path from the internet to the database. This is the "belt and braces" approach the VPC design is built around.

---

## 5. Static Hosting — S3

- **Bucket:** `makan-lab-storage-2026-<account-id>-ap-southeast-1-an` with **Static Website Hosting** enabled
- **Index document:** `index.html`
- **Access:** the "Block Public Access" setting is scoped down so that the bucket policy can grant `s3:GetObject` to `Principal: *`
- **Scope of the public policy:** read-only, object-level only — no `s3:ListBucket`, no writes, no deletes

S3 lives outside the VPC. That's intentional: it's a globally addressable service, and fronting it with CloudFront is a natural next step.

---

## 6. Security Groups

Security groups are stateful firewalls attached to ENIs. Three are in use:

| SG | Inbound rules | Notes |
|---|---|---|
| `ec2-web-sg` | `22/tcp` from my public IP only · `80/tcp` from `0.0.0.0/0` | Explicitly **not** open to `0.0.0.0/0` on SSH |
| `rds-mysql-sg` | `3306/tcp` from `ec2-web-sg` | References the SG by ID, not by CIDR, so it stays correct even if the EC2 IP changes |
| (default) | — | Left unused to avoid accidental broad access |

Using an SG reference for MySQL instead of a CIDR range is the detail that makes this least-privilege rather than "least effort."

---

## 7. Identity — IAM

- **Root account:** no access keys, used only for the initial setup, MFA-ready
- **Admin user:** `makan-admin`, a dedicated IAM user with `AdministratorAccess`, used for all day-to-day operations
- **CLI:** configured to use the admin user's access keys via `aws configure`

The guiding rule is simple: **root should never log in once the account is bootstrapped**.

---

## 8. Why This Design

- **Separation of concerns** — compute, storage, and data are distinct tiers with different failure, scaling, and security profiles
- **Blast radius** — a compromised EC2 instance cannot reach the database without also having network permission from `ec2-web-sg`
- **Cost discipline** — everything here fits inside the Free Tier; no NAT Gateway was provisioned (yet) because it isn't free
- **Learning value** — everything is provisioned manually so that the primitives stick. The Terraform re-implementation is the follow-up, not the starting point

---

## 9. Known Limitations

These are intentional scope cuts for v1 of the lab:

- **Single AZ** — the database is not Multi-AZ; a real production setup would span at least two
- **No NAT Gateway** — the private subnet cannot reach the internet at all, which means RDS maintenance windows handle patching rather than in-VPC yum/dnf
- **HTTP only** — the EC2 web server is plain HTTP; HTTPS would need ACM + an ALB or CloudFront
- **No centralized logging** — CloudWatch Logs integration is planned but not in this iteration

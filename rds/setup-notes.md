# 🗄️ RDS (MySQL) — Setup Notes

The database tier of the lab is a **managed MySQL instance on Amazon RDS**, deployed into the private subnet of the custom VPC. It is not reachable from the internet and can only be accessed from the EC2 web server.

---

## What RDS Is

**Amazon RDS** is AWS's managed relational database service. You pick an engine (MySQL here), an instance class, storage, a subnet group, and a security group — and AWS handles the OS, the engine install, patching, backups, and failover. You lose some OS-level control in exchange for a lot of operational work.

This lab uses RDS specifically to demonstrate the **managed vs. self-managed tradeoff**: Apache is installed by hand on EC2 so I learn the OS mechanics, while the database is managed so the lab focuses on networking and identity instead of `my.cnf` tuning.

---

## What I Configured and Why

### 1. Engine and Instance

- **DB identifier:** `makan-lab-db`
- **Region:** `ap-southeast-1` (Singapore)
- **Engine:** MySQL Community (latest minor version available in the AWS console)
- **Template:** Free Tier
- **DB instance class:** **`db.t4g.micro`** (Graviton/ARM) — this is the Free-Tier-eligible class in `ap-southeast-1`. `db.t3.micro` is **not** Free Tier here; `db.t4g.micro` is the correct modern default and gets better price/performance on top.
- **Storage:** 20 GiB, gp3, storage autoscaling disabled (keeps the lab predictable)
- **Deployment:** Single-AZ — good enough for a learning environment, not what I'd pick for prod
- **Master username:** a custom value (not `root`, not `admin`) for habit's sake
- **Master password:** strong, stored in my password manager — never committed anywhere

> **Note on instance class:** AWS is progressively moving RDS Free Tier onto Graviton (`t4g`) in newer regions. In `ap-southeast-1`, the console will block `db.t3.micro` for Free-Tier accounts and steer you to `db.t4g.micro`. This lab uses `db.t4g.micro` — cheaper, ARM-based, and the current recommended path.

### 2. Network & Security

This is the important part.

- **VPC:** `home-lab-vpc`
- **DB subnet group:** built from **the private subnet only** (and its AZ-pair sibling would be added here in a multi-AZ design). RDS requires a subnet group spanning at least two AZs, but in a single-AZ lab you can still wire it up with the subnets you have and keep deployment Single-AZ.
- **Publicly accessible:** **No** — RDS will not assign a public endpoint, even if the SG permitted it
- **VPC security group:** a dedicated `rds-mysql-sg`
- **Inbound rule:** **only** port `3306/tcp` from **the security group ID of the EC2 web server (`ec2-web-sg`)** — not a CIDR range

> **Why reference a security group instead of an IP?** Because the EC2 instance's private IP can change (stop/start cycles, re-launch, scaling). Referencing the SG by ID means the rule stays correct no matter which ENIs are in the group. That's the actual definition of "least privilege" at the network layer.

### 3. Backup & Maintenance

- **Automated backups:** enabled, default retention (7 days)
- **Backup window:** default
- **Maintenance window:** default
- **Minor version auto-upgrade:** enabled so patches land without manual intervention

### 4. Encryption

- **Encryption at rest:** enabled (AWS-managed KMS key is fine for a lab)
- **Encryption in transit:** TLS available and recommended from the client

---

## Connecting from EC2

From the EC2 instance in the public subnet:

```bash
# Install the MySQL client (Amazon Linux 2023)
sudo dnf install mariadb105 -y

# Connect using the RDS endpoint
mysql -h <rds-endpoint>.rds.amazonaws.com -u <master-user> -p
```

> The `mariadb105` package provides a drop-in MySQL-compatible CLI on Amazon Linux 2023.

From my laptop directly: **the connection times out on purpose.** The private subnet has no route to the internet, the RDS instance has no public endpoint, and the SG wouldn't allow it anyway. That is the correct behavior.

---

## Best Practices Applied

- **Private subnet only** — the database cannot be reached from the internet by routing, not just by firewall rules
- **Publicly accessible: No** — second layer of protection, belt-and-braces
- **SG-to-SG rule** on 3306 — the web server can reach the DB; nothing else can
- **No root-like credentials** — a custom master user name keeps default-scan dictionaries from matching
- **Encryption at rest enabled**
- **Automated backups enabled** — non-negotiable even for a lab
- **Minor version auto-upgrade enabled** — a managed DB that you never patch is worse than an unmanaged one

---

## Key Parameters Summary

| Parameter | Value |
|---|---|
| DB identifier | `makan-lab-db` |
| Region | `ap-southeast-1` (Singapore) |
| Engine | MySQL Community |
| Instance class | `db.t4g.micro` (Graviton/ARM) |
| Storage | 20 GiB gp3 |
| Multi-AZ | No (v1) |
| VPC | `home-lab-vpc` |
| Subnet group | private-subnet-backed |
| Publicly accessible | No |
| Security group | `rds-mysql-sg` (3306 from `ec2-web-sg` only) |
| Backup retention | 7 days |
| Encryption at rest | Enabled |
| Auto minor upgrade | Enabled |

---

## Gotchas

- **DB subnet groups need at least two subnets in different AZs** to be *creatable*, even if you only deploy Single-AZ. Plan your subnets accordingly — this is the first thing to fix in v2 of the lab.
- **If you forget to set Publicly Accessible to No**, RDS will provision a public endpoint and your "private" database will resolve to a public IP. Double-check this at create time.
- **RDS security groups want a source that is a security group, not an instance.** If you paste an IP, you'll miss the whole point of SG-referenced rules.
- **The RDS endpoint is a DNS name, not an IP.** If you try to whitelist an IP for the DB, you'll chase a moving target.

---

## What I'd Do Next

- Move to **Multi-AZ** for automatic failover
- Put the master password in **AWS Secrets Manager** and have EC2 pull it at runtime via IAM
- Enable **Performance Insights** and **Enhanced Monitoring**
- Add a **read replica** for analytical queries without touching the primary

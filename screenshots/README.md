# 📸 Screenshots

This folder is where the AWS Console captures that document the running lab live.

## Suggested filenames

Save your captures with these names so the README and notes files can reference them later without breaking:

| Filename | What it shows |
|---|---|
| `01-console-home.png` | AWS Console home — region set to Asia Pacific (Singapore), `makan-admin` signed in, "Recently visited" list showing IAM, RDS, KMS, VPC, S3, EC2 |
| `02-ec2-instance-running.png` | EC2 Instances view — `makan-web-server` running on `t3.micro` in `ap-southeast-1a`, 3/3 status checks passed |
| `03-s3-bucket.png` | S3 Buckets view — `makan-lab-storage-2026-…` bucket listed in `ap-southeast-1`, created April 15, 2026 |
| `04-rds-database.png` | RDS Databases view — `makan-lab-db` Available, MySQL Community on `db.t4g.micro` in `ap-southeast-1` |

## Things to check before committing

- ❌ Don't commit screenshots that expose your **access keys** (`AKIA…` strings), **secret keys**, or **public IPv4 addresses** of EC2 instances you're still using
- ⚠️ AWS account IDs are not secret per se (they show up in every ARN you ever share), but for a public portfolio repo many people still prefer to blur them — your call
- ✅ Safe to show: region names, resource names, instance types, status/health, AZ names, creation dates

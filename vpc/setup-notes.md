# 🌐 VPC — Setup Notes

## What a VPC Is

A **Virtual Private Cloud** is a logically isolated section of the AWS network that you own. Inside it, you control the IP address range, subnets, route tables, gateways, and which resources can talk to what. Think of it as your own datacenter-shaped slice of AWS.

Every non-serverless resource (EC2, RDS, ELB, …) lives inside a VPC. The default VPC AWS gives you works, but building a custom one is where you actually learn networking on AWS.

---

## What I Configured and Why

### 1. The VPC itself

- **Name tag:** `home-lab-vpc`
- **Region:** `ap-southeast-1` (Asia Pacific — Singapore)
- **IPv4 CIDR:** `10.0.0.0/16`
- **IPv6:** none (not needed for this lab)
- **Tenancy:** default

> **Why `/16`?** It gives 65k addresses. Way more than the lab needs, but the cost is zero and it leaves room for more subnets (extra AZs, a NAT subnet, a bastion subnet…) later without re-planning CIDRs.

### 2. Subnets

| Name | CIDR | AZ | Type |
|---|---|---|---|
| `home-lab-public-1a` | `10.0.1.0/24` | `ap-southeast-1a` | Public |
| `home-lab-private-1a` | `10.0.2.0/24` | `ap-southeast-1a` | Private |

> **Why `/24`?** 256 addresses per subnet — plenty for the handful of ENIs in this lab, and a familiar size that matches most tutorials.

> **Why a single AZ?** For the cost and complexity floor of a learning lab. Multi-AZ is a planned v2.

- **Auto-assign public IPv4:** enabled on the **public** subnet only

### 3. Internet Gateway

- **Name tag:** `home-lab-igw`
- **State:** attached to `home-lab-vpc`

The IGW is what actually lets the public subnet reach (and be reached from) the internet. Without it, `0.0.0.0/0` has nowhere to go.

### 4. Route Tables

Two route tables, one per subnet type:

**Public route table (`home-lab-public-rt`)**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | `igw-xxxxxxxx` |

Associated with the public subnet.

**Private route table (`home-lab-private-rt`)**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |

Associated with the private subnet. **No `0.0.0.0/0` route** — this is the core of the design.

---

## Best Practices Applied

- **Intentional CIDR planning** instead of accepting defaults. `10.0.0.0/16` / `/24` subnets leave clear room to grow
- **Public/private separation enforced by routing**, not just by security groups. Belt and braces
- **One route table per subnet type** rather than reusing the VPC main route table, so the intent is explicit
- **Named/tagged everything** (`home-lab-*`) so the console is readable and cleanup is painless
- **Auto-assign public IP disabled on the private subnet** so it's physically impossible to accidentally launch a publicly-addressable instance there

---

## Key Parameters & Commands

If you prefer the CLI, the equivalent of what I did in the console:

```bash
# Create the VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=home-lab-vpc}]'

# Enable DNS hostnames
aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-hostnames

# Create the public subnet
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-southeast-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=home-lab-public-1a}]'

# Create the private subnet
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-southeast-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=home-lab-private-1a}]'

# Create and attach the Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=home-lab-igw}]'
aws ec2 attach-internet-gateway --vpc-id <vpc-id> --internet-gateway-id <igw-id>

# Public route table + default route + association
aws ec2 create-route-table --vpc-id <vpc-id>
aws ec2 create-route \
  --route-table-id <rtb-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <igw-id>
aws ec2 associate-route-table --route-table-id <rtb-id> --subnet-id <public-subnet-id>

# Private route table (no default route) + association
aws ec2 create-route-table --vpc-id <vpc-id>
aws ec2 associate-route-table --route-table-id <private-rtb-id> --subnet-id <private-subnet-id>

# Enable auto-assign public IPv4 on the public subnet
aws ec2 modify-subnet-attribute --subnet-id <public-subnet-id> --map-public-ip-on-launch
```

---

## Gotchas I Ran Into

- **Forgetting to associate the route table with the subnet** — AWS doesn't associate it for you. The subnet silently falls back to the main route table and nothing works the way you expect.
- **Auto-assign public IP is a subnet setting**, not an instance setting. If you forget to enable it on the public subnet, new instances launched there won't get a public IP even if they should.
- **Naming matters.** An hour into console work, untagged resources all look identical. Tag as you go.

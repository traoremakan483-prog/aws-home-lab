# 🖥️ EC2 + Apache — Setup Notes

This document covers the EC2 instance that hosts the live web page for the Home Lab, and walks through the exact steps used to install and configure Apache on it.

---

## What EC2 Is (Briefly)

**Amazon EC2** (Elastic Compute Cloud) gives you virtual machines on demand. You choose an AMI (the OS image), an instance type (the hardware shape), a subnet (where it lives on the network), a security group (its firewall), and a key pair (how you SSH in). AWS handles the hypervisor, the hardware, and the physical network — you own everything above the OS.

---

## Instance Configuration

| Setting | Value | Reason |
|---|---|---|
| **Instance name** | `makan-web-server` | Descriptive tag, matches how it's labeled in the console |
| **Region** | `ap-southeast-1` (Singapore) | All lab resources live here |
| **AMI** | Amazon Linux 2023 | Maintained by AWS, `dnf`-based, minimal, Free Tier friendly |
| **Instance type** | `t3.micro` | Free Tier eligible in `ap-southeast-1` — **`t2.micro` is not available** for new accounts in Singapore, so `t3.micro` is the correct modern default here |
| **VPC** | `home-lab-vpc` | The custom VPC created for this lab |
| **Subnet** | `home-lab-public-1a` (`ap-southeast-1a`) | Public subnet — needs internet reachability |
| **Auto-assign public IP** | Enabled | So the instance gets a routable IP |
| **Security group** | `ec2-web-sg` | SSH (22) from my IP only, HTTP (80) from anywhere |
| **Key pair** | Existing RSA key pair in the region | Required for SSH access; private key stored locally |
| **Root volume** | 8 GiB gp3 | Default size is enough for Apache + a static page |

> **Note on instance type:** AWS has been rolling off `t2.micro` in newer regions. In `ap-southeast-1`, the Free Tier now maps to **`t3.micro`**, which is a Nitro-based instance with better baseline performance than `t2`. Using `t3.micro` is the current AWS-recommended path for Free Tier here — not a downgrade, it's the replacement.

### Security group (`ec2-web-sg`) inbound rules

| Type | Protocol | Port | Source | Reason |
|---|---|---|---|---|
| SSH | TCP | 22 | *my public IP / 32* | Admin access, locked to me |
| HTTP | TCP | 80 | `0.0.0.0/0` | The whole point — public web server |

Outbound is left at the default (allow all) so Apache and `dnf` can reach the internet for packages and updates.

---

## Connecting to the Instance

Once launched, from a local shell:

```bash
chmod 400 ~/path/to/my-keypair.pem
ssh -i ~/path/to/my-keypair.pem ec2-user@<EC2-PUBLIC-DNS>
```

> `chmod 400` is mandatory — SSH refuses to use a key file with permissive permissions.

---

## Apache Installation — Step by Step

Here are the exact commands run on the EC2 instance, followed by what each one actually does.

```bash
sudo dnf update -y
```

Updates every installed package on Amazon Linux 2023 to the latest version. The `-y` flag auto-confirms so the update doesn't pause on a prompt. Running this before installing anything new avoids the classic "package X needs a newer libY" dependency drift.

```bash
sudo dnf install httpd -y
```

Installs the Apache HTTP server (`httpd` is the package/daemon name). The web server binary lands at `/usr/sbin/httpd`, and the default document root is created at `/var/www/html`.

```bash
sudo systemctl start httpd
```

Starts the Apache service **right now** in the current boot session. Without this, the package is installed but nothing is listening on port 80.

```bash
sudo systemctl enable httpd
```

Tells systemd to start Apache automatically at every boot. This is the one people forget — without it, a reboot silently breaks the web server.

```bash
sudo systemctl status httpd
```

Verifies that Apache is actually running. You want to see `active (running)` in green. This is the moment to catch any startup error (port conflict, permissions, missing config) before you go further.

```bash
sudo bash -c 'echo "<h1>Makan Cloud Lab</h1>" > /var/www/html/index.html'
```

Writes a minimal HTML file to Apache's default document root. A couple of details:

- `sudo bash -c '...'` is used because the redirection (`>`) is evaluated by the shell, so `sudo` has to own the whole command including the redirect — otherwise only `echo` would run as root and the redirect would fail on the permission check.
- `index.html` is what Apache serves by default when the URL path is `/` — so visiting `http://<EC2-PUBLIC-DNS>/` shows this page immediately.

---

## Verifying the Setup

From my laptop:

```bash
curl http://<EC2-PUBLIC-DNS>
```

Expected output:

```html
<h1>Makan Cloud Lab</h1>
```

Or, from a browser: open `http://<EC2-PUBLIC-DNS>` and the page renders.

If it doesn't:

1. Check the security group allows port 80 from your IP
2. Confirm `systemctl status httpd` says `active (running)`
3. Make sure you're using **HTTP** not HTTPS — this lab is plain HTTP only

---

## Best Practices Applied

- **No SSH from `0.0.0.0/0`.** SSH is restricted to my IP. This single change blocks the overwhelming majority of opportunistic scanning.
- **`systemctl enable` always pairs with `start`.** Ensures the service survives a reboot.
- **Private key kept off the instance.** Only the public key is on EC2 (through the key pair); the private key stays on my laptop.
- **No secrets baked into `index.html`.** It's a static demo page, safe to be public.
- **`dnf update` before any install.** Old packages, new install = dependency pain.
- **Use the AWS-managed AMI** (Amazon Linux 2023) instead of a random community AMI — less attack surface, faster patching.

---

## What I'd Do Next

- Put the EC2 instance behind an **Application Load Balancer** and scale it with an **Auto Scaling Group**
- Add **ACM + HTTPS** via an ALB listener on 443
- Move from SSH to **AWS Systems Manager Session Manager** so I can close port 22 entirely
- Ship Apache access/error logs to **CloudWatch Logs** via the CloudWatch Agent

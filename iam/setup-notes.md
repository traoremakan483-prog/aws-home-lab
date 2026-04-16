# 🔐 IAM — Setup Notes

IAM (Identity & Access Management) is how you say **who** can do **what** inside an AWS account. Getting this right on day one is the single highest-leverage security decision in any AWS environment.

---

## What IAM Is

**IAM** lets you create and manage:

- **Users** — long-lived identities for humans (or, in the old days, for apps)
- **Groups** — collections of users that share permissions
- **Roles** — identities that get *assumed* temporarily, usually by AWS services (e.g. an EC2 instance assuming a role to call S3)
- **Policies** — JSON documents that allow or deny specific API actions on specific resources

The **root user** is the identity you sign up with — it has unlimited power, cannot be constrained, and you want to use it as little as humanly possible.

---

## What I Configured and Why

### 1. Root account hardening

- **Strong root password** stored in a password manager
- **No root access keys.** If any existed, they're deleted. Root API credentials are the worst possible thing to leak.
- **MFA-ready** — the workflow is set up to enable virtual MFA on root

> **Why no root access keys?** Because a root access key can do anything in the account, bypasses SCPs in certain edge cases, and cannot be meaningfully restricted. AWS's own guidance is explicit: don't use root access keys.

### 2. Admin IAM user

- **Name:** `makan-admin` — a dedicated admin user for me, not `admin`, not `root`, not something a scanner would guess
- **Access type:** console access + programmatic access (access key + secret key)
- **Permissions:** attached the AWS-managed `AdministratorAccess` policy
- **Console password:** strong, stored in a password manager
- **Access key ID / secret:** stored in `~/.aws/credentials` via `aws configure`, never committed to git
- **Default region for the CLI:** `ap-southeast-1`

### 3. Everyday usage pattern

- The **root user** is locked in a drawer — only used when the task *must* be done as root (a handful of things like closing the account, changing the account email, etc.)
- **All day-to-day operations** (console clicks, CLI calls, Terraform later on) run as the admin IAM user
- **The admin user's access keys are kept local** and never appear in any file inside this repo

### 4. CLI configuration

```bash
aws configure
# AWS Access Key ID: ******************
# AWS Secret Access Key: **************************
# Default region name: ap-southeast-1
# Default output format: json
```

Credentials land in `~/.aws/credentials` and `~/.aws/config`. Both files are outside the repo and untouched by `git`.

---

## Best Practices Applied

- ✅ **Don't use root.** Ever, beyond the initial bootstrap.
- ✅ **No root access keys.** Deleted if any existed.
- ✅ **MFA-ready on root and on the admin user.**
- ✅ **One human, one IAM user.** No shared logins.
- ✅ **Named with intent.** Not `admin` / `user1` / `test`.
- ✅ **Access keys stored only locally**, in the standard `~/.aws/credentials` file. Never in source control, never in documents, never in screenshots.
- ✅ **Least privilege is the goal, even when the user has admin today.** The next iteration is to split permissions by function (network, DB, storage, read-only auditor) — admin is scaffolding for the lab, not the end state.

---

## What I Would Harden Next (v2)

- **Enforce MFA** on every IAM user through a policy condition
- **Split admin into scoped users/roles** — `lab-network-admin`, `lab-db-admin`, `lab-readonly` — so even *I* don't have blanket admin day-to-day
- **Switch from long-lived IAM users to IAM Identity Center (SSO)** with short-lived credentials
- **Give EC2 an IAM Role** (instance profile) for anything it needs to do with AWS APIs, instead of placing keys on the instance
- **Turn on CloudTrail + GuardDuty** so unusual IAM activity actually gets noticed

---

## Key Steps (CLI equivalent)

```bash
# Create the admin user
aws iam create-user --user-name makan-admin

# Attach the AdministratorAccess managed policy
aws iam attach-user-policy \
  --user-name makan-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Give them a console login password (requires reset on first login)
aws iam create-login-profile \
  --user-name makan-admin \
  --password '<strong-password>' \
  --password-reset-required

# Generate programmatic access keys (save these — secret is shown once)
aws iam create-access-key --user-name makan-admin
```

> After this, `aws configure` is run with the new keys and the root account goes back in the drawer.

---

## Gotchas

- **Secret access keys are shown exactly once.** If you miss it, delete the key and create a new one — there is no "show again."
- **`AdministratorAccess` is not the same as root.** Admin can do almost anything, but some account-level actions (close the account, change the root email/password, change support plan, change tax info) still require root.
- **Access keys in a screenshot are leaked keys.** Redact `AKIA…` strings before publishing anything.
- **Never commit `~/.aws/credentials`.** Add it to your `.gitignore` prophylactically even though it lives in `$HOME` — one stray `cp` at the wrong moment is all it takes.

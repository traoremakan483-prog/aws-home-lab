# 🪣 S3 — Setup Notes

This bucket is configured as a **static website** host: it serves plain HTML directly, reachable by its S3 website endpoint, no server required.

---

## What S3 Is

**Amazon S3** (Simple Storage Service) is AWS's object store. It holds files (objects) inside named containers (buckets). Objects are addressed by key (the "path"), there's no real filesystem, and the service is globally addressable — buckets live outside of any VPC.

Two notable things about S3 for this lab:

1. **Bucket names are globally unique** across all AWS accounts. If the name you want is taken, you'll get a clear error at create time.
2. **Static website hosting** is a bucket-level feature that makes S3 respond to HTTP GETs the way a simple web server would (index document, error document, website URL).

---

## What I Configured and Why

### 1. Bucket

- **Name:** `makan-lab-storage-2026-<account-id>-ap-southeast-1-an` (globally unique, includes the account ID as a disambiguator so bucket-name collisions are impossible)
- **Region:** `ap-southeast-1` (Singapore) — same region as the rest of the lab
- **Created:** April 15, 2026 (UTC+08:00)
- **Object Ownership:** Bucket owner preferred (ACLs disabled) — modern recommendation
- **Block Public Access:** **partially disabled** so the bucket policy can grant public read. Only the settings that would block bucket-policy-based public access are turned off; the ACL-related blocks remain on.

> **Why disable Block Public Access at all?** Because this bucket is **intentionally** public (that's the whole point of a static website). The risk you're guarding against with BPA is *accidental* public access; when it's deliberate and scoped, it's fine.

### 2. Static Website Hosting

- **Static website hosting:** Enabled
- **Index document:** `index.html`
- **Error document:** `error.html` (optional, a simple 404 page)

Once enabled, the bucket gets a **website endpoint** of the form:

```
http://<bucket-name>.s3-website-ap-southeast-1.amazonaws.com
```

That endpoint is what users hit — not the regular S3 REST endpoint.

### 3. Uploaded Content

- `index.html` — a minimal landing page
- `error.html` — a minimal 404 page

No sensitive content. No credentials. No private data.

### 4. Bucket Policy

See [bucket-policy.json](bucket-policy.json). It grants **only** `s3:GetObject` on objects inside the bucket to **any** principal. It does **not** grant:

- `s3:ListBucket` (can't enumerate the bucket contents)
- `s3:PutObject` (can't upload)
- `s3:DeleteObject` (can't delete)
- anything at the bucket level (`arn:aws:s3:::<name>` without the `/*`)

That's the minimum scope needed to serve a static site.

> Replace `NOM-DU-BUCKET` with your actual bucket name before applying the policy.

---

## Best Practices Applied

- **Least-privilege policy:** only `s3:GetObject`, only on `*` objects inside the bucket, nothing else.
- **No wildcard action, no wildcard resource at the bucket level.** The resource is `arn:aws:s3:::<name>/*` — objects only, not the bucket itself.
- **ACLs disabled** (Bucket owner preferred), because ACL-based access control is the legacy path and mixing ACLs with policies is where most public-bucket mistakes come from.
- **BPA only partially disabled.** Only the specific settings that block bucket-policy-based public grants. Anything ACL-related stays blocked.
- **Versioning** can be enabled on top of this if the site needs rollback history — it's independent of the public-read setup.

---

## Key Steps (CLI equivalent)

```bash
# Create the bucket
aws s3api create-bucket \
  --bucket <my-unique-bucket-name> \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Disable the BPA settings that would block policy-based public access
aws s3api put-public-access-block \
  --bucket <my-unique-bucket-name> \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Enable static website hosting
aws s3 website s3://<my-unique-bucket-name>/ \
  --index-document index.html \
  --error-document error.html

# Apply the public-read bucket policy (edit bucket-policy.json first)
aws s3api put-bucket-policy \
  --bucket <my-unique-bucket-name> \
  --policy file://bucket-policy.json

# Upload the site
aws s3 cp index.html s3://<my-unique-bucket-name>/
aws s3 cp error.html s3://<my-unique-bucket-name>/
```

The site is then reachable at:

```
http://<my-unique-bucket-name>.s3-website-ap-southeast-1.amazonaws.com
```

---

## Verifying

```bash
curl http://<my-unique-bucket-name>.s3-website-ap-southeast-1.amazonaws.com
```

A successful request returns the HTML of `index.html`. A request to a missing key returns the content of `error.html` with a 404.

---

## Gotchas

- **The website endpoint is different from the REST endpoint.** Only the *website* endpoint honors the index/error document configuration.
- **Block Public Access overrides everything.** If BPA is fully on, the policy is valid but access is still denied. The console gives a helpful banner about this — read it.
- **Policy ARN syntax matters.** Granting on `arn:aws:s3:::<name>` (no `/*`) doesn't cover the objects. Granting on `arn:aws:s3:::<name>/*` covers the objects but not the bucket itself. You typically only want the latter.
- **HTTPS requires CloudFront.** S3 website endpoints are HTTP only. Putting CloudFront in front with an ACM certificate is the standard answer.

# AWS Profile Setup

## Purpose

Configure AWS CLI profiles for MAIP environments. **Run profile setup once per machine.**

---

## Validated Setup Steps

### Step 1: Configure Profiles

```bash
# Development
aws configure --profile maip-dev
# Prompts: Access Key, Secret Key, Region: us-east-1, Output: json

# QA
aws configure --profile maip-qa

# Staging
aws configure --profile maip-staging

# Production
aws configure --profile maip-prod
```

### Step 2: Add Shell Aliases (Optional but Recommended)

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias maip-dev='export AWS_PROFILE=maip-dev AWS_REGION=us-east-1 MAIP_ENV=dev && echo "Switched to MAIP Dev"'
alias maip-qa='export AWS_PROFILE=maip-qa AWS_REGION=us-east-1 MAIP_ENV=qa && echo "Switched to MAIP QA"'
alias maip-staging='export AWS_PROFILE=maip-staging AWS_REGION=us-east-1 MAIP_ENV=staging && echo "Switched to MAIP Staging"'
alias maip-prod='export AWS_PROFILE=maip-prod AWS_REGION=us-east-1 MAIP_ENV=prod && echo "Switched to MAIP Prod"'
alias maip-prod-dr='export AWS_PROFILE=maip-prod AWS_REGION=us-west-2 MAIP_ENV=prod && echo "Switched to MAIP Prod DR"'
alias maip-whoami='aws sts get-caller-identity && echo "Region: $AWS_REGION Environment: $MAIP_ENV"'
```

Reload: `source ~/.bashrc` or `source ~/.zshrc`

### Step 3: Verify

```bash
maip-dev  # or: export AWS_PROFILE=maip-dev AWS_REGION=us-east-1
aws sts get-caller-identity
```

**Expected Output:**
```json
{
    "UserId": "AIDAXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

---

## Environment Matrix

| Profile | Environment | Primary Region | DR Region | Use Case |
|---------|-------------|----------------|-----------|----------|
| `maip-dev` | Development | us-east-1 | - | Daily development |
| `maip-qa` | QA | us-east-1 | - | Testing |
| `maip-staging` | Staging | us-east-1 | us-west-2 | Pre-production |
| `maip-prod` | Production | us-east-1 | us-west-2 | Live system |

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Unable to locate credentials` | Profile not configured | Run `aws configure --profile maip-dev` |
| `ExpiredToken` | Session expired | Re-run SSO login or refresh credentials |
| `Access Denied` | Wrong profile or permissions | Verify profile with `aws sts get-caller-identity` |
| `Region not set` | Missing region | Add `AWS_REGION=us-east-1` to export |

---

## Required Before Other AWS Operations

**ALWAYS run one of these before any AWS command:**

```bash
export AWS_PROFILE=maip-dev AWS_REGION=us-east-1 MAIP_ENV=dev
```

Or use alias:
```bash
maip-dev
```

This sets three required variables: `AWS_PROFILE`, `AWS_REGION`, `MAIP_ENV`

# AWS Profile Setup - MAIP

## Profiles

| Profile | Environment | Region | Purpose |
|---------|-------------|--------|---------|
| maip-dev | Development | us-east-1 | Daily development |
| maip-qa | QA | us-east-1 | Testing |
| maip-staging | Staging | us-east-1, us-west-2 | Pre-production |
| maip-prod | Production | us-east-1, us-west-2 | Live system |

## Configure Profiles

```bash
# Development
aws configure --profile maip-dev
# Enter: Access Key, Secret Key, Region: us-east-1, Output: json

# QA
aws configure --profile maip-qa

# Staging
aws configure --profile maip-staging

# Production
aws configure --profile maip-prod
```

## Shell Aliases (add to ~/.bashrc or ~/.zshrc)

```bash
# MAIP AWS Profile Aliases
alias maip-dev='export AWS_PROFILE=maip-dev AWS_REGION=us-east-1 && echo "Switched to MAIP Dev"'
alias maip-qa='export AWS_PROFILE=maip-qa AWS_REGION=us-east-1 && echo "Switched to MAIP QA"'
alias maip-staging='export AWS_PROFILE=maip-staging AWS_REGION=us-east-1 && echo "Switched to MAIP Staging"'
alias maip-prod='export AWS_PROFILE=maip-prod AWS_REGION=us-east-1 && echo "Switched to MAIP Prod (us-east-1)"'
alias maip-prod-dr='export AWS_PROFILE=maip-prod AWS_REGION=us-west-2 && echo "Switched to MAIP Prod DR (us-west-2)"'

# Quick status check
alias maip-whoami='aws sts get-caller-identity && echo "Region: $AWS_REGION"'
```

## Verify Setup

```bash
# Check current identity
maip-dev
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAXXXXXXXXXX",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/your-username"
# }
```

## Environment Variables for Applications

```bash
# Set for local Spring Boot / Python development
export MAIP_ENV=dev
export AWS_PROFILE=maip-dev
export AWS_REGION=us-east-1
```

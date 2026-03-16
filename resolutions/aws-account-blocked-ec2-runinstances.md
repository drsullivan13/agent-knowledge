## AWS account blocked during EC2 RunInstances
**Date:** 2026-03-16
**Context:** AWS CLI / Terraform / EC2 provisioning
**Tags:** aws, terraform, ec2, runinstances, account-blocked, deployment

### Problem / Observation

`terraform apply` can partially succeed (IAM, SG, Secrets, CloudWatch, ECR created) but fail when creating `aws_instance` with:

`Blocked: This account is currently blocked and not recognized as a valid account ... account-verification`

This is an account-level restriction, not a Terraform configuration error.

### Resolution / Insight

Treat this as a hard external blocker and stop retry loops after one confirmation retry. Capture evidence with the failing `terraform apply` output plus EC2 state query. Escalate to account owner to resolve AWS account verification in Support.

### Commands / Code

```bash
# Reproduce/confirm blocker
cd /Users/dansullivan/workspace/kalshi-agent/infra
terraform apply -auto-approve -var-file=terraform.tfvars

# Show no running/pending instances for project tag
aws ec2 describe-instances \
  --region us-east-1 \
  --filters 'Name=tag:Project,Values=kalshi-agent' \
  --query 'Reservations[].Instances[].State.Name' \
  --output json

# Keep successful evidence from same session (e.g., ECR image push) separate
aws ecr describe-images --region us-east-1 --repository-name kalshi-agent
```

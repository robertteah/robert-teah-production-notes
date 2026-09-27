# Terraform State Backend — S3 + DynamoDB Locking

## The incident that made this non-negotiable

At Luxottica, on Project VisionLens, our Terraform remote state lived in S3 with no locking in front of it. It worked — until two engineers ran `terraform apply` against the same staging environment within 90 seconds of each other: one adding an EKS node group, one updating an RDS parameter group.

Both applies reported success. Twenty minutes later, a third engineer ran `terraform plan` and saw a wall of unexpected resource changes. With no DynamoDB lock, both applies had read the same initial state, computed their diffs independently, and written their results back — last write won, silently overwriting part of the other's changes. The real AWS resources were fine; Terraform's own record of them wasn't, which is the dangerous case: the next `apply` would have tried to "correct" working infrastructure back to stale values.

Recovery was surgical — `terraform state rm` + `terraform import` for the specific out-of-sync resources, rather than a full restore, since the actual AWS state was correct. Recovery time: 2.5 hours, and only possible because S3 versioning was already enabled. Without it, this would have been significantly worse.

## What this backend enforces

- **DynamoDB locking on every environment.** No exceptions, no "just this once for a quick change."
- **Separate state files per environment *and* per component** (VPC state isolated from EKS state, isolated from RDS state). A bad apply in one component can't touch another's state, even under concurrent access.
- **S3 versioning required**, checked weekly by automation — it's the difference between a 2.5-hour recovery and an unrecoverable one.

## Usage

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-<env>"
    key            = "<component>/terraform.tfstate"   # e.g. "eks/terraform.tfstate", "vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock-<env>"
    encrypt        = true
  }
}
```

Each component (`vpc`, `eks`, `rds`, `iam`) gets its own `key`, backed by the same lock table. The module boundary matters as much as the backend config: the VPC module is isolated from the EKS module because core networking changes rarely and compute changes often — separating them means a compute change can never accidentally touch networking, with or without a lock held.

## Lock table

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock-<env>"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## The lesson

Remote state without locking isn't a best-practice gap you get around to eventually — it's a production incident waiting for two people to work at once under deadline pressure, which is exactly when it's most likely to happen. The incident in staging made a stronger case for this than any architecture doc could have.

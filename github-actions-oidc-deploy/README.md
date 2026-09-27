# GitHub Actions → AWS via OIDC (zero stored credentials)

## The problem this removes

Long-lived AWS access keys stored as GitHub Secrets are a standing liability: they don't expire on their own, they get copied into forks and re-used across pipelines, and every rotation is a manual, easy-to-forget chore. At Luxottica, the CI/CD pipeline for Project VisionLens (GitHub Actions → Trivy scan → Helm promotion → EKS blue/green) authenticated to AWS with zero stored secrets at all, using OpenID Connect instead.

## How it works

GitHub's OIDC provider issues a short-lived token for the running workflow. AWS IAM trusts that provider and lets the workflow assume a role directly — no access key, no secret key, nothing to rotate, nothing sitting in GitHub Secrets that could leak.

### IAM trust policy (attached to the role the workflow assumes)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

The `sub` condition is the part worth not skipping — scope it to the exact repo and branch (or environment) allowed to assume the role. Without it, any workflow anywhere that can present a GitHub OIDC token for your org could assume it.

### Workflow

See [`.github/workflows/deploy.yml`](./.github/workflows/deploy.yml).

## Operational reason, not just a cleaner YAML file

- Eliminates the credential-rotation chore entirely — there's nothing long-lived to rotate.
- Removes a credential-exfiltration attack surface: a leaked workflow log or a compromised action can't leak a key that doesn't exist.
- Every assumed-role session is scoped to exactly the repo/branch that requested it, and shows up in CloudTrail as its own identity — better audit trail than a shared static key ever gave us.

## The tradeoff

Setup is more upfront work than pasting two secrets into GitHub — you need the OIDC provider registered once per AWS account and a trust policy scoped correctly per repo. That one-time cost is why teams skip it and use static keys instead. It's worth paying once.

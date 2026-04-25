# tsaws documentation

Public documentation for the Tailscale AWS Connector (`tsaws`).

`tsaws` runs inside your AWS VPC, discovers opted-in services from Route 53 private hosted zones, and registers them as Tailscale Services. Once deployed, private AWS endpoints (RDS, ElastiCache, internal ALBs, and others) become reachable by name over your tailnet with no manual Service configuration.

## Contents

| File | Description |
|---|---|
| [connector-aws.md](connector-aws.md) | Full deployment guide: prerequisites, Terraform/CloudFormation, eligibility, configuration reference, health checks, dry-run mode |
| [architecture.md](architecture.md) | Architecture overview: runtime model, reconciliation loop, package layout, design decisions, M1 implementation status |
| [aws-permissions.md](aws-permissions.md) | Minimum-privilege IAM policy for the Connector task role, with per-statement rationale |
| [policy-template.hujson](policy-template.hujson) | Starter Tailscale policy file snippet: tagOwners and grants for the Connector, Services, and admin portal |

## Quick start

1. Store your Tailscale OAuth client secret in AWS Secrets Manager.
2. Declare the Connector tags in your tailnet policy file (see [connector-aws.md — Prerequisites](connector-aws.md#prerequisites)).
3. Deploy via Terraform:

```hcl
module "tsaws" {
  source = "github.com/radioboxtv/tsaws//deploy/terraform"

  image_uri          = "ghcr.io/radioboxtv/tsaws:latest"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnets
  oauth_secret_arn   = aws_secretsmanager_secret.tsaws_oauth.arn
  hosted_zone_ids    = ["Z1234567890ABC"]
}
```

See [connector-aws.md](connector-aws.md) for the full guide.

## Source

The Connector binary and Terraform/CloudFormation templates are published from a private repository. Container images are published to `ghcr.io/radioboxtv/tsaws`.

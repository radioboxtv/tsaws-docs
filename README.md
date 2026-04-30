# tsaws

The Tailscale AWS Connector. A single-container Go daemon that runs inside your AWS VPC, discovers eligible services from Route 53 private hosted zones, and registers them as Tailscale Services. Once deployed, your private AWS endpoints (RDS, ElastiCache, internal ALBs, MSK brokers, and others) become reachable by name over your tailnet with no manual Service configuration.

Most of the connector's behavior is configured through your tailnet policy file via a node-attribute capability (`tsaws.com/config`), not through env vars on the container. That means you can change globs, tags, and grants without redeploying.

## Supported AWS managed services

Auto-detected port and metadata for:

- Amazon RDS (Postgres, MySQL/MariaDB, SQL Server, Oracle)
- Amazon Aurora (writer, reader, custom endpoints each registered as separate Services)
- Amazon ElastiCache (Redis, Memcached)
- Amazon MemoryDB
- Amazon MSK (plaintext, TLS, SASL/SCRAM)
- Amazon DocumentDB
- Amazon OpenSearch
- Amazon Neptune

Any other TCP endpoint reachable from the connector's subnet works too, with a `tailscale:port` AWS resource tag or a `default_port` in the cap.

## Prerequisites

- Tailscale account with Services enabled
- Tailscale OAuth client with scopes: `services` (write), `devices:core` (write), `auth_keys` (write). **Not** `policy_file` write.
- AWS VPC with at least one Route 53 private hosted zone attached
- Terraform 1.5+ (or AWS Console / CloudFormation)
- AWS CLI configured against the target account (for stashing the OAuth secret)

## Five steps to your first connection

1. **Add tagOwners, autoApprovers, and grants.** Paste [`policy-template.hujson`](policy-template.hujson) into your tailnet policy file. Customize the admin and per-environment groups.
2. **Add the `tsaws.com/config` node-attribute grant.** A `grants` block targeting `tag:tsaws` with `app: tsaws.com/config`. Inside, your tiered FQDN globs and a `custom_tags` map for per-environment tagging.
3. **Stash the Tailscale OAuth client secret in AWS Secrets Manager.**
4. **Deploy via Terraform.** Five inputs: `image_uri`, `vpc_id`, `private_subnet_ids`, `oauth_secret_arn`, `hosted_zone_ids`. No eligibility flags on the container.
5. **Verify.** Open `https://<connector-hostname>.<tailnet>.ts.net` to confirm the portal is up. Connect to a registered Service by name from any tailnet device.

The full walkthrough with example HuJSON, Terraform, and verification commands is in [getting-started.md](getting-started.md).

## Documentation

| File | Description |
|---|---|
| [getting-started.md](getting-started.md) | First-time setup walkthrough. Tailnet policy first, node-attr cap second, deploy last. ~10 minutes. |
| [how-tsaws-works.md](how-tsaws-works.md) | Conceptual: runtime model, discovery loop, eligibility, port derivation, identity, health, safety properties. |
| [configuration.md](configuration.md) | Full reference. Configuration surfaces, env vars, AWS resource tags, ACL patterns A/B/C, network topology, deployment, considerations. |
| [node-attr-config.md](node-attr-config.md) | `tsaws.com/config` capability schema, validation, audit events, example grants. |
| [api-reference.md](api-reference.md) | Portal HTTP endpoints, status `:8080` endpoint, webhook payload schema. |
| [aws-permissions.md](aws-permissions.md) | Minimum-privilege IAM policy for the connector task role with per-statement rationale. |
| [policy-template.hujson](policy-template.hujson) | Starter tailnet policy. tagOwners, autoApprovers, admin grants, non-admin per-environment grants, node-attr cap grant. |

## Source

The connector binary and Terraform/CloudFormation templates are published from a private repository. Container images are published to `ghcr.io/radioboxtv/tsaws`.

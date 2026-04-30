# Getting started with tsaws

Stand up a Tailscale AWS Connector and reach private AWS endpoints over the tailnet in about 10 minutes. This guide intentionally pushes most of the configuration into your tailnet policy file and the `tsaws.com/config` node-attribute capability, not the connector container's environment. That way you can change globs, tags, and grants without redeploying the connector.

If you want to understand what the connector is doing under the hood as you go, read [how-tsaws-works.md](how-tsaws-works.md). For the full configuration reference, see [configuration.md](configuration.md).

## Before you start

1. **Tailscale account with Services enabled.** Confirm the Services tab is visible in the admin console.
2. **Tailscale OAuth client** with scopes `services` (write), `devices:core` (write), and `auth_keys` (write). Create one at [login.tailscale.com/admin/settings/oauth](https://login.tailscale.com/admin/settings/oauth). Scope it to a single tag, typically `tag:tsaws`. Do **not** grant `policy_file` write — the connector is read-only against the tailnet policy file.
3. **AWS account** with at least one VPC and one Route 53 private hosted zone attached to that VPC. Records in the zone should resolve to the AWS endpoints you want to expose (RDS, ElastiCache, internal ALBs, and so on).
4. **An admin group** in your tailnet you control. The starter ACL grants this group portal access. `autogroup:admin` works as a minimum.
5. **Terraform 1.5+** (or the AWS Console / CloudFormation if you prefer that path).
6. **The AWS CLI**, configured against the target account. Required only for stashing the OAuth secret.

## Step 1: Author your tailnet policy file

Open your tailnet policy file in the admin console (Access controls). Paste in the contents of [`policy-template.hujson`](policy-template.hujson). You'll customize three things in this step: the admin group, the per-environment groups, and the FQDN globs.

The template has four blocks:

### tagOwners

Declares every tag the connector might apply. **Tailscale does not support wildcards in `tagOwners`** — every tag must be enumerated by its full name. The connector's greedy-with-fallback chain handles undeclared tags gracefully (it drops the unowned ones and falls back to a smaller set), so you can deploy first and add declarations after.

Substitute the AWS-specific values below to match your region, VPC name, subnet name, AZ, account ID, and cluster name. Tag values are lowercased; non-alphanumeric runs collapse to a single hyphen. If a VPC has no Name tag, the connector substitutes the VPC ID (e.g. `tag:aws-vpc-vpc-0a1b2c3d`); same pattern for subnets.

```jsonc
"tagOwners": {
  "tag:tsaws":              ["autogroup:admin"],
  "tag:tsaws-service":      ["tag:tsaws"],
  "tag:tsaws-admin-portal": ["tag:tsaws"],

  // AWS metadata tags applied to the connector node. Enumerate the
  // concrete tags the connector will produce for your deployment.
  // After first deploy, GET /api/policy-snippets in the portal returns
  // the exact list of tags the connector wanted to apply but found
  // undeclared, ready to paste in.
  "tag:aws-region-us-east-1":         ["tag:tsaws"],
  "tag:aws-vpc-prod-vpc":             ["tag:tsaws"],
  "tag:aws-subnet-prod-private-1a":   ["tag:tsaws"],
  "tag:aws-az-us-east-1a":            ["tag:tsaws"],
  "tag:aws-account-123456789012":     ["tag:tsaws"],
  "tag:aws-cluster-prod-cluster":     ["tag:tsaws"],
  "tag:aws-ecs-fargate":              ["tag:tsaws"],

  // Per-environment service tags applied via custom_tags in Step 2.
  "tag:env-prod":    ["tag:tsaws"],
  "tag:env-staging": ["tag:tsaws"],
}
```

If you don't want the AWS metadata tags at all, disable them with `TSAWS_CONNECTOR_REGION_TAG_ENABLED=false` and the matching variables for the dimensions you don't need (see [configuration.md](configuration.md#connector-node-tags)). The connector then only applies `tag:tsaws` itself.

### autoApprovers

The Connector advertises Services and itself as a Service Host. Tailnet's auto-approver mechanism is what permits a tagged device to advertise Services without an admin manually clicking through each one.

```jsonc
"autoApprovers": {
  "services": {
    "tag:tsaws-service": ["tag:tsaws"],
  },
}
```

### grants — admin pattern

Admins get full reach across every registered service plus portal access:

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws-service"],
  "ip":  ["*"],
},
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws-admin-portal"],
  "ip":  ["443"],
},
```

### grants — non-admin per-environment pattern

Non-admin users get scoped reach. The example splits prod and staging by environment tag. Pair this with `custom_tags` in Step 2: the connector tags each Service `tag:env-prod` or `tag:env-staging` based on FQDN, and these grants then control which group can reach which environment.

```jsonc
{
  "src": ["group:devs"],
  "dst": ["tag:env-staging"],
  "ip":  ["*"],
},
{
  "src": ["group:sre"],
  "dst": ["tag:env-prod", "tag:env-staging"],
  "ip":  ["*"],
},
```

Substitute your own group names. If you don't yet have groups defined, declare them in the same policy file:

```jsonc
"groups": {
  "group:devs": ["alice@example.com", "bob@example.com"],
  "group:sre":  ["carol@example.com"],
}
```

For per-Service grants (one tag per backend rather than per environment), see [configuration.md](configuration.md#acl-patterns) — pattern C.

## Step 2: Add the node-attribute config grant

This is where the bulk of the connector's behavior lives. The `tsaws.com/config` capability is delivered through a `grants` block in the same policy file. The connector reads it on every NetMap push, so you can tighten globs, swap tags, or change the refresh rate without redeploying.

Add this block to your `grants` array:

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "domains": [
        "*.prod.example.internal",
        "*.staging.example.internal",
      ],
      "custom_tags": {
        "*.prod.example.internal":    ["tag:env-prod"],
        "*.staging.example.internal": ["tag:env-staging"],
      },
      "port_blocklist": [22, 3389],
    }],
  },
},
```

What each field does:

| Field | Effect |
|---|---|
| `domains` | FQDN allowlist. A record's bare FQDN must match at least one glob to be eligible. Glob syntax: `?`, `*`, `[abc]`. |
| `custom_tags` | Per-FQDN-glob → service tag list. The connector applies these tags during the next discovery cycle. The map drives the per-environment ACL pattern from Step 1. |
| `port_blocklist` | Ports the connector refuses to register. The connector default is empty; this example blocks 22 (SSH) and 3389 (RDP) defensively. |

The full schema (ten fields including `refresh_rate`, `max_services`, `default_port`, `service_tag`, `connector_tag`, `portal_tag`, `discover_all_from_zone`) is in [node-attr-config.md](node-attr-config.md). Add what you need; absent fields fall back to the connector's startup configuration.

Substitute `example.internal` with your actual zone root. Tiered globs like `*.prod.<zone>` and `*.staging.<zone>` are the recommended starting point: they match the per-environment ACL pattern without you having to tag individual AWS resources.

> **Tip.** You can edit the cap value at any time in the admin console and the connector will pick up the change within one discovery cycle (default 5 minutes; tighten to 30s while you iterate by adding `"refresh_rate": "30s"` to the cap). No restart required.

## Step 3: Stash the OAuth secret in AWS Secrets Manager

```bash
aws secretsmanager create-secret \
  --name tsaws/oauth-secret \
  --secret-string "tskey-client-..." \
  --region us-east-1
```

Note the returned ARN; you'll pass it to the Terraform module in the next step. The connector retrieves the secret at startup via the ECS secrets injection mechanism: it never ends up in the task definition or environment variable plaintext.

## Step 4: Deploy the connector

The Terraform module in `deploy/terraform/` creates an ECS Fargate task, security group, IAM task and execution roles, and a CloudWatch log group. The minimum viable input is five values: container image, VPC, subnet IDs, secret ARN, and zone IDs.

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

That's it. No `TSAWS_GLOBS`, no `TSAWS_SERVICE_TAG`, no eligibility flags. Everything the cap covers in Step 2 is omitted from the deployment. The connector reads it from the policy file at runtime.

Run `terraform apply`. The container takes ~30 seconds to start, register with the tailnet, and complete its first discovery cycle.

If you prefer CloudFormation, use the template at `deploy/cloudformation/tsaws.yaml` with the same five inputs.

## Step 5: Verify

The connector exposes an admin portal as a Tailscale Service on port 443. Open it in any browser on a tailnet device:

```
https://<connector-hostname>.<tailnet>.ts.net
```

The hostname is auto-derived; check the Tailscale admin console for the exact name (it'll look like `ta-1a-prod-vpc-us-east-1`, with the most-unique per-replica segment first). If the portal loads, the connector is alive, authenticated, and serving over the tailnet.

In the portal:

- The **Services** panel lists every FQDN the connector discovered, with its current state (`registered`, `degraded`, `ineligible`, etc.)
- The **Discovery** panel shows per-zone breakdowns and per-record decisions, so you can confirm your `domains` globs match what you expected
- The **Tags** panel shows which connector tags landed and which are missing from `tagOwners` — useful if a metadata tag isn't getting applied
- The **Events** panel shows the last 500 audit events including any cap validation failures (`NodeAttrFetchFailed`)

Once a Service shows `registered`, connect to it by name. From any tailnet device:

```bash
psql -h api-prod.<tailnet>.ts.net -U postgres
redis-cli -h cache-staging.<tailnet>.ts.net
```

The Service name is the FQDN with the zone root stripped, dots replaced with hyphens, and lowercased.

## Where next

- **[configuration.md](configuration.md)** — full env var, AWS resource tag, and node-attr cap reference. Patterns A/B/C for ACLs (admin broad, per-environment, per-Service).
- **[node-attr-config.md](node-attr-config.md)** — every cap field with validation rules and example grants.
- **[how-tsaws-works.md](how-tsaws-works.md)** — runtime model, discovery loop, identity model, safety properties.
- **[api-reference.md](api-reference.md)** — every portal HTTP endpoint.
- **[aws-permissions.md](aws-permissions.md)** — IAM policy for the connector task role with per-statement rationale.

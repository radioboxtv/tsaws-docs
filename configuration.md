# Configuration reference

The full set of knobs and considerations for tsaws. For a 10-minute first-time setup, see [getting-started.md](getting-started.md). For runtime model and discovery loop behavior, see [how-tsaws-works.md](how-tsaws-works.md).

This document is structured around the four configuration surfaces and how they layer. If you're looking for a specific variable, the [environment variable reference](#environment-variable-reference) is alphabetical-ish by section.

## Configuration surfaces

The connector reads configuration from four places. They layer:

1. **Environment variables** — read once at process startup. Sets the floor for everything except per-Service overrides. Restart required to change.
2. **Node-attribute capability `tsaws.com/config`** — pushed via NetMap, applied each discovery cycle. Covers ten discovery-loop tunables and tag overrides. Overrides matching env vars while the cap is active.
3. **Portal `POST /api/config/runtime`** — operator override for live experiments. Same field set as the cap. Reverts on the next NetMap push when a cap is also active.
4. **AWS resource tags (`tailscale:*`)** — per-resource overrides. Read each cycle from the Resource Groups Tagging API.

The tailnet policy file is independent. The connector reads it for bootstrap validation only and never writes.

### Precedence and delegation

For a single FQDN, configuration resolves like this:

```
AWS resource tag (per-resource)
   │  overrides
   ▼
node-attr cap "custom_tags" / portal POST   (per-glob or per-cap)
   │  overrides
   ▼
node-attr cap top-level fields              (connector-wide)
   │  overrides
   ▼
environment variables                       (startup floor)
```

For example, if `TSAWS_DEFAULT_PORT=5432` is in the environment and the cap also sets `default_port: 6432`, the connector uses 6432 until the cap clears. If a specific record carries `tailscale:port=11211`, that record uses 11211 regardless.

The portal `POST /api/config/runtime` writes through to the same in-memory overrides as the cap. If a cap is active and you change a field via the portal, the next NetMap push will revert it. The portal Config page shows a banner when a cap is active.

Per-Service health config (mode, drain state) is its own layer, persisted to `TSAWS_HEALTH_CONFIG_FILE` when set. Operator-set health modes are never overwritten by AutoDetect.

## Required environment variables

Two variables are required. Everything else has a default.

| Variable | Description |
|---|---|
| `TS_CLIENT_SECRET` | Tailscale OAuth client secret. Inject via Secrets Manager or `TSAWS_CLIENT_SECRET_SOURCE`, not plaintext. |
| `TSAWS_HOSTED_ZONE_IDS` | Comma-separated Route 53 private hosted zone IDs (e.g. `Z1234567890ABC,Z0987654321DEF`). |

When using `TSAWS_CLIENT_SECRET_SOURCE`, `TS_CLIENT_SECRET` is not required:

```
TSAWS_CLIENT_SECRET_SOURCE=aws-secretsmanager://tsaws/oauth-secret
```

Supported source schemes: `env://VAR`, `aws-secretsmanager://<id>`, `aws-ssm://<param>`, `file:///path`.

## Node-attribute cap fields

The cap value is a JSON object delivered via the `tsaws.com/config` capability in your tailnet policy file. All fields are optional; absent fields leave the existing override untouched.

| Field | Type | Effect |
|---|---|---|
| `refresh_rate` | string (Go duration) | Discovery cycle cadence. Min `1s`. |
| `max_services` | int | Service cap. `0` = unlimited. |
| `default_port` | uint16 | Last-resort port. `0` = no fallback. |
| `port_blocklist` | uint16[] | Ports the connector refuses to register. Each `1..65535`, no duplicates. |
| `domains` | string[] | FQDN allowlist (`path.Match` globs). |
| `discover_all_from_zone` | bool | When `true`, every record from configured zones passes the filter. |
| `service_tag` | string | Override the entire fallback chain for new Service upserts. Must start with `tag:`. |
| `custom_tags` | `map[string][]string` | Per-FQDN-glob tag list. Each glob compiled via `path.Match`; each tag must start with `tag:`. |
| `connector_tag` | string | Connector node's own tsnet identity tag. Triggers a tsnet host rebuild on change. |
| `portal_tag` | string | Tag on the admin portal Service. Triggers a tsnet host rebuild on change. |

Validation is skip-not-fail: invalid fields are dropped, valid sibling fields land, and a `NodeAttrFetchFailed` audit event records the rejected names. Full schema, validation details, and example grants are in [node-attr-config.md](node-attr-config.md).

## Eligibility

A record passes the eligibility filter when **at least one** opt-in rule matches. Rules combine as logical OR.

### Domains glob (recommended primary)

`domains` (cap) or `TSAWS_GLOBS` (env): comma-separated FQDN glob patterns. The bare FQDN must match at least one. Glob syntax is `path.Match`: `?`, `*`, `[abc]`, `[!abc]`.

```
TSAWS_GLOBS=*.prod.company.internal,db.internal
```

The empty list and `["*"]` are very different: empty disables this filter (no records pass on globs); `["*"]` matches every record (subject to other filters).

### Zone-wide opt-in

`discover_all_from_zone` (cap) or `TSAWS_ZONE_OPT_IN=true`: every record in an allowlisted zone is eligible regardless of globs or tags. Useful when an operator dedicates a hosted zone to tsaws.

### Per-resource AWS tag

Apply the tag `tailscale:register=true` to an AWS resource. The connector discovers this via the Resource Groups Tagging API and marks the corresponding Route 53 record eligible, independent of glob and zone settings.

### Port blocklist

`port_blocklist` (cap) or `TSAWS_PORT_BLOCKLIST` (env). Empty by default: every port that gets through eligibility, port derivation, and the cap is registered. Set to a comma-separated list (`22,3389`) to refuse registration for specific ports at the connector layer.

The blocklist is defense-in-depth, not the primary access control. Tailnet ACL grants gate access regardless of which port is registered.

## ACL patterns

Three patterns ship in the [`policy-template.hujson`](policy-template.hujson) starter. Pick the one that matches your access model. They compose: admin grants are usually a superset of non-admin scoping.

### Pattern A: admin broad reach

Admins get reach to every registered Service plus the portal.

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

Use this for operators who manage the connector itself or need backstop access during incidents. The `tag:tsaws-service` tag is the connector's default service tag; if you've overridden `service_tag`, substitute your value.

### Pattern B: per-environment via custom_tags

The connector applies per-FQDN-glob tags via the cap; ACLs grant by environment tag rather than per-Service. Best for fleets where access patterns split cleanly by environment (prod / staging / dev) or by compliance boundary (PII vs. non-PII).

In your `tsaws.com/config` cap:

```jsonc
"custom_tags": {
  "*.prod.example.internal":    ["tag:env-prod", "tag:pii-allowed"],
  "*.staging.example.internal": ["tag:env-staging"],
  "*.dev.example.internal":     ["tag:env-dev"],
}
```

In your `grants` array:

```jsonc
{
  "src": ["group:devs"],
  "dst": ["tag:env-staging", "tag:env-dev"],
  "ip":  ["*"],
},
{
  "src": ["group:sre"],
  "dst": ["tag:env-prod"],
  "ip":  ["*"],
},
{
  "src": ["group:audit"],
  "dst": ["tag:pii-allowed"],
  "ip":  ["*"],
},
```

A FQDN matched by multiple globs collects the union of all matching tag lists. Sorting and deduplication keep the upsert fingerprint stable across NetMap pushes.

Don't forget to declare every applied tag in `tagOwners`:

```jsonc
"tag:env-prod":     ["tag:tsaws"],
"tag:env-staging":  ["tag:tsaws"],
"tag:env-dev":      ["tag:tsaws"],
"tag:pii-allowed":  ["tag:tsaws"],
```

### Pattern C: per-Service tags via AWS resource tag

For workflows where operators want one tag per backend rather than per environment, apply the `tailscale:tags` AWS resource tag directly. The connector reads it during discovery and passes the value through to the Service identity tag set, replacing the cap-derived tag chain for that resource.

On the AWS resource (RDS instance, ElastiCache cluster, etc.):

```
tailscale:tags = tag:postgres-customers
```

In your policy file `tagOwners`:

```jsonc
"tag:postgres-customers": ["tag:tsaws"],
"tag:redis-cache":        ["tag:tsaws"],
```

In your `grants` array:

```jsonc
{
  "src": ["group:customer-success"],
  "dst": ["tag:postgres-customers"],
  "ip":  ["5432"],
},
{
  "src": ["group:cache-readers"],
  "dst": ["tag:redis-cache"],
  "ip":  ["6379"],
},
```

Use when the access pattern is "one team owns one backend" rather than "many teams share many environments". Trade-off: every new backend means another `tagOwners` entry, another `grants` block, and another AWS tag — but the granularity is at the Service level.

### Approving Service connections

For tailnet users to reach a registered Service, an `autoApprovers.services` entry must exist for the Service's tag:

```jsonc
"autoApprovers": {
  "services": {
    "tag:tsaws-service": ["tag:tsaws"],
  },
}
```

This authorizes any device carrying `tag:tsaws` to advertise Services under `tag:tsaws-service`. Adjust the tag names to match your service tag and connector tag values.

## Connector node tags

The connector's tsnet node is tagged at startup with metadata inferred from the AWS runtime environment. These tags appear on the Connector's device entry in the Tailscale admin console and can be used in tailnet ACL policy to scope access to or from the connector itself.

Baseline AWS metadata tags are applied to the Connector node only, not to individual Services. Service identity stays minimal; use `custom_tags` (pattern B) or `tailscale:tags` (pattern C) for per-Service segmentation.

All tag gates default to enabled. Set a variable to `false` to suppress that tag.

| Variable | Default | Tag applied |
|---|---|---|
| `TSAWS_CONNECTOR_REGION_TAG_ENABLED` | `true` | `tag:aws-region-<slug>` |
| `TSAWS_CONNECTOR_VPC_TAG_ENABLED` | `true` | `tag:aws-vpc-<slug>` (VPC Name tag, or VPC ID when no Name tag) |
| `TSAWS_CONNECTOR_SUBNET_TAG_ENABLED` | `true` | `tag:aws-subnet-<slug>` |
| `TSAWS_CONNECTOR_AZ_TAG_ENABLED` | `true` | `tag:aws-az-<slug>` |
| `TSAWS_CONNECTOR_ACCOUNT_TAG_ENABLED` | `true` | `tag:aws-account-<id>` |
| `TSAWS_CONNECTOR_CLUSTER_TAG_ENABLED` | `true` | `tag:aws-cluster-<name>` (ECS or EKS cluster name) |
| `TSAWS_CONNECTOR_DEPLOYMENT_TAG_ENABLED` | `true` | `tag:aws-ecs-fargate`, `tag:aws-ecs-ec2`, `tag:aws-ec2`, `tag:aws-eks`, or `tag:aws-lambda` |
| `TSAWS_CONNECTOR_CUSTOM_TAGS` | none | Comma-separated additional tags applied unconditionally |
| `TSAWS_EKS_CLUSTER_NAME` | none | Required for `tag:aws-cluster-<name>` on EKS (not derivable from IMDS) |

Tag values are lowercased and non-alphanumeric runs collapse to a single hyphen. Region, AZ, and account come from EC2 IMDSv2. VPC and subnet names come from `ec2:DescribeVpcs` and `ec2:DescribeSubnets`. ECS cluster comes from the ECS task metadata endpoint. Deployment model comes from `AWS_EXECUTION_ENV`.

Tag application uses a greedy-with-fallback chain: if the full set is undeclared in `tagOwners`, the connector falls back to a smaller set, ultimately to `TSAWS_CONNECTOR_TAG` alone. The portal `/api/tags` and `/api/policy-snippets` endpoints surface what was applied and which declarations are missing.

All tags applied to the Connector node must be declared in `tagOwners` in your tailnet policy file before the node joins. **Tailscale does not support wildcards in `tagOwners`**: enumerate concrete tag names rather than patterns like `tag:aws-region-*`. The connector's greedy-with-fallback chain handles missing declarations gracefully, so deploy first and use `/api/policy-snippets` to discover the exact list to add.

## Service identity tags

Three layers compose the tag set the connector applies to a registered Service:

1. **Service-level tag** from `service_tag` (cap) or the OAuth-derived fallback chain. One tag, applied to every Service.
2. **Per-FQDN tags** from `custom_tags` (cap). Multiple tags, applied to Services whose FQDN matches a glob.
3. **Per-resource override** from `tailscale:tags` AWS resource tag. Replaces the entire computed set for that one resource.

The upserted set is `{service_tag-or-fallback} ∪ {custom_tags-matches}` — unless `tailscale:tags` is present on the resource, in which case it wins.

Existing Services keep their old tags until the connector re-PUTs them, which happens automatically on the next discovery cycle when `EligibilityInputsChanged` fires (any of `domains`, `discover_all_from_zone`, `default_port`, `port_blocklist`, `service_tag`, `custom_tags`).

## Network topology

Three deployable topologies. Pick the one that matches your egress posture.

### Option 1: NAT gateway

The standard deployment places the Connector in a private subnet with a NAT gateway providing outbound internet access. No inbound security group rules required.

```
Internet
    │
[NAT Gateway]
    │
[Private subnet]
    └── ECS task: tsaws  (outbound only)
```

Simplest. NAT gateway costs approximately $32/month plus data transfer.

### Option 2: Peer Relay in a public subnet

A Tailscale Peer Relay node in a public subnet acts as a relay hop for the connector. The connector stays in a fully private subnet with no NAT gateway.

```
Internet
    │
[Peer Relay node, public subnet, EIP]
    │  (VPC-internal routing, no NAT)
[Private subnet]
    └── ECS task: tsaws  (no internet egress required)
```

Eliminates the NAT gateway entirely. EIP for the Peer Relay is approximately $3.60/month — significantly less than a NAT gateway under sustained load.

### Option 3: Peer Relay as a sidecar

A Peer Relay container runs in the same ECS task as the connector. Security group opens a single inbound UDP port (default 41641) at the NAT gateway, replacing full egress with one forwarded port.

```
Internet
    │  (UDP 41641 only)
[NAT Gateway, single port forward]
    │
[Private subnet]
    └── ECS task: tsaws + peer-relay sidecar
```

Useful in environments that restrict egress but allow specific inbound ports. Track configuration in [issue #88](https://github.com/radioboxtv/tsaws/issues/88).

## Health checks

The connector runs a background health check for each advertised `(service, port)` pair. After `TSAWS_HEALTH_UNHEALTHY_THRESHOLD` consecutive failures (default 3), it withdraws the host advertisement and transitions the Service to `degraded`. After `TSAWS_HEALTH_HEALTHY_THRESHOLD` consecutive successes (default 2), it re-advertises.

### Modes

`TSAWS_HEALTH_MODE` selects the global default:

- `l4_tcp` — TCP connect within `TSAWS_HEALTH_TIMEOUT`. Default.
- `l7_http` — HTTP GET; passes on any 2xx, 3xx, or 4xx response. Path defaults to `/`; override with `tailscale:health-path`.
- `none` — Advertisement is sticky.
- L4.5 protocol probes — `redis` (RESP `PING`), `postgres` (StartupMessage), `mysql` (handshake read), `kafka` (ApiVersions), `mongodb` (`isMaster` OP_QUERY), `memcached` (`version`), `opensearch` (HTTPS `/_cluster/health`).
- `auto` — FQDN-based selection (e.g. `.cache.amazonaws.com` → `redis`, `.rds.amazonaws.com` → `postgres`).

### Auto-promotion knobs (default on)

- `TSAWS_HEALTH_AUTO_L7=true` — Targets on ports 80, 443, 8080, 8443 are run through the HTTP detector and upgraded from L4 to L7 only when the backend actually speaks HTTP. Backends that don't (Postgres-over-TLS, custom binary protocols on 443) stay on L4.
- `TSAWS_HEALTH_AUTO_PROTOCOL=true` — When `TSAWS_HEALTH_MODE=l4_tcp`, the engine selects an L4.5 probe per service from FQDN heuristics.
- `TSAWS_HEALTH_AUTO_DETECT=true` — At first registration of a service, runs the multi-protocol detection sweep once and applies the recommended mode via the per-service health config store. Operator-set modes are never overwritten.

### Per-service overrides

- AWS resource tag `tailscale:health-mode=...` (and `tailscale:health-path=...` for L7).
- Portal `PUT /api/services/{name}/health-check` with a JSON body.
- Live changes are pushed to running probes without re-registration.

Per-service config persists across restarts when `TSAWS_HEALTH_CONFIG_FILE` is set; otherwise overrides are in memory only.

## Dry-run mode

Run the connector with `TSAWS_DRY_RUN=true` (or `--dry-run`) to preview registrations without writing to Tailscale or AWS.

Output is structured JSON with three sections:

```json
{
  "Summary": { "WouldCreate": 3, "WouldUpdate": 1, "WouldSkip": 12, "Orphans": 0, "Failed": 0 },
  "Desired": [
    { "fqdn": "api.prod.internal.", "service_name": "svc:api-prod", "ports": ["tcp:443"], "action": "would-create" },
    { "fqdn": "db.prod.internal.",  "service_name": "svc:db-prod",  "ports": ["tcp:5432"], "action": "would-update", "changes": ["ports: [tcp:3306] -> [tcp:5432]"] }
  ],
  "Orphans": [
    { "service_name": "svc:old-service", "action": "orphan" }
  ]
}
```

Actions: `would-create`, `would-update`, `would-skip`, `orphan`.

If `TS_CLIENT_SECRET` is unset, dry-run still runs full discovery but skips the comparison against live Tailscale state.

## UDP proxy

When `TSAWS_UDP_ENABLED=true`, the connector forwards UDP datagrams in addition to TCP for every advertised `(service, port)` pair. Disabled by default.

For each advertised pair, the connector calls `tsnet.ListenPacket("udp", addr)` on the node's tailnet IPv4 address. The first packet from a new client address creates a flow (a connected UDP socket to the backend); subsequent packets reuse it. Flows idle for longer than `TSAWS_UDP_IDLE_TIMEOUT` (default 60s) are torn down. Byte counts are tracked in the same traffic counter as TCP.

**Protocol selection is global.** When UDP is enabled, every discovered service gets UDP forwarding on its configured port. There is no per-service or per-protocol selector. Use this only when all proxied services legitimately need UDP (a tailnet exclusively proxying DNS resolvers or syslog endpoints).

The connector registers UDP services with the `do-not-validate` sentinel for ports because the Tailscale Services API UDP port format is unconfirmed upstream. The local UDP proxy works regardless; the API may reject UDP port entries until officially supported.

## High-availability deployment

Run two independent connector replicas, one per Availability Zone. Both register the same Services idempotently; the Tailscale control plane tracks all active hosts and routes connections to a healthy one.

Each replica needs a unique `name_prefix` to avoid IAM and resource name collisions. All other inputs are shared.

```hcl
module "tsaws_az1" {
  source = "github.com/radioboxtv/tsaws//deploy/terraform"

  name_prefix        = "tsaws-az1"
  image_uri          = "ghcr.io/radioboxtv/tsaws:latest"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = [module.vpc.private_subnets[0]]
  oauth_secret_arn   = aws_secretsmanager_secret.tsaws_oauth.arn
  hosted_zone_ids    = ["Z1234567890ABC"]
}

module "tsaws_az2" {
  source = "github.com/radioboxtv/tsaws//deploy/terraform"

  name_prefix        = "tsaws-az2"
  image_uri          = "ghcr.io/radioboxtv/tsaws:latest"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = [module.vpc.private_subnets[1]]
  oauth_secret_arn   = aws_secretsmanager_secret.tsaws_oauth.arn
  hosted_zone_ids    = ["Z1234567890ABC"]
}
```

Concurrent upserts are safe: Service registration uses GET-before-PUT with ETag retries.

Don't pass the same `ecs_cluster_arn` to both modules if the cluster ARN is computed by Terraform in the same root module; Terraform cannot evaluate the `count` conditional in the module before apply. Either let each module create its own cluster (default when `ecs_cluster_arn` is omitted) or pass a pre-existing cluster ARN as a literal string.

## Deployment

### Terraform (recommended)

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

The `tailnet` variable defaults to `-`, which resolves to the tailnet associated with the OAuth client.

### CloudFormation

Use the template at `deploy/cloudformation/tsaws.yaml`. It accepts the same parameters as the Terraform module.

### Manual ECS Fargate

1. Pull the connector image from `ghcr.io/radioboxtv/tsaws:latest`.
2. Create an ECS task definition with the env vars from [Environment variable reference](#environment-variable-reference). Inject `TS_CLIENT_SECRET` as a Secrets Manager reference, not plaintext.
3. Run the task in a private subnet with NAT gateway egress (or one of the Peer Relay topologies). No inbound rules required.
4. Attach the IAM task role with the permissions in [aws-permissions.md](aws-permissions.md).

## AWS resource tags

Apply these tags directly to an AWS resource to override connector behavior for the corresponding Service.

| Tag | Description |
|---|---|
| `tailscale:register=true` | Mark this resource eligible for registration (alternative to glob or zone opt-in). |
| `tailscale:service-name=<name>` | Override the derived Service name. |
| `tailscale:ports=<port>[,<port>]` | Override the derived port list. Comma-separated TCP ports. |
| `tailscale:target=<host:port>` | Override the backend proxy target. |
| `tailscale:description=<text>` | Override the Service description. |
| `tailscale:tags=<tag>[,<tag>]` | Override the Tailscale ACL tags applied to this Service. Wins over `service_tag` and `custom_tags`. |
| `tailscale:health-mode=<mode>` | Per-Service health mode: `l4_tcp`, `l7_http`, `none`, `redis`, `postgres`, `mysql`, `kafka`, `mongodb`, `memcached`, `opensearch`, or `auto`. |
| `tailscale:health-path=<path>` | HTTP path for L7 health checks. Default `/`. |

## Environment variable reference

All configuration via environment variables. Variables are read once at startup; restart to change.

A subset of these tunables is also live-tunable via the node-attribute cap (`refresh_rate`/`TSAWS_RECONCILE_INTERVAL`, `max_services`/`TSAWS_MAX_SERVICES`, `default_port`/`TSAWS_DEFAULT_PORT`, `port_blocklist`/`TSAWS_PORT_BLOCKLIST`, `domains`/`TSAWS_GLOBS`, `discover_all_from_zone`/`TSAWS_ZONE_OPT_IN`, `service_tag`/`TSAWS_SERVICE_TAG`, plus `connector_tag` and `portal_tag` and `custom_tags`). See [node-attr-config.md](node-attr-config.md).

### Required

| Variable | Description |
|---|---|
| `TS_CLIENT_SECRET` | Tailscale OAuth client secret. Mutually exclusive with `TSAWS_CLIENT_SECRET_SOURCE`. |
| `TSAWS_HOSTED_ZONE_IDS` | Comma-separated Route 53 PHZ IDs. |

### Secrets

| Variable | Description |
|---|---|
| `TSAWS_CLIENT_SECRET_SOURCE` | URI for the OAuth secret: `env://TS_CLIENT_SECRET`, `aws-secretsmanager://<secret-id>`, `aws-ssm://<param>`, `file:///path`. |

### Tailscale identity

| Variable | Default | Description |
|---|---|---|
| `TSAWS_CONNECTOR_TAG` | `tag:tsaws` | Connector tsnet node ACL tag. |
| `TSAWS_CONNECTOR_HOSTNAME` | auto | Verbatim hostname for the tsnet node. When unset, the connector derives `ta-<suffix>-<base>-<region>`, where `suffix` is the most-unique per-replica identifier (subnet Name → SubnetID short → AZ short → ECS task hash → EKS pod hash) and `base` is the broader VPC context (VPC Name → first hosted zone → VPC ID short). Two replicas in the same VPC but different subnets or AZs land on distinct hostnames; tsnet's automatic `-2`/`-3` suffix is only the last resort when no per-replica dimension can be inferred. Clamped to 63 chars (DNS label max). The hostname is also used as the `owner=` prefix in Service comments for the conflict tie-break, so changing it via the portal triggers a full reconnect (and deletes the orphan portal Service from the previous hostname). |
| `TSAWS_TAILNET` | `-` | Tailnet name (`-` = default for the OAuth client). |
| `TSAWS_SERVICE_TAG` | none | Single ACL tag applied to every registered Service. Empty = falls back to `TSAWS_CONNECTOR_TAG`. |

### Discovery and eligibility

| Variable | Default | Description |
|---|---|---|
| `TSAWS_GLOBS` | `*` | Comma-separated DNS glob patterns. Empty disables glob matching. |
| `TSAWS_ZONE_OPT_IN` | `false` | When `true`, all records in allowlisted zones are eligible. |
| `TSAWS_DEFAULT_PORT` | none | Fallback TCP port when no other source derives one. |
| `TSAWS_PORT_BLOCKLIST` | empty | Comma-separated ports to never register. |
| `TSAWS_MAX_SERVICES` | `1000` | Per-cycle Service cap. Records beyond the cap enter `cap-exceeded`. |

### Operation

| Variable | Default | Description |
|---|---|---|
| `TSAWS_RECONCILE_INTERVAL` | `5m` | Discovery loop cadence. Min `1s`. |
| `TSAWS_STATUS_ADDR` | `:8080` | Local in-VPC status HTTP endpoint. |
| `TSAWS_DRY_RUN` | `false` | Run full discovery; make no writes. |
| `TSAWS_SHUTDOWN_DRAIN` | `30s` | Maximum time to wait for in-flight proxy connections after SIGTERM. |
| `TSAWS_SHUTDOWN_KEEP_SESSIONS` | `false` | When `true`, skip force-close of in-flight connections on shutdown. |
| `TSAWS_METRICS_ADDR` | none | Prometheus endpoint (e.g. `:9090`). Empty disables. |
| `TSAWS_WEBHOOK_URL` | none | Outbound webhook URL: every event POSTed as JSON. |
| `TSAWS_WEBHOOK_SECRET` | none | HMAC-SHA256 secret for `X-Tsaws-Signature`. |

### Health checks

| Variable | Default | Description |
|---|---|---|
| `TSAWS_HEALTH_MODE` | `l4_tcp` | Global health mode (see [Health checks](#health-checks)). |
| `TSAWS_HEALTH_TIMEOUT` | `5s` | Per-probe timeout. |
| `TSAWS_HEALTH_INTERVAL` | `30s` | Interval between probes. |
| `TSAWS_HEALTH_UNHEALTHY_THRESHOLD` | `3` | Consecutive failures before withdrawing advertisement. |
| `TSAWS_HEALTH_HEALTHY_THRESHOLD` | `2` | Consecutive successes before re-advertising. |
| `TSAWS_HEALTH_AUTO_L7` | `true` | Upgrade 80/443/8080/8443 to L7 when the backend actually speaks HTTP. |
| `TSAWS_HEALTH_AUTO_PROTOCOL` | `true` | When mode is `l4_tcp`, select an L4.5 probe from FQDN heuristics. |
| `TSAWS_HEALTH_AUTO_DETECT` | `true` | Run multi-protocol detection at first registration; apply recommended mode. |
| `TSAWS_HEALTH_CONFIG_FILE` | none | Path for persisting per-Service health overrides across restarts. |

### UDP

| Variable | Default | Description |
|---|---|---|
| `TSAWS_UDP_ENABLED` | `false` | Forward UDP datagrams for every advertised pair (global flag). |
| `TSAWS_UDP_IDLE_TIMEOUT` | `60s` | Per-flow idle timeout. |

### Connector node tags

| Variable | Default | Description |
|---|---|---|
| `TSAWS_CONNECTOR_REGION_TAG_ENABLED` | `true` | Apply `tag:aws-region-<slug>`. |
| `TSAWS_CONNECTOR_VPC_TAG_ENABLED` | `true` | Apply `tag:aws-vpc-<slug>`. |
| `TSAWS_CONNECTOR_SUBNET_TAG_ENABLED` | `true` | Apply `tag:aws-subnet-<slug>`. |
| `TSAWS_CONNECTOR_AZ_TAG_ENABLED` | `true` | Apply `tag:aws-az-<slug>`. |
| `TSAWS_CONNECTOR_ACCOUNT_TAG_ENABLED` | `true` | Apply `tag:aws-account-<id>`. |
| `TSAWS_CONNECTOR_CLUSTER_TAG_ENABLED` | `true` | Apply `tag:aws-cluster-<name>`. |
| `TSAWS_CONNECTOR_DEPLOYMENT_TAG_ENABLED` | `true` | Apply runtime deployment tag. |
| `TSAWS_CONNECTOR_CUSTOM_TAGS` | none | Comma-separated additional tags. |
| `TSAWS_EKS_CLUSTER_NAME` | none | EKS cluster name (required for `tag:aws-cluster-<name>` on EKS). |

### Admin portal

| Variable | Default | Description |
|---|---|---|
| `TSAWS_PORTAL_ENABLED` | `true` | Register the admin portal Service. |
| `TSAWS_PORTAL_TAG` | `tag:tsaws-admin-portal` | Portal Service ACL tag. |
| `TSAWS_PORTAL_VPC_ID` | none | Legacy. Retained for backward compatibility; safe to leave unset. |

### Rate limiting

| Variable | Default | Description |
|---|---|---|
| `TSAWS_RATELIMIT_TS_RPS` | `10` | Tailscale API requests per second. |
| `TSAWS_RATELIMIT_TS_BURST` | `20` | Tailscale API burst. |
| `TSAWS_RATELIMIT_AWS_RPS` | `50` | AWS API requests per second. |
| `TSAWS_RATELIMIT_AWS_BURST` | `100` | AWS API burst. |
| `TSAWS_RATELIMIT_ROUTE53_RPS` | `5` | Route 53 API requests per second. |
| `TSAWS_RATELIMIT_ROUTE53_BURST` | `10` | Route 53 API burst. |

### AWS context

The connector reads region from the AWS SDK chain (`AWS_REGION`, then `AWS_DEFAULT_REGION`, then EC2 instance metadata). Set explicitly only when running outside AWS or pointing at a non-default region.

## Considerations

### Port blocklist trade-off

`TSAWS_PORT_BLOCKLIST` is empty by default. The reasoning: tailnet ACL grants are the primary access control. A connector-side blocklist is only useful as defense-in-depth against a misconfigured AWS resource tag. Setting `22,3389` is a reasonable hardening default if you never want SSH or RDP advertised over the tailnet, but it is not required. Don't rely on it for security; rely on grants.

### UDP global flag implications

`TSAWS_UDP_ENABLED=true` is global. Every advertised Service gets UDP forwarding on its TCP port. Mixed fleets where some backends should be TCP-only and others UDP cannot be selectively configured today. Use only when all proxied services legitimately need UDP.

### Policy file and the connector

The connector reads the tailnet policy file at startup for bootstrap validation (confirming `autoApprovers.services` and `tagOwners` cover the tags it intends to use). It never writes. Operators author `autoApprovers.services` once; subsequent changes go through your normal policy-file change-management flow.

If your existing OAuth client carries `policy_file` write scope, the connector will warn at startup and continue with the unused scope; reduce it on next rotation.

### Service retention vs. deletion

When a source DNS record disappears, the connector withdraws the host advertisement but retains the Tailscale Service object. Auto-deletion is opt-in and disabled by default. The Service shows in `orphaned` state in the portal until you manually delete it, or until you opt in to auto-deletion.

The admin portal Service (`ta-portal-<connectorHostname>`) is the one exception: the connector deletes it on graceful shutdown, and on a hostname or `connector_tag` change it deletes the orphan portal Service registered under the previous hostname. Data-plane Services are never auto-deleted by the connector.

### Tag declaration order

Declare every tag in `tagOwners` **before** the connector first joins. The connector uses a greedy-with-fallback chain: if it tries to claim a tag that isn't owned, it falls back to a smaller set, and the unclaimed tags simply don't appear on the node. The portal `/api/policy-snippets` endpoint generates HuJSON repair snippets for any tag the connector found undeclared.

Tailscale does not support wildcards in `tagOwners`. Patterns like `tag:aws-region-*` are not valid; you must enumerate `tag:aws-region-us-east-1`, `tag:aws-region-us-west-2`, and so on. Use the `/api/policy-snippets` workflow rather than trying to predict every concrete tag value.

## Limitations

- TCP is the primary transport. UDP is available as a global opt-in (`TSAWS_UDP_ENABLED=true`); no per-service selector.
- Single VPC per connector deployment. Cross-VPC and cross-account discovery are not supported.
- One Route 53 FQDN equals one Tailscale Service. Aurora writer and reader endpoints become separate Services.
- The connector does not delete Tailscale Services when a source record is removed. Host advertisements are withdrawn; the Service object is retained.
- The connector has no network visibility into your workloads: it discovers via DNS and proxies TCP. It can confirm that a backend accepts TCP connections, not that it's the right backend.

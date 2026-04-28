# Tailscale AWS Connector

The Tailscale AWS Connector (`tsaws`) automatically discovers private AWS services and registers them as Tailscale Services, making them accessible to authorized tailnet users by name. Once deployed, it scans opted-in Route 53 private hosted zones, creates a Tailscale Service for each eligible DNS record, and advertises itself as the Service Host that proxies tailnet traffic to the in-VPC endpoint.

No manual Service configuration is required for services you opt in. The Connector reconciles continuously: new records become Services, removed records withdraw the host advertisement (the Service itself is retained).

## How it works

The Connector runs as a single container inside your VPC. On each reconciliation cycle (default every 5 minutes) it:

1. Lists DNS records from each allowlisted Route 53 private hosted zone.
2. Filters records to those that match at least one opt-in rule (see [Eligibility](#eligibility)).
3. For managed services (RDS, Aurora, ElastiCache, and others), calls the relevant AWS describe API to confirm the port and service type.
4. Derives a stable Tailscale Service name from the FQDN and upserts the Service in the Tailscale control plane.
5. Advertises itself as a Service Host via its embedded tsnet node, then health-checks the backend and withdraws the advertisement if it becomes unreachable.

The Connector holds a single tsnet node identity on the tailnet, tagged with your configured host tag (default `tag:tsaws`). All proxied connections flow through this node.

## Supported AWS services

The Connector supports any TCP endpoint reachable from its subnet, and optionally UDP endpoints (see [UDP proxy](#udp-proxy)). For the following managed services it automatically detects the correct port from the AWS describe API:

- Amazon RDS (Postgres 5432, MySQL/MariaDB 3306, SQL Server 1433, Oracle 1521)
- Amazon Aurora (writer, reader, and custom endpoints each become separate Services)
- Amazon ElastiCache (Redis 6379, Memcached 11211)
- Amazon MemoryDB (6379)
- Amazon MSK (9092 plaintext, 9094 TLS, 9096 SASL/SCRAM)
- Amazon DocumentDB (27017)
- Amazon OpenSearch (443 or 9200)
- Amazon Neptune (8182)

For any other TCP endpoint in Route 53, provide the port via the `tailscale:ports` AWS resource tag or `TSAWS_DEFAULT_PORT`.

## Prerequisites

Before deploying:

1. A Tailscale account with Services enabled.
2. A Tailscale OAuth client with the following scopes: `services` (write), `devices:core` (write), `auth_keys` (write). Create OAuth clients at [login.tailscale.com/admin/settings/oauth](https://login.tailscale.com/admin/settings/oauth).

   **Tag scoping recommendation.** Scope the OAuth client to a single tag — typically `tag:tsaws`. The Connector's R35 greedy-with-fallback chain handles Service registrations under the broader policy tag set without requiring the OAuth client to own them all. A single-tag OAuth client minimises blast radius if the secret is compromised.

   Do **not** grant `policy_file` write. The Connector is read-only against the tailnet policy file and never writes to it. If your existing OAuth client has `policy_file` write, the Connector will warn at startup and continue with the unused scope; reduce it on next rotation.
3. The Connector's tags declared as `tagOwners` in your tailnet policy file. The `tag:tsaws` host tag must own `tag:tsaws-service` and `tag:tsaws-admin-portal` because the Connector mints the auth keys that place resources under those tags. Worked example:
   ```json
   "tagOwners": {
     "tag:tsaws":              ["autogroup:admin"],
     "tag:tsaws-service":      ["tag:tsaws"],
     "tag:tsaws-admin-portal": ["tag:tsaws"]
   }
   ```

   Connector node metadata tags (`tag:aws-region-*`, `tag:aws-vpc-*`, `tag:aws-az-*`, `tag:aws-subnet-*`, `tag:aws-account-*`, `tag:aws-cluster-*`, plus the deployment-model tag) must also be declared in `tagOwners` before the Connector node joins. See [`policy-template.hujson`](policy-template.hujson) for a complete starter template, and [Connector node tags](#connector-node-tags) for the full dimension list.
4. An AWS VPC with at least one Route 53 private hosted zone attached to it.
5. An IAM role for the Connector task (see [IAM permissions](#iam-permissions)).
6. A network path from the Connector's subnet to the Tailscale control plane (see [Network topology](#network-topology)).

## Network topology

The Connector's embedded tsnet node requires outbound connectivity to:

| Destination | Port | Purpose |
|---|---|---|
| `controlplane.tailscale.com` | 443 (TCP) | Node registration, netmap updates, key exchange |
| `login.tailscale.com` | 443 (TCP) | OAuth token exchange |
| `log.tailscale.com` | 443 (TCP) | Structured audit log delivery |
| Tailscale DERP relays | 443 (TCP) or 3478 (UDP) | Encrypted relay when direct WireGuard connections are not possible |

It also needs access to the AWS API endpoints for Route 53, the managed service describe APIs, and Secrets Manager. In most VPC configurations these are reachable via AWS-internal routes (VPC endpoints or the AWS public endpoint via NAT).

### Option 1: NAT gateway (current default)

The standard deployment places the Connector in a private subnet with a NAT gateway providing outbound internet access. No inbound security group rules are required.

```
Internet
    │
[NAT Gateway]
    │
[Private subnet]
    └── ECS task: tsaws  (outbound only)
```

This is the simplest topology and requires no additional configuration beyond the standard Terraform module. The NAT gateway costs approximately $32/month plus data transfer charges.

### Option 2: Peer Relay in a public subnet (planned, M3)

A Tailscale Peer Relay node running in a public subnet acts as a relay hop for the Connector. The Connector stays in a fully private subnet with no NAT gateway and no internet egress of its own. All Tailscale control plane and relay traffic routes through the Peer Relay over VPC-internal networking.

```
Internet
    │
[Peer Relay node — public subnet, EIP]
    │  (VPC-internal routing, no NAT)
[Private subnet]
    └── ECS task: tsaws  (no internet egress required)
```

This eliminates the NAT gateway entirely. An Elastic IP for the Peer Relay costs approximately $3.60/month when attached to a running instance — significantly less than a NAT gateway under sustained load.

### Option 3: Peer Relay as a sidecar (planned, M3)

A Peer Relay container runs in the same ECS task as the Connector. The security group opens a single inbound UDP port (default 41641) at the NAT gateway, replacing the need for full internet egress with a single forwarded port.

```
Internet
    │  (UDP 41641 only)
[NAT Gateway — single port forward]
    │
[Private subnet]
    └── ECS task: tsaws + peer-relay sidecar
```

This keeps the NAT gateway but reduces its surface from full egress to one port, which is easier to pass through a security review in environments that restrict egress but allow specific inbound ports.

Options 2 and 3 are planned for M3. Track progress in [issue #88](https://github.com/radioboxtv/tsaws/issues/88).

## Deploy the Connector

The OAuth client secret must be stored in AWS Secrets Manager before deploying. The Connector retrieves it at startup via the ECS secrets injection mechanism; it is never stored in the task definition or environment variable plaintext.

```bash
aws secretsmanager create-secret \
  --name tsaws/oauth-secret \
  --secret-string "tskey-client-..." \
  --region us-east-1
```

### Terraform (recommended)

The Terraform module in `deploy/terraform/` creates an ECS Fargate task, security group, IAM task and execution roles, and a CloudWatch log group.

```hcl
module "tsaws" {
  source = "./deploy/terraform"

  image_uri          = "ghcr.io/radioboxtv/tsaws:latest"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnets
  oauth_secret_arn   = aws_secretsmanager_secret.tsaws_oauth.arn
  hosted_zone_ids    = ["Z1234567890ABC"]
}
```

The `tailnet` variable defaults to `-`, which resolves to the tailnet associated with the OAuth client. You do not need to supply your tailnet name.

### High-availability deployment

Run two independent Connector replicas — one per Availability Zone — for redundancy. The Tailscale control plane tracks all active Service Hosts and routes tailnet connections to a healthy one automatically. If one Connector task fails health checks or is stopped, the other continues serving without operator intervention.

Each replica must have a unique `name_prefix` to avoid IAM role and resource name collisions. All other inputs (zone IDs, OAuth secret, VPC) are shared.

```hcl
module "tsaws_az1" {
  source = "./deploy/terraform"

  name_prefix        = "tsaws-az1"
  image_uri          = "ghcr.io/radioboxtv/tsaws:latest"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = [module.vpc.private_subnets[0]]
  oauth_secret_arn   = aws_secretsmanager_secret.tsaws_oauth.arn
  hosted_zone_ids    = ["Z1234567890ABC"]
  zone_opt_in        = true
}

module "tsaws_az2" {
  source = "./deploy/terraform"

  name_prefix        = "tsaws-az2"
  image_uri          = "ghcr.io/radioboxtv/tsaws:latest"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = [module.vpc.private_subnets[1]]
  oauth_secret_arn   = aws_secretsmanager_secret.tsaws_oauth.arn
  hosted_zone_ids    = ["Z1234567890ABC"]
  zone_opt_in        = true
}
```

Both replicas discover the same zone and register the same Services idempotently. Concurrent upserts are safe: Service registration uses GET-before-PUT with ETag-based retries, so duplicate writes converge without conflict.

Do not pass the same `ecs_cluster_arn` to both modules if the cluster ARN is computed by Terraform in the same root module; Terraform cannot evaluate the `count` conditional in the module before apply. Either let each module create its own cluster (the default when `ecs_cluster_arn` is omitted) or pass a pre-existing cluster ARN as a literal string.

### CloudFormation

Use the template at `deploy/cloudformation/tsaws.yaml`. It accepts the same parameters as the Terraform module and creates equivalent resources.

### Manual (ECS Fargate)

1. Pull the Connector image from `ghcr.io/radioboxtv/tsaws:latest`.
2. Create an ECS task definition with the environment variables from [Configuration](#configuration). Inject `TS_CLIENT_SECRET` as a Secrets Manager secret reference, not a plaintext environment variable.
3. Run the task in a private subnet with NAT gateway egress. No inbound security group rules are required.
4. Attach the IAM task role with the permissions listed in [IAM permissions](#iam-permissions).

## Eligibility

The Connector registers a DNS record only if it matches **at least one** of the following opt-in rules. This is intentional: there is no scan-everything default.

### DNS glob patterns (default behavior)

`TSAWS_GLOBS` controls which FQDNs are eligible by pattern matching. When unset, it defaults to `["*"]`, which registers every record in the allowlisted zones. Set it to a comma-separated list to narrow scope:

```
TSAWS_GLOBS=*.prod.company.internal,db.internal
```

To disable glob-based opt-in entirely (use tags or zone opt-in instead), set `TSAWS_GLOBS=""`.

### Zone-wide opt-in

Set `TSAWS_ZONE_OPT_IN=true` to register all records in every allowlisted zone regardless of tags or glob patterns.

### Per-resource tag

Add the tag `tailscale:register=true` to an AWS resource. The Connector discovers this via the AWS Resource Groups Tagging API and marks the corresponding Route 53 record eligible, independent of glob and zone settings.

### Port blocklist

Ports `22` and `3389` are blocked by default and never registered, even if a record is otherwise eligible. Override with `TSAWS_PORT_BLOCKLIST`. Set to an empty string to disable all blocking.

## Service naming

The Tailscale Service name is derived from the FQDN by stripping the zone root, lowercasing, and replacing dots with hyphens. Given zone `company.internal` and record `api.prod.company.internal`, the Service name is `api-prod` and the tailnet hostname is `api-prod.<tailnet>.ts.net`.

Override the name on a specific resource with the `tailscale:service-name` AWS tag.

## Access control

Three Tailscale ACL tags are independently configurable:

| Tag | Variable | Default |
|---|---|---|
| Connector node identity | `TSAWS_CONNECTOR_TAG` | `tag:tsaws` |
| Registered Services | `TSAWS_SERVICE_TAG` | falls back to `TSAWS_CONNECTOR_TAG` |
| Admin portal | `TSAWS_PORTAL_TAG` | `tag:tsaws-admin-portal` |

The Connector's tag must be declared as the owner of the other two in your tailnet policy file `tagOwners`, because the Connector mints the auth keys that place resources under those tags:

```json
"tagOwners": {
  "tag:tsaws":              ["autogroup:admin"],
  "tag:tsaws-service":      ["tag:tsaws"],
  "tag:tsaws-admin-portal": ["tag:tsaws"]
}
```

This separation enables fine-grained access segmentation: tailnet users or groups can be granted access to registered Services without any access to the portal, and portal access can be restricted to operators independently of service access. The Connector node itself remains a distinct identity on the tailnet.

### Approving service connections

For tailnet users to reach a registered Service, an `autoApprovers.services` entry must exist in your tailnet policy file for the Service's ACL tag. The Connector does not write policy file entries; configure this once before deploying.

```json
"autoApprovers": {
  "services": {
    "tag:tsaws-service": ["tag:tsaws"]
  }
}
```

This authorizes any device carrying `tag:tsaws` to advertise Services under `tag:tsaws-service`. Adjust the tag names to match your `TSAWS_SERVICE_TAG` and `TSAWS_CONNECTOR_TAG` values.

## Connector node tags

The Connector's tsnet node is tagged at startup with metadata inferred from the AWS runtime environment. These tags identify the Connector's region, VPC, subnet, AZ, account, cluster, and deployment model. They appear on the Connector's device entry in the Tailscale admin console and can be used in tailnet ACL policy to scope access.

Baseline AWS metadata tags are applied to the Connector node only, not to individual Services. Service identity stays minimal, and operators can scope ACLs by environment without exploding the per-Service tag set.

All tag gates default to enabled. Set a variable to `false` to suppress that tag.

| Variable | Default | Tag applied |
|---|---|---|
| `TSAWS_CONNECTOR_REGION_TAG_ENABLED` | `true` | `tag:aws-region-<slug>` (e.g. `tag:aws-region-us-east-1`) |
| `TSAWS_CONNECTOR_VPC_TAG_ENABLED` | `true` | `tag:aws-vpc-<slug>` (VPC Name tag, or VPC ID when no Name tag) |
| `TSAWS_CONNECTOR_SUBNET_TAG_ENABLED` | `true` | `tag:aws-subnet-<slug>` (subnet Name tag, or subnet ID when no Name tag) |
| `TSAWS_CONNECTOR_AZ_TAG_ENABLED` | `true` | `tag:aws-az-<slug>` (e.g. `tag:aws-az-us-east-1a`) |
| `TSAWS_CONNECTOR_ACCOUNT_TAG_ENABLED` | `true` | `tag:aws-account-<id>` |
| `TSAWS_CONNECTOR_CLUSTER_TAG_ENABLED` | `true` | `tag:aws-cluster-<name>` (ECS cluster short name or EKS cluster name) |
| `TSAWS_CONNECTOR_DEPLOYMENT_TAG_ENABLED` | `true` | `tag:aws-ecs-fargate`, `tag:aws-ecs-ec2`, `tag:aws-ec2`, `tag:aws-eks`, or `tag:aws-lambda` |
| `TSAWS_CONNECTOR_CUSTOM_TAGS` | none | Comma-separated additional tags applied unconditionally |

Tag values are lower-cased and non-alphanumeric runs are collapsed to a single hyphen. Region, AZ, and account come from EC2 IMDSv2. VPC and subnet names come from `ec2:DescribeVpcs` and `ec2:DescribeSubnets`. ECS cluster comes from the ECS task metadata endpoint. Deployment model comes from `AWS_EXECUTION_ENV`.

On runtimes where a field is not available (e.g. no IMDS on Lambda), the corresponding tag is silently omitted. For EKS deployments, set `TSAWS_EKS_CLUSTER_NAME` to the cluster name because it cannot be derived from IMDS.

All tags applied to the Connector node must be declared in `tagOwners` in your tailnet policy file before the node joins.

## Admin portal

When `TSAWS_PORTAL_ENABLED` is `true` (the default), the Connector registers itself as a Tailscale Service and serves a read-only admin web UI on port 443 over the tailnet. Access is gated by ACL grants referencing `TSAWS_PORTAL_TAG`.

To access the portal, create a grant in your tailnet policy file that allows devices bearing your user tag to connect to `tag:tsaws-admin-portal` on port 443.

The portal exposes the following endpoints:

| Endpoint | Description |
|---|---|
| `/api/status` | All discovered FQDNs and their current state (registered, degraded, etc.) |
| `/api/events` | Last 500 structured events, newest first. Filter by kind with `?kind=ServiceCreated`. |
| `/api/cycles` | Last 20 reconciliation cycle summaries: start time, duration, and created/updated/reused/failed counts |
| `/api/config` | Effective Connector configuration. OAuth secret and secret source are redacted. |
| `/api/health` | TCP probe results for Tailscale and Route 53 connectivity targets |
| `/api/traffic` | Per-Service traffic byte counters |
| `/api/runtime` | Go runtime metrics (goroutine count, heap, version) |

## Health checks

The Connector runs a background health check for each advertised `(service, port)` pair. On `TSAWS_HEALTH_UNHEALTHY_THRESHOLD` consecutive failures (default 3), it withdraws the host advertisement and transitions the Service to `degraded` state. After `TSAWS_HEALTH_HEALTHY_THRESHOLD` consecutive successes (default 2), it re-advertises and returns to `registered`.

Health check mode is controlled by `TSAWS_HEALTH_MODE`:

- `l4_tcp` (default): TCP connect to the backend port. Passes on successful handshake within `TSAWS_HEALTH_TIMEOUT`.
- `l7_http`: HTTP GET to the backend. Passes on any 2xx, 3xx, or 4xx response. Path defaults to `/`; override per-resource with the `tailscale:health-path` AWS tag.
- `none`: No health checks. Advertisement is sticky; only source removal or shutdown withdraws it.

Protocol-aware L4.5 modes perform a minimal application handshake without credentials, giving a more accurate signal than a bare TCP connect for managed protocol backends:

- `redis`: Sends `PING` (RESP2) and expects `+PONG`.
- `postgres`: Sends a StartupMessage and expects any server response (including an authentication challenge).
- `mysql`: Reads the server handshake packet (expects a 4-byte header).
- `kafka`: Sends an ApiVersions request and reads the response length prefix.
- `mongodb`: Sends a minimal `isMaster` OP_QUERY and reads the response header.
- `memcached`: Sends `version\r\n` and expects `VERSION`.
- `opensearch`: HTTPS GET to `/_cluster/health` (InsecureSkipVerify); passes on any 2xx response.

Set `TSAWS_HEALTH_AUTO_L7=true` to automatically upgrade targets on ports 80, 443, 8080, or 8443 from L4 TCP to L7 HTTP checks. Targets on other ports remain L4.

Set `TSAWS_HEALTH_AUTO_PROTOCOL=true` to automatically select a protocol-aware L4.5 mode based on the service FQDN when the global mode is `l4_tcp`. The FQDN is matched against known AWS managed service DNS suffixes (e.g. `.cache.amazonaws.com` selects `redis`, `.rds.amazonaws.com` selects `postgres`). Services that do not match any suffix remain on `l4_tcp`. Setting `TSAWS_HEALTH_MODE=auto` applies the same FQDN-based selection globally.

Per-service overrides are applied via the `tailscale:health-mode` AWS resource tag.

## UDP proxy

When `TSAWS_UDP_ENABLED=true`, the Connector forwards UDP datagrams in addition to TCP connections. This is disabled by default.

**How it works:** for each advertised `(service, port)` pair, the Connector calls `tsnet.ListenPacket("udp", addr)` on the node's tailnet IPv4 address. Incoming datagrams are forwarded to the backend using a per-source flow table: the first packet from a new client address creates a flow (a connected UDP socket to the backend), and subsequent packets reuse it. Flows idle for longer than `TSAWS_UDP_IDLE_TIMEOUT` (default 60s) are torn down. Byte counts are tracked in the same traffic counter as TCP.

**Protocol selection is global.** When UDP is enabled, every discovered service gets UDP forwarding on its configured port. There is no per-service or per-protocol selector. Use this for workloads where all proxied services legitimately need UDP (for example, a tailnet exclusively proxying DNS resolvers or syslog endpoints). Mixed fleets (some TCP-only, some UDP) are not selectively configurable in the current release.

**Upstream note:** The Tailscale Services API port format for UDP (`"udp:PORT"`) is unconfirmed. The local UDP proxy works independently of this API field; the Tailscale control plane may reject the UDP port entries until the API officially supports UDP service ports.

## Dry-run mode

Run the Connector with `TSAWS_DRY_RUN=true` (or `--dry-run`) to preview what would be registered without making any writes to the Tailscale control plane or AWS.

The output is a JSON document with three sections:

```json
{
  "Summary": { "WouldCreate": 3, "WouldUpdate": 1, "WouldSkip": 12, "Orphans": 0, "Failed": 0 },
  "Desired": [
    { "fqdn": "api.prod.internal.", "service_name": "svc:api-prod", "ports": ["tcp:443"], "action": "would-create" },
    { "fqdn": "db.prod.internal.",  "service_name": "svc:db-prod",  "ports": ["tcp:5432"], "action": "would-update", "changes": ["ports: [tcp:3306] -> [tcp:5432]"] },
    ...
  ],
  "Orphans": [
    { "service_name": "svc:old-service", "action": "orphan" }
  ]
}
```

Each record in `Desired` has one of four actions:

| Action | Meaning |
|---|---|
| `would-create` | No matching Service exists in Tailscale; one would be created |
| `would-update` | Service exists but ports or description differ; it would be updated |
| `would-skip` | Service exists and matches desired state; no write needed |
| `orphan` | Service exists in Tailscale but has no matching Route 53 source |

Use dry-run to validate your zone allowlist, glob patterns, and tag configuration before enabling writes. If `TS_CLIENT_SECRET` is not set, dry-run still runs full discovery but skips the comparison against live Tailscale state.

## Status and monitoring

### Local HTTP endpoint

The Connector serves a JSON status endpoint at `TSAWS_STATUS_ADDR` (default `:8080`). It lists every discovered FQDN and its current state. Query it from within the VPC:

```
GET http://<task-ip>:8080/
```

Each record has one of these states:

| State | Meaning |
|---|---|
| `eligible` | Passed eligibility gates, queued for registration |
| `registered` | Service advertised and health check passing |
| `degraded` | Advertised but health check failing |
| `ineligible` | Did not match any opt-in rule |
| `deleted-from-source` | Source record removed; host advertisement withdrawn, Service retained |
| `orphaned` | Service exists in Tailscale but has no matching Route 53 source |
| `cap-exceeded` | Eligible but the configured service cap was reached this cycle |

### Structured logs

All events are emitted as JSON lines to stdout, picked up by CloudWatch Logs or any log aggregator. Each reconciliation cycle carries a `correlation_id` field. Key event types include `ServiceCreated`, `ServiceUpdated`, `ServiceReused`, `ServiceHostRegistered`, `HealthCheckFailed`, `HealthCheckRecovered`, `ReconciliationCompleted`, and `CapExceeded`.

### Tailscale admin console

The Connector tags each audit log entry with `source=tsaws-connector`, making its activity filterable in the Tailscale admin console.

## Configuration reference

All configuration is via environment variables. Variables are read once at startup; restart to pick up changes.

A subset of these tunables (`TSAWS_RECONCILE_INTERVAL`, `TSAWS_MAX_SERVICES`, `TSAWS_DEFAULT_PORT`, `TSAWS_PORT_BLOCKLIST`, `TSAWS_GLOBS`, `TSAWS_ZONE_OPT_IN`) can also be updated at runtime via the Tailscale node-attribute capability — no restart required. See [node-attr-config.md](node-attr-config.md).

### Required

| Variable | Description |
|---|---|
| `TS_CLIENT_SECRET` | Tailscale OAuth client secret. Mutually exclusive with `TSAWS_CLIENT_SECRET_SOURCE`. |
| `TSAWS_HOSTED_ZONE_IDS` | Comma-separated Route 53 private hosted zone IDs to discover (e.g. `Z1234567890ABC,Z0987654321DEF`). |

### Secrets

| Variable | Description |
|---|---|
| `TSAWS_CLIENT_SECRET_SOURCE` | URI for the OAuth secret: `env://TS_CLIENT_SECRET` (default), `aws-secretsmanager://<secret-id>`, `aws-ssm://<parameter-name>`, or `file:///path/to/secret`. |

### Tailscale

| Variable | Default | Description |
|---|---|---|
| `TSAWS_CONNECTOR_TAG` | `tag:tsaws` | Tailscale ACL host tag for the Connector's tsnet node. |
| `TSAWS_CONNECTOR_HOSTNAME` | auto | Verbatim hostname for the tsnet node. When unset, derived from region + VPC + subnet. Override only when you need deterministic per-replica hostnames; multi-replica disambiguation otherwise comes from tsnet's automatic `-2`/`-3` suffix. The hostname is also used as the `owner=` prefix in Service comments for the conflict tie-break, so changing it via the portal triggers a full reconnect. |
| `TSAWS_TAILNET` | `-` | Tailnet name. `-` resolves to the default tailnet for the OAuth client. You do not need to set this explicitly. |
| `TSAWS_SERVICE_TAG` | none | Single Tailscale ACL tag applied to every registered Service. When empty, falls back to `TSAWS_CONNECTOR_TAG`. Use this to apply a stable tag to all Services independent of the identity tags system (e.g. `tag:tsaws-service`). |

### Discovery and eligibility

| Variable | Default | Description |
|---|---|---|
| `TSAWS_GLOBS` | `*` | Comma-separated DNS glob patterns for eligibility. Unset defaults to `*` (all records). Set to empty string to disable glob matching. |
| `TSAWS_ZONE_OPT_IN` | `false` | When `true`, all records in allowlisted zones are eligible regardless of tags or globs. |
| `TSAWS_DEFAULT_PORT` | none | Fallback TCP port when no other source derives a port. |
| `TSAWS_PORT_BLOCKLIST` | `22,3389` | Comma-separated ports to never register. Set to empty string to disable. |
| `TSAWS_MAX_SERVICES` | `1000` | Hard cap on Services upserted per reconciliation cycle. Records are processed in FQDN-ascending order; those beyond the cap enter `cap-exceeded` state. |

### Operation

| Variable | Default | Description |
|---|---|---|
| `TSAWS_RECONCILE_INTERVAL` | `5m` | How often the reconciliation loop runs. Minimum `1s`. Live-tunable via the node-attribute cap; see [node-attr-config.md](node-attr-config.md). |
| `TSAWS_STATUS_ADDR` | `:8080` | Address for the local status HTTP endpoint. |
| `TSAWS_DRY_RUN` | `false` | When `true`, run full discovery but make no writes. |
| `TSAWS_SHUTDOWN_DRAIN` | `30s` | Maximum time to wait for in-flight proxy connections to finish after SIGTERM/SIGINT. |
| `TSAWS_SHUTDOWN_KEEP_SESSIONS` | `false` | When `true`, `Shutdown` skips the force-close of in-flight connections. Use during planned restarts where session continuity matters more than fast shutdown. |
| `TSAWS_METRICS_ADDR` | none | Address for a Prometheus metrics HTTP endpoint (e.g. `:9090`). Empty disables metrics. |
| `TSAWS_WEBHOOK_URL` | none | Outbound webhook URL: every event in the ring is POSTed here as JSON. Empty disables webhook. |
| `TSAWS_WEBHOOK_SECRET` | none | HMAC-SHA256 secret used to sign webhook payloads (header `X-Tsaws-Signature`). |

### Health checks

| Variable | Default | Description |
|---|---|---|
| `TSAWS_HEALTH_MODE` | `l4_tcp` | Health check protocol: `l4_tcp`, `l7_http`, `none`, `redis`, `postgres`, `mysql`, `kafka`, `mongodb`, `memcached`, `opensearch`, or `auto`. |
| `TSAWS_HEALTH_TIMEOUT` | `5s` | Per-probe timeout. |
| `TSAWS_HEALTH_INTERVAL` | `30s` | Interval between probes. |
| `TSAWS_HEALTH_UNHEALTHY_THRESHOLD` | `3` | Consecutive failures before withdrawing advertisement. |
| `TSAWS_HEALTH_HEALTHY_THRESHOLD` | `2` | Consecutive successes before re-advertising after a failure. |
| `TSAWS_HEALTH_AUTO_L7` | `false` | Upgrade targets on ports 80, 443, 8080, 8443 from L4 TCP to L7 HTTP checks automatically. Targets on other ports remain L4. |
| `TSAWS_HEALTH_AUTO_PROTOCOL` | `false` | When `TSAWS_HEALTH_MODE` is `l4_tcp`, automatically select a protocol-aware L4.5 mode from the service FQDN. Services that do not match a known suffix remain on `l4_tcp`. |
| `TSAWS_HEALTH_CONFIG_FILE` | none | Path to a JSON file persisting per-Service health-mode overrides and forced-unhealthy (drain) state across restarts. Empty keeps overrides in memory only — they are lost on restart. |

### UDP proxy

| Variable | Default | Description |
|---|---|---|
| `TSAWS_UDP_ENABLED` | `false` | When `true`, forward UDP datagrams for every advertised (service, port) pair in addition to TCP. |
| `TSAWS_UDP_IDLE_TIMEOUT` | `60s` | Flow idle timeout. Flows with no traffic for this duration are torn down. |

### Connector node tags

The Connector applies these tags automatically at startup using a greedy-with-fallback chain. Disable an individual dimension by setting its `*_ENABLED` variable to `false`. See the [Connector node tags](#connector-node-tags) section above for how the dimensions combine.

| Variable | Default | Description |
|---|---|---|
| `TSAWS_CONNECTOR_REGION_TAG_ENABLED` | `true` | Apply `tag:aws-region-<slug>` (e.g. `tag:aws-region-us-east-1`) to the Connector node. |
| `TSAWS_CONNECTOR_VPC_TAG_ENABLED` | `true` | Apply `tag:aws-vpc-<slug>` (VPC Name tag, or VPC ID when no Name tag). |
| `TSAWS_CONNECTOR_SUBNET_TAG_ENABLED` | `true` | Apply `tag:aws-subnet-<slug>` (subnet Name tag, or subnet ID when no Name tag). |
| `TSAWS_CONNECTOR_AZ_TAG_ENABLED` | `true` | Apply `tag:aws-az-<slug>` (e.g. `tag:aws-az-us-east-1a`). |
| `TSAWS_CONNECTOR_ACCOUNT_TAG_ENABLED` | `true` | Apply `tag:aws-account-<id>`. |
| `TSAWS_CONNECTOR_CLUSTER_TAG_ENABLED` | `true` | Apply `tag:aws-cluster-<name>` (ECS cluster short name or EKS cluster name from `TSAWS_EKS_CLUSTER_NAME`). |
| `TSAWS_CONNECTOR_DEPLOYMENT_TAG_ENABLED` | `true` | Apply the runtime deployment tag: `tag:aws-ecs-fargate`, `tag:aws-ecs-ec2`, `tag:aws-ec2`, `tag:aws-eks`, or `tag:aws-lambda`. |
| `TSAWS_CONNECTOR_CUSTOM_TAGS` | none | Comma-separated additional tags applied unconditionally to the Connector node. |
| `TSAWS_EKS_CLUSTER_NAME` | none | EKS cluster name (required for `tag:aws-cluster-*` on EKS; not derivable from IMDS). |

### Admin portal

| Variable | Default | Description |
|---|---|---|
| `TSAWS_PORTAL_ENABLED` | `true` | When `false`, the Connector does not register the admin portal Service. |
| `TSAWS_PORTAL_TAG` | `tag:tsaws-admin-portal` | Tailscale ACL tag for the admin portal Service. |
| `TSAWS_PORTAL_VPC_ID` | none | Legacy: previously used in the portal Service name hash. Since per-Connector portal Services were introduced (per-replica HA), the portal Service name is `tsaws-portal-<connectorHostname>` and this variable is no longer load-bearing. Retained for backward compatibility; safe to leave unset. |

### Rate limiting

| Variable | Default | Description |
|---|---|---|
| `TSAWS_RATELIMIT_TS_RPS` | `10` | Tailscale API rate limit in requests per second. |
| `TSAWS_RATELIMIT_TS_BURST` | `20` | Tailscale API burst allowance. |
| `TSAWS_RATELIMIT_AWS_RPS` | `50` | AWS API (non-Route 53) rate limit in requests per second. |
| `TSAWS_RATELIMIT_AWS_BURST` | `100` | AWS API burst allowance. |
| `TSAWS_RATELIMIT_ROUTE53_RPS` | `5` | Route 53 API rate limit in requests per second. |
| `TSAWS_RATELIMIT_ROUTE53_BURST` | `10` | Route 53 API burst allowance. |

### AWS context

The Connector reads region from the AWS SDK chain (`AWS_REGION`, then `AWS_DEFAULT_REGION`, then EC2 instance metadata). You don't normally set these explicitly: when running on ECS Fargate / EC2 / Lambda / EKS the runtime supplies them. Override only when running outside AWS or pointing at a non-default region.

## AWS resource tags

Apply these tags directly to an AWS resource to override Connector behavior for the corresponding Service.

| Tag | Description |
|---|---|
| `tailscale:register=true` | Mark this resource eligible for registration (alternative to glob or zone opt-in). |
| `tailscale:service-name=<name>` | Override the derived Service name. |
| `tailscale:ports=<port>[,<port>]` | Override the derived port list. Comma-separated TCP ports. |
| `tailscale:target=<host:port>` | Override the backend proxy target. |
| `tailscale:description=<text>` | Override the Service description. |
| `tailscale:tags=<tag>[,<tag>]` | Override the Tailscale ACL tags applied to this Service. |
| `tailscale:health-mode=<mode>` | Per-Service health check mode: `l4_tcp`, `l7_http`, `none`, `redis`, `postgres`, `mysql`, `kafka`, `mongodb`, `memcached`, `opensearch`, or `auto`. |
| `tailscale:health-path=<path>` | HTTP path for L7 health checks. Default `/`. |

## IAM permissions

The Connector requires read-only access only. It never mutates AWS resources.

Attach the following policy to the ECS task role. The Terraform module and CloudFormation template create this role automatically.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Route53Read",
      "Effect": "Allow",
      "Action": ["route53:ListHostedZones", "route53:ListResourceRecordSets", "route53:GetHostedZone"],
      "Resource": "*"
    },
    {
      "Sid": "EC2Describe",
      "Effect": "Allow",
      "Action": ["ec2:Describe*"],
      "Resource": "*"
    },
    {
      "Sid": "ELBDescribe",
      "Effect": "Allow",
      "Action": ["elasticloadbalancing:Describe*"],
      "Resource": "*"
    },
    {
      "Sid": "ManagedServicesDescribe",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters",
        "elasticache:Describe*",
        "memorydb:Describe*",
        "kafka:Describe*",
        "kafka:List*",
        "docdb:Describe*",
        "es:Describe*",
        "neptune-db:GetEngineStatus",
        "tag:GetResources"
      ],
      "Resource": "*"
    }
  ]
}
```

The full policy with per-statement rationale is in `docs/aws-permissions.md`.

## Limitations

- TCP is the primary transport. UDP datagram forwarding is available as an opt-in (`TSAWS_UDP_ENABLED=true`) but applies to all discovered services globally; there is no per-service selector. L7 routing, TLS termination, and protocol-aware proxying are not performed.
- Single VPC per Connector deployment. Cross-VPC and cross-account discovery are not supported.
- One Route 53 FQDN equals one Tailscale Service. Aurora writer and reader endpoints become separate Services.
- The Connector does not delete Tailscale Services when a source record is removed. Host advertisements are withdrawn; the Service object is retained. Opt-in auto-deletion is planned for a future release.
- Port `22` (SSH) and `3389` (RDP) are blocked by default.
- The Connector does not have network visibility into your workloads: it discovers services via DNS and proxies TCP connections. It cannot verify that a backend endpoint is correct, only that it accepts TCP connections.

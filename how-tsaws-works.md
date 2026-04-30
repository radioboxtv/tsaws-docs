# How tsaws works

A conceptual overview of the Connector's runtime model, discovery loop, and the design properties that follow from them. For configuration details, see [configuration.md](configuration.md). For setup, see [getting-started.md](getting-started.md).

## What it does

`tsaws` runs as a single ECS Fargate task inside a customer VPC, discovers eligible DNS records from Route 53 private hosted zones, registers each FQDN as a Tailscale Service via the Tailscale Services API, and advertises itself as a Service Host via tsnet. The result: tailnet users can reach private AWS endpoints (RDS, ElastiCache, internal ALBs, and others) by name through the tailnet, with no manual Service configuration.

## Runtime model

```
VPC (private subnet, NAT egress)
├── ECS Fargate task: tsaws-az1  (AZ a)
│   ├── tsnet node      joins tailnet as tag:tsaws, advertises Service Host routes
│   ├── discovery loop  runs every N minutes (default 5m)
│   ├── portal          admin UI on port 443 over the tailnet
│   └── status HTTP     :8080 in-VPC health and state endpoint
└── ECS Fargate task: tsaws-az2  (AZ b, optional, for HA)
    ├── tsnet node      same tailnet, same Services, different host
    ├── discovery loop  idempotent, converges to the same Service set
    ├── portal          per-replica portal Service
    └── status HTTP     :8080
```

The process is a single static Go binary. It uses `tailscale.com/tsnet` to embed a Tailscale node directly with no sidecar `tailscaled`. Authentication is via a scoped OAuth client secret (`TS_CLIENT_SECRET`), exchanged at runtime for a short-lived access token.

For HA, run two independent replicas in separate Availability Zones. Both register the same Services idempotently; the Tailscale control plane tracks all active hosts and routes tailnet connections to a healthy one. See [configuration.md](configuration.md) for the Terraform pattern.

## Discovery loop

Each cycle the Connector:

1. **Discovers**. Lists DNS records from each allowlisted Route 53 private hosted zone via `route53:ListResourceRecordSets`.
2. **Filters**. Applies opt-in rules: zone-wide opt-in, FQDN globs, and per-resource AWS tags. Rules combine as logical OR. See [Eligibility](#eligibility).
3. **Enriches**. For records that resolve to managed AWS endpoints (RDS, Aurora, ElastiCache, MemoryDB, MSK, DocumentDB, OpenSearch, Neptune), calls the relevant AWS describe API to confirm the canonical port and metadata.
4. **Translates**. Derives a Tailscale Service name (flattened FQDN minus zone root, dots replaced with hyphens, lowercased), the port set, and the ACL tag set.
5. **Registers**. Upserts the Service in the Tailscale control plane via `PUT /api/v2/tailnet/{tailnet}/services/svc:{name}`. The registrar uses GET-before-PUT to satisfy the API constraint that updates must include existing IPv4 and IPv6 addresses.
6. **Advertises**. Calls `host.Advertise` on the tsnet node for each `(service, port)` pair. Pairs that disappeared since the last cycle are deregistered. The Service object itself is retained.

Cycles are idempotent. Two replicas running in parallel converge to the same Service set; concurrent upserts are safe through ETag-based retry.

## Eligibility

A record passes the filter when **at least one** opt-in rule matches:

- **Zone-wide opt-in**. Every record in an allowlisted Route 53 zone is eligible. Useful when an operator dedicates a hosted zone to tsaws.
- **FQDN glob**. The record's bare FQDN matches at least one glob in the connector's `domains` list. Glob syntax is `path.Match`: `?`, `*`, `[abc]`, `[!abc]`.
- **Per-resource AWS tag**. The underlying AWS resource carries `tailscale:register=true`. Discovered via the Resource Groups Tagging API and applied independently of glob and zone settings.

The connector never registers a record that fails every rule. There is no scan-everything default.

## Port derivation

Ports are derived in this priority order:

1. SRV record value
2. `tailscale:port` AWS resource tag
3. Managed-service enrichment (RDS engine type, MSK security protocol, etc.)
4. TCP probe of the backend
5. `default_port` (cap or `TSAWS_DEFAULT_PORT`)
6. `do-not-validate` sentinel sent to the Tailscale API (UDP only)

If every step yields nothing, the record is dropped with a `ServiceTranslationFailed` event. There is no per-Service "guess".

## Service naming

The Tailscale Service name is the FQDN minus the zone root, lowercased, with dots replaced by hyphens. Given zone `company.internal` and record `api.prod.company.internal`, the Service name is `api-prod` and the tailnet hostname is `api-prod.<tailnet>.ts.net`.

On collision (two records from different zones flatten to the same name), the Connector appends a 6-character hash. Override the derived name on a specific resource with the `tailscale:service-name` AWS tag.

## Identity model

Three identities exist, each with its own Tailscale ACL tag:

| Identity | Default tag | Purpose |
|---|---|---|
| Connector node | `tag:tsaws` | The tsnet node's tailnet identity. Controls who can reach the connector itself. |
| Registered Services | (falls back to connector tag) | Applied to every Service the connector creates. Controls tailnet access to the proxied AWS endpoints. |
| Admin portal Service | `tag:tsaws-admin-portal` | Applied to the portal's own Service. Lets operators gate portal access independently of the data-plane services. |

The connector node is also tagged with AWS metadata at startup:

- `tag:aws-region-<slug>`
- `tag:aws-vpc-<slug>`
- `tag:aws-subnet-<slug>`
- `tag:aws-az-<slug>`
- `tag:aws-account-<id>`
- `tag:aws-cluster-<name>`
- `tag:aws-ecs-fargate` / `tag:aws-ecs-ec2` / `tag:aws-ec2` / `tag:aws-eks` / `tag:aws-lambda` (deployment model)

These are applied to the Connector node only, not to individual Services. Service identity stays minimal so operators can scope ACLs by environment without exploding the per-Service tag set. Apply per-Service tags via `custom_tags` in the node-attribute cap (FQDN-glob → tag list mapping); see [configuration.md](configuration.md#acl-patterns).

Tag application uses a greedy-with-fallback chain: if the full set is undeclared in `tagOwners`, the connector falls back to a smaller set, ultimately to the connector tag alone. The portal `/api/tags` and `/api/policy-snippets` endpoints surface what was applied and which declarations are missing.

## Health engine

For each advertised `(service, port)` pair, a background goroutine probes the backend and gates the Tailscale Service Host advertisement on the result.

Modes:

- `l4_tcp` — TCP connect. Passes on successful handshake within the timeout.
- `l7_http` — HTTP GET. Passes on any 2xx, 3xx, or 4xx response.
- `l4.5` protocol probes — `redis` (RESP `PING`), `postgres` (StartupMessage), `mysql` (handshake read), `kafka` (ApiVersions), `mongodb` (`isMaster` OP_QUERY), `memcached` (`version`), `opensearch` (HTTPS `/_cluster/health`).
- `none` — Advertisement is sticky; only source removal or shutdown withdraws it.
- `auto` — Pick a mode from FQDN heuristics (e.g. `.cache.amazonaws.com` → `redis`).

Defaults that ship enabled:

- `TSAWS_HEALTH_AUTO_L7=true` — Targets on ports 80, 443, 8080, 8443 are probed with the HTTP detector and upgraded from L4 to L7 only when the backend actually speaks HTTP. Backends that do not (Postgres-over-TLS, custom binary protocols on 443) stay on L4.
- `TSAWS_HEALTH_AUTO_PROTOCOL=true` — When the global mode is `l4_tcp`, the engine selects an L4.5 probe per service from FQDN heuristics.
- `TSAWS_HEALTH_AUTO_DETECT=true` — At first registration of a service, the engine runs the multi-protocol detection sweep once and applies the recommended mode via the per-service health config store. Operator-set modes are never overwritten.

Per-service overrides come from the `tailscale:health-mode` AWS resource tag, the portal `/api/services/{name}/health-check` PUT endpoint, or the on-disk health config file.

After `unhealthy_threshold` consecutive failures (default 3), the connector withdraws the host advertisement and transitions the Service to `degraded`. After `healthy_threshold` consecutive successes (default 2), it re-advertises and returns to `registered`.

## Configuration surfaces

The connector reads configuration from four surfaces, applied in this order:

1. **Environment variables** — read once at process start. Sets the floor.
2. **Node-attribute capability `tsaws.com/config`** — pushed by NetMap, applied each cycle. Overrides env vars for the fields it covers. Survives only as long as the cap remains.
3. **Portal `POST /api/config/runtime`** — operator override for live experimentation. Reverts on the next NetMap push if the cap is also active.
4. **AWS resource tags** (`tailscale:*`) — per-resource overrides. Read each cycle.

Per-Service health config (mode, drain state) is also persisted to `TSAWS_HEALTH_CONFIG_FILE` when set.

The policy file (tailnet ACLs, autoApprovers) is independent. The connector reads it for bootstrap validation only and never writes to it.

See [configuration.md](configuration.md#configuration-surfaces) for delegation order and precedence details.

## Safety properties

These hold by design:

- **Read-only IAM**. The connector requires only `route53:List*`, `route53:GetHostedZone`, `ec2:Describe*`, `elasticloadbalancing:Describe*`, the managed-service `Describe*` actions, and `tag:GetResources`. No `route53:Change*`, no EC2 mutations, no IAM mutations. Ever. See [aws-permissions.md](aws-permissions.md).
- **No `policy_file` write OAuth scope**. The connector reads the tailnet policy file for bootstrap validation but never writes. Operators author `autoApprovers.services` themselves once, before deployment.
- **GET-before-PUT**. Service upserts always fetch the current addrs before writing, so concurrent replicas converge without overwriting each other.
- **Deregister, do not delete**. When a source DNS record disappears, the host advertisement is withdrawn but the Tailscale Service object is retained. Auto-deletion is opt-in and disabled by default. The admin portal Service is the one exception: it is deleted on graceful shutdown, and the orphan portal Service from a prior hostname is deleted when an operator changes `connector_tag` or the connector hostname.
- **Skip-not-fail validation**. Cap and runtime config validation drops invalid fields rather than rejecting the whole update. The connector keeps running on prior state.

## Implementation status

| Requirement | Status |
|---|---|
| R1 Deployment artifacts | Done — Dockerfile, Terraform module, CloudFormation template, CI publish job |
| R2 Route 53 discovery | Done |
| R4 Discovery filtering | Done — zone opt-in, globs, resource tags |
| R5 Service naming | Done |
| R6 Transport and ports | Done |
| R7 Service tags | Done — `service_tag` and `custom_tags` |
| R8 Service description | Done |
| R9 Service idempotency | Done |
| R10 Service Host registration | Done |
| R11 Service Host deregistration | Done |
| R12 Managed services | Done — RDS, Aurora, ElastiCache, MemoryDB, MSK, DocumentDB, OpenSearch, Neptune |
| R13 Notifications and events | Done — slog JSON events, in-memory ring buffers |
| R21 Service approval | Done — operator-authored `autoApprovers.services` |
| R22 Dry-run mode | Done — would-create, would-update, would-skip, orphan diff |
| R28 Service cap enforcement | Done |
| R29 Health engine | Done — L4, L7, seven L4.5 protocols, AutoL7, AutoProtocol, AutoDetect |
| R30 Admin portal | Done |
| R32 Bootstrap validation | Done — STS, Route 53 IAM, OAuth, zone existence, identity tag declarations |
| R33 Rate limiting | Done — token-bucket per API |
| R37 Node-attribute runtime config | Done — ten cap fields, skip-not-fail validation |
| R40 Connector node tags | Done — region, AZ, VPC, subnet, account, cluster, deployment |
| R\_UDP UDP proxy | Done — opt-in, per-source flow table |
| R38 Hostname replica suffix | Done — `ta-<suffix>-<base>-<region>` shape; suffix from subnet Name, SubnetID, AZ, ECS task ARN, or EKS pod name; 63-char DNS-label budget |

Tracked but not in scope:

- HA orchestration beyond independent replicas
- Drift reconciliation and auto-deletion
- Cloud Map private DNS namespace discovery
- Multi-region, multi-account, multi-VPC discovery
- ACL synthesis and grant authoring

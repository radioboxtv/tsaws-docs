# tsaws: Architecture, Design, and Status

## What it is

`tsaws` is the Tailscale Service Connector for AWS. It runs as a single ECS Fargate task inside a customer VPC, discovers eligible DNS records from Route 53 private hosted zones, registers each FQDN as a Tailscale Service via the Tailscale Services API, and advertises itself as a Service Host via tsnet. The result: Tailscale users can reach private AWS endpoints (RDS, ElastiCache, internal ALBs, etc.) by name through the tailnet, with no manual Service configuration.

## Architecture

### Runtime model

```
VPC (private subnet, NAT egress)
├── ECS Fargate task: tsaws-az1  (AZ a)
│   ├── tsnet node  ← joins tailnet as tag:tsaws, advertises Service Host routes
│   ├── reconciler  ← runs every N minutes (default 5m)
│   └── status HTTP ← :8080, local health/state endpoint
└── ECS Fargate task: tsaws-az2  (AZ b, optional — for HA)
    ├── tsnet node  ← same tailnet, same Services, different host
    ├── reconciler  ← idempotent — converges to the same Service set
    └── status HTTP ← :8080
```

The process is a single static Go binary. It uses `tailscale.com/tsnet` to embed a Tailscale node directly (no sidecar tailscaled). Authentication is via a scoped OAuth client secret (`TS_CLIENT_SECRET`), exchanged at runtime for a short-lived access token.

For HA, run two independent replicas in separate Availability Zones. Both register the same Services idempotently; the Tailscale control plane tracks all active hosts and routes tailnet connections to a healthy one. See `docs/connector-aws.md` for the Terraform pattern.

### Reconciliation loop

Each cycle:
1. **Discover** — list all DNS records from allowlisted Route 53 private hosted zones
2. **Filter** — apply opt-in rules: zone-wide opt-in (`TSAWS_ZONE_OPT_IN=true`), DNS glob patterns (`TSAWS_GLOBS`), or per-record AWS resource tag (`tailscale:register=true`)
3. **Enrich** — for records that resolve to managed AWS endpoints (RDS, Aurora, ElastiCache, MemoryDB, DocumentDB, OpenSearch, Neptune), call the relevant AWS describe API to get the canonical port and metadata
4. **Translate** — derive a Tailscale Service name (flattened FQDN relative to zone root), ports (from SRV record, AWS tags, managed enrichment, TCP probe, or `TSAWS_DEFAULT_PORT`), and ACL tags
5. **Register** — upsert the Service in the Tailscale control plane (`PUT /api/v2/tailnet/{tailnet}/services/svc:{name}`)
6. **Advertise** — call `host.Advertise` on the tsnet node for each (service, port) pair; deregister pairs that disappeared since last cycle

### Package layout

| Package | Role |
|---|---|
| `cmd/tsaws` | Binary entrypoint, reconciler loop, dry-run mode |
| `internal/config` | Load and validate all config from env vars at startup |
| `internal/discovery/route53` | List PHZ records via `route53:ListResourceRecordSets` |
| `internal/discovery/filter` | Zone opt-in, glob, and resource-tag eligibility filter |
| `internal/discovery/managed` | Detect managed AWS endpoints by DNS suffix; enrich with port/metadata via AWS describe APIs |
| `internal/discovery/probe` | TCP port probe as last-resort port derivation |
| `internal/translate` | Map a Route 53 record + managed metadata to `ServiceSpec` (name, ports, FQDN) |
| `internal/registrar` | Tailscale Services API client: OAuth token exchange, GET-before-PUT upsert, List for dry-run diff |
| `internal/approver` | Policy-file `autoApprovers.services` reader (for bootstrap validation only) |
| `internal/envinfer` | AWS and runtime environment inference at startup: IMDSv2, EC2 describe, ECS/EKS metadata; assembles Connector node tag list (R40) |
| `internal/host` | tsnet node lifecycle: join tailnet, `Advertise`/`Deregister` service routes |
| `internal/status` | In-memory state machine per FQDN; HTTP endpoint at `TSAWS_STATUS_ADDR` |
| `internal/events` | Structured `log/slog` event emission; in-memory ring buffers for portal event and cycle history |
| `internal/portal` | Read-only admin web UI served over tailnet: status, events, cycles, config, health, traffic, runtime |
| `internal/health` | Per-service L4/L7 health engine: probe goroutines, threshold tracking, advertisement gating |
| `internal/imds` | EC2 IMDSv2 client for baseline identity tag population (AZ, VPC ID, account ID) |
| `internal/lifecycle` | Bootstrap validation: STS credentials, Route 53 IAM, OAuth token, zone existence |
| `internal/ratelimit` | Token-bucket rate limiter for Tailscale API calls; exponential backoff on 429 |

### Key design decisions

- **One FQDN = one Service.** Aurora writer and reader endpoints produce separate Services.
- **TCP only.** No L7, no protocol parsing, no credential injection. TLS passthrough for managed services.
- **Read-only IAM.** No `route53:Change*`, no EC2 mutations. Ever.
- **Port derivation order:** SRV record value → `tailscale:port` AWS tag → managed service enrichment → TCP probe → `TSAWS_DEFAULT_PORT` → `do-not-validate` sentinel
- **Service name:** flattened FQDN minus zone root, dots replaced with hyphens, lowercased. Collision: append 6-char hash.
- **OAuth:** client secret exchanged for access token via `client_credentials` grant; token cached with 30s expiry buffer.
- **GET-before-PUT:** registrar fetches existing service addrs before every PUT to satisfy the Tailscale API constraint that updates must include existing IPv4+IPv6 addrs.
- **Deregistration:** set-diff of `(service, port)` pairs between reconcile cycles; stale pairs are deregistered from the tsnet host.

## Configuration (env vars)

The full reference is in `docs/connector-aws.md`. Key variables:

| Variable | Default | Purpose |
|---|---|---|
| `TS_CLIENT_SECRET` | required | Tailscale OAuth client secret |
| `TSAWS_HOSTED_ZONE_IDS` | required | Comma-separated Route 53 PHZ IDs to discover |
| `TSAWS_CONNECTOR_TAG` | `tag:tsaws` | Tailscale host tag for the tsnet node |
| `TSAWS_SERVICE_TAG` | — | Single Tailscale ACL tag applied to every registered Service; falls back to `TSAWS_CONNECTOR_TAG` when empty |
| `TSAWS_TAILNET` | `-` | Tailnet name (`-` = default for the OAuth client) |
| `TSAWS_ZONE_OPT_IN` | `false` | All records in allowlisted zones are eligible |
| `TSAWS_GLOBS` | `*` | Comma-separated DNS glob patterns for eligibility |
| `TSAWS_RECONCILE_INTERVAL` | `5m` | Reconcile loop cadence |
| `TSAWS_DRY_RUN` | `false` | Print discovered services without writing |
| `TSAWS_PORTAL_ENABLED` | `true` | Serve admin portal as a Tailscale Service on port 443 |
| `TSAWS_HEALTH_MODE` | `l4_tcp` | Health check protocol: `l4_tcp`, `l7_http`, or `none` |
| `TSAWS_CONNECTOR_REGION_TAG_ENABLED` | `true` | Apply `tag:aws-region-<slug>` to Connector node (default on) |
| `TSAWS_CONNECTOR_CUSTOM_TAGS` | — | Comma-separated additional tags applied to Connector node |

## Deployment

- **Container:** distroless static image, multi-stage build (`golang:1.26-alpine` → `gcr.io/distroless/static-debian12`), ARM64. Published to `ghcr.io/radioboxtv/tsaws:latest` on every push to main.
- **Terraform module:** `deploy/terraform/` — ECS cluster (optional), Fargate task (ARM64), security group, IAM task + execution roles, CloudWatch log group
- **CloudFormation:** `deploy/cloudformation/tsaws.yaml` — equivalent stack
- **Network:** private subnet + NAT gateway (current default). Peer Relay options (no NAT gateway required) are planned for M3; see `docs/connector-aws.md` for topology options.

## M1 implementation status

| Requirement | Status |
|---|---|
| R1: Deployment artifacts | Done — Dockerfile, Terraform, CloudFormation, CI publish job |
| R2: Route 53 discovery | Done |
| R4: Discovery filtering | Done — zone opt-in, globs, resource tags |
| R5: Service naming | Done |
| R6: Transport and ports | Done |
| R7: Service tags | Done — `TSAWS_SERVICE_TAG` applied to every registered Service |
| R40: Connector node tags | Done — region, AZ, VPC, subnet, account, cluster, deployment from IMDSv2/EC2/ECS/EKS; all gates default on |
| R8: Service description | Done |
| R9: Service idempotency | Done |
| R10: Service Host registration | Done |
| R11: Service Host deregistration | Done |
| R12: Managed service handling | Done — RDS, Aurora, ElastiCache, MemoryDB, MSK, DocumentDB, OpenSearch, Neptune |
| R13: Notifications / events | Done — slog JSON events + in-memory ring buffer (last 500 events, last 20 cycles) |
| R21: Service approval | Done — operator configures `autoApprovers.services` in policy file; Connector does not write policy file |
| R22: Dry-run mode | Done — structured JSON output with would-create/would-update/would-skip/orphan diff |
| R28: Service cap enforcement | Done — `TSAWS_MAX_SERVICES`, cap-exceeded state |
| R29: Health engine | Done — L4 TCP and L7 HTTP probes, threshold-based advertisement gating |
| R30: Admin portal | Done — tailnet-served UI: status, events, cycles, config, health, traffic, runtime |
| R32: Bootstrap validation | Done — STS, Route 53 IAM, OAuth token, zone existence, identity tag declarations |
| R33: Rate limiting | Done — token bucket for Tailscale API, configurable RPS/burst/policy-interval |

## Known gaps and open issues

- **#88** — Peer Relay support (no NAT gateway required). Planned for M3.
- M2+ work (HA, auto-deletion, drift reconciliation) and M3+ (Cloud Map, multi-region, peer relay) are tracked as open issues but out of scope for M1.

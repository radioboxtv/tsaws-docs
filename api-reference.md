# API reference

The connector exposes two HTTP surfaces:

- **Admin portal** — served as a Tailscale Service over the tailnet on port 443. Gated by an ACL grant on `tag:tsaws-admin-portal`. All `/api/*` endpoints below are on the portal.
- **Local status endpoint** — served on `TSAWS_STATUS_ADDR` (default `:8080`) on the connector's VPC interface. In-VPC reachability only; not exposed over the tailnet. Useful for ECS healthchecks and Prometheus-style scrapers running inside the VPC.

The portal also serves an HTML SPA at `GET /` and embedded static assets under `/assets/*`.

Outbound webhook delivery (when `TSAWS_WEBHOOK_URL` is set) is documented in [Webhook payload](#webhook-payload).

## Authentication

The portal has no application-layer authentication. Tailnet ACL grants on `tag:tsaws-admin-portal` are the only access control. Anyone whose grants reach the portal Service can call any endpoint.

The local `:8080` status endpoint has no authentication. It is reachable only inside the VPC and exposes the same structure as `GET /api/status`. Restrict via security group if you don't want it visible to other workloads in the VPC.

## Status and discovery

### `GET /api/status`

Every discovered FQDN with its current state.

```jsonc
{
  "services": [
    {
      "fqdn": "api.prod.example.internal.",
      "service_name": "svc:api-prod",
      "state": "registered",
      "ports": ["tcp:443"],
      "tags": ["tag:tsaws-service", "tag:env-prod"],
      "host_advertised": true,
      "health": "healthy"
    }
  ]
}
```

States: `eligible`, `registered`, `degraded`, `ineligible`, `deleted-from-source`, `orphaned`, `cap-exceeded`.

### `GET /api/discovery`

Per-Route-53-zone breakdown with per-record decisions and reason counts from the most recent reconcile cycle.

```jsonc
{
  "zones": [
    {
      "id": "Z1234567890ABC",
      "records_total": 42,
      "eligible": 12,
      "ineligible_reasons": { "no-glob-match": 27, "in-blocklist": 3 },
      "records": [
        { "fqdn": "api.prod.example.internal.", "decision": "eligible" },
        { "fqdn": "internal.example.internal.", "decision": "ineligible", "reason": "no-glob-match" }
      ]
    }
  ]
}
```

Use this when a record you expect to register isn't appearing: the per-record reason field tells you which rule it failed.

### `GET /api/cycles`

The last 20 reconciliation cycles.

```jsonc
{
  "cycles": [
    {
      "correlation_id": "01HX...",
      "started_at": "2026-04-30T12:00:00Z",
      "duration_ms": 1283,
      "created": 0,
      "updated": 1,
      "reused": 11,
      "failed": 0
    }
  ]
}
```

### `GET /api/events`

The event ring (last 500 events, newest first).

Query parameters:

- `kind` — filter by event kind (e.g. `?kind=ServiceCreated`)
- `correlation_id` — filter to a single reconcile cycle (e.g. `?correlation_id=01HX...`)

Event kinds include `ServiceCreated`, `ServiceUpdated`, `ServiceReused`, `ServiceHostRegistered`, `HealthCheckFailed`, `HealthCheckRecovered`, `ReconciliationCompleted`, `CapExceeded`, `NodeAttrFetchFailed`, `ConfigChangedFromNodeAttr`.

### `GET /api/health`

Connectivity health from the connector to its dependencies.

```jsonc
{
  "tailscale": { "controlplane": "ok", "login": "ok", "log": "ok" },
  "targets": [
    { "name": "route53", "host": "route53.amazonaws.com", "port": 443, "status": "ok", "latency_ms": 18 }
  ]
}
```

Operator-supplied probe targets configured via env are listed alongside Tailscale's.

### `GET /api/traffic`

Per-Service byte counters for both TCP and UDP traffic.

```jsonc
{
  "services": {
    "svc:api-prod": { "tcp_bytes_in": 12345, "tcp_bytes_out": 67890, "udp_bytes_in": 0, "udp_bytes_out": 0 }
  }
}
```

### `GET /api/runtime`

Go runtime metrics: goroutine count, heap size, build info, version, uptime.

### `GET /api/ratelimit`

Token-bucket rate limiter stats for the Tailscale, AWS (non-Route 53), and Route 53 APIs.

```jsonc
{
  "tailscale": { "tokens_available": 18, "tokens_per_second": 10, "burst": 20, "throttled_count": 2 },
  "aws":       { "tokens_available": 95, "tokens_per_second": 50, "burst": 100, "throttled_count": 0 },
  "route53":   { "tokens_available": 9,  "tokens_per_second": 5,  "burst": 10,  "throttled_count": 1 }
}
```

## Configuration

### `GET /api/config`

The effective connector configuration. OAuth secret and secret source are redacted.

### `POST /api/config/runtime`

Apply a partial runtime override. Request body is a JSON object using the same field names as the node-attribute cap. Validation is atomic: if any field is invalid, the whole update is rejected.

```jsonc
{
  "refresh_rate": "30s",
  "max_services": 100
}
```

If a `tsaws.com/config` cap is also active, the next NetMap push will overwrite portal-set values. The portal Config page shows a banner when the cap is active.

### `POST /api/config/connector-hostname`

Override the tsnet hostname. Triggers a tsnet host rebuild and brief tailnet disconnection (typically under 2 seconds). Body:

```jsonc
{ "hostname": "tsaws-prod-customer-vpc" }
```

### `GET /api/tags`

Per-tag desired vs current state for the connector node, service marker, and admin portal tags.

```jsonc
{
  "connector": [
    { "tag": "tag:tsaws", "applied": true, "owner_declared": true },
    { "tag": "tag:aws-region-us-east-1", "applied": true, "owner_declared": true },
    { "tag": "tag:env-prod", "applied": false, "owner_declared": false, "reason": "no tagOwners entry" }
  ]
}
```

### `GET /api/policy-snippets`

HuJSON repair snippets for any tag the connector found undeclared in the policy file. Useful for filling out `tagOwners` after a first deploy.

```jsonc
{
  "tagOwners": {
    "tag:env-prod": ["tag:tsaws"]
  }
}
```

### `POST /api/tags/retry`

Re-check tag availability against the latest policy file and re-run the reconciler. Use after editing `tagOwners` to apply tags that were previously rejected, without waiting for the next NetMap push.

## Per-Service health

### `GET /api/services/{name}/health-check`

Get the per-Service health configuration (mode, path, drain state).

### `PUT /api/services/{name}/health-check`

Update the per-Service health configuration. Body:

```jsonc
{ "mode": "redis" }
```

Valid modes: `l4_tcp`, `l7_http`, `none`, `redis`, `postgres`, `mysql`, `kafka`, `mongodb`, `memcached`, `opensearch`, `auto`. For `l7_http`, optionally include `"path": "/healthz"`.

The change is pushed to the running probe goroutine without re-registration; takes effect on the next probe tick.

### `DELETE /api/services/{name}/health-check`

Remove the per-Service override. The Service reverts to the global `TSAWS_HEALTH_MODE` (or AutoDetect / AutoProtocol selection if those are enabled).

### `GET /api/services/{name}/health-history`

Rolling probe history for a single Service. Returns latency and success values over the recent history window for charting.

### `POST /api/services/{name}/health-check/drain`

Force a Service unhealthy to drain it from rotation. The host advertisement is withdrawn after `unhealthy_threshold` failures (default 3); during the drain window, in-flight TCP connections are not force-closed.

### `DELETE /api/services/{name}/health-check/drain`

Restore the Service to active. Probes resume; the host advertisement re-establishes after `healthy_threshold` successes.

### `GET /api/services/{name}/probe-detect`

Run AutoL7 detection on demand. Returns per-port `is_http`, `reason`, and `cached` verdicts. Useful for verifying that a port that should be HTTP is actually being detected as HTTP before flipping `TSAWS_HEALTH_MODE`.

### `GET /api/services/{name}/detect-protocols`

Run every supported protocol probe in port-aware preference order. Returns one entry per port with all per-protocol verdicts.

```jsonc
{
  "ports": [
    {
      "port": 5432,
      "verdicts": {
        "postgres": { "ok": true, "latency_ms": 14 },
        "redis":    { "ok": false, "reason": "timeout" }
      },
      "recommended": "postgres"
    }
  ]
}
```

## Local status endpoint

`GET /` on `TSAWS_STATUS_ADDR` (default `:8080`).

Returns the same JSON structure as portal `GET /api/status`, but reachable only from within the VPC. No tailnet, no auth. Use for ECS task health checks or internal Prometheus scraping.

```bash
curl http://<task-ip>:8080/
```

## Webhook payload

When `TSAWS_WEBHOOK_URL` is set, every event in the ring is POSTed as JSON to that URL. When `TSAWS_WEBHOOK_SECRET` is also set, payloads are signed with HMAC-SHA256 and the signature is delivered in the `X-Tsaws-Signature` header.

```
POST <TSAWS_WEBHOOK_URL>
Content-Type: application/json
X-Tsaws-Signature: sha256=<hex>

{
  "kind": "ServiceCreated",
  "correlation_id": "01HX...",
  "timestamp": "2026-04-30T12:00:00.123Z",
  "service_name": "svc:api-prod",
  "fqdn": "api.prod.example.internal.",
  "ports": ["tcp:443"]
}
```

To verify the signature, compute `HMAC-SHA256(TSAWS_WEBHOOK_SECRET, raw_body)` and compare hex-encoded to the value after the `sha256=` prefix.

The connector retries failed deliveries with exponential backoff. If the receiver is down, events accumulate in memory only up to the ring size (500); older events are dropped.

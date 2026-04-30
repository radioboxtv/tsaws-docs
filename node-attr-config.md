# Node-attribute runtime config

The connector subscribes to the IPN bus and applies a Tailscale node-attribute
capability on every NetMap push. Six reconciler tunables and four tag-config
fields can be updated by ACL grant without redeploying the connector.

## Capability key

```
tsaws.com/config
```

The connector reads this exact key from `nm.SelfNode.CapMap()`. If multiple
grants target the connector node, the first one is used and a warn is logged
once per watcher session.

## Schema overview

The cap value is a JSON object. Every field is optional; absent fields leave
the existing override untouched. Empty values for the string tag fields are
treated as "not provided" by the cap (use the portal API to clear).

| Field | Type | Live? | Default |
|----|----|----|----|
| [`refresh_rate`](#refresh_rate) | string (Go duration) | yes | startup cfg |
| [`max_services`](#max_services) | int | yes | startup cfg |
| [`default_port`](#default_port) | uint16 | yes | startup cfg |
| [`port_blocklist`](#port_blocklist) | uint16[] | yes | startup cfg |
| [`domains`](#domains) | string[] | yes | startup cfg |
| [`discover_all_from_zone`](#discover_all_from_zone) | bool | yes | startup cfg |
| [`service_tag`](#service_tag) | string | yes | R35 fallback chain |
| [`custom_tags`](#custom_tags) | map[glob]→string[] | yes | none |
| [`connector_tag`](#connector_tag) | string | yes (rebuild) | OAuth-derived |
| [`portal_tag`](#portal_tag) | string | yes (rebuild) | OAuth-derived |

"Live? yes" = takes effect within one reconcile cycle of the NetMap push.
"Live? yes (rebuild)" = the connector tears down and re-establishes the
tsnet node before the new value takes effect. There is a brief
disconnection (typically under 2 seconds) for rebuild paths.

Validation is **skip-not-fail**: invalid fields are dropped from the update,
valid sibling fields still land, and a `NodeAttrFetchFailed` event records
the rejected field names. The connector keeps running on prior state.

## Field reference

### `refresh_rate`

How often the connector runs a reconciliation cycle: discover Route 53
records, evaluate eligibility filters, decide which Services to upsert
or remove, and re-advertise hosts. Lower values make the connector
react faster to AWS state changes; higher values reduce AWS API quota
spend and Tailscale control-plane traffic.

Type: string parsed by Go's `time.ParseDuration` (e.g., `"30s"`, `"5m"`,
`"1h30m"`). **Minimum: 1s**. Below that, validation rejects.

When this changes, the connector resets its ticker so the next cycle
fires `refresh_rate` from now (not from the original interval base). A
pending tick on the old interval is drained before the reset to avoid
an immediate stale fire.

### `max_services`

Cap on the total number of Services the connector will register. When
discovery yields more eligible records than this, the connector keeps
the lexicographically-first `max_services` and emits a warn for the
rest. Useful as a safety stop while you're tuning `domains`/zone
filters; not a long-term throttle.

Type: int. **0 means unlimited.** Negative values rejected.

### `default_port`

The TCP port assumed for an eligible record when neither the
`tailscale:ports` AWS resource tag nor a managed-service detector
yields one. Only used as a last resort.

Type: uint16. **0 means no fallback** (a record without an explicit port
will fail translation rather than getting an arbitrary port). Set to a
common port (e.g. `5432` for a Postgres-heavy environment) only if you
want every untagged record to be treated as that protocol.

### `port_blocklist`

Ports the connector will refuse to register, even if the resource tag
or managed-service detector requested them. The most common use is
defense-in-depth against accidentally exposing administrative ports
(e.g. SSH, RDP) over Tailscale.

Type: uint16[]. Each entry must be 1..65535. **Duplicates rejected**.
The default is empty (block nothing) since the resource tag is
operator-declared.

### `domains`

Per-Connector FQDN allowlist. A record's bare FQDN must match at least
one of these globs (`path.Match` semantics: `?` `*` `[abc]` `[!abc]`)
to pass the eligibility filter. An empty list means "no records pass
the glob filter alone" — typically combined with
`discover_all_from_zone: true` for zone-wide opt-in.

Type: string[]. Each glob must compile via `path.Match`. **Bad globs
rejected**, valid globs apply.

The empty list and `["*"]` are very different: the empty list disables
this filter entirely (no records pass on globs); `["*"]` matches every
record (subject to other filters).

### `discover_all_from_zone`

Switches the eligibility filter from "must match domains AND must have
the resource tag" to "all records from allowlisted hosted zones pass".
Useful when an operator has dedicated a zone to tsaws and doesn't
want to tag every record individually.

Type: bool. When `true`, the `domains` and `tailscale:include` resource
tag are bypassed for records sourced from one of `cfg.HostedZoneIDs`.
The connector still respects `port_blocklist`, `max_services`, and
managed-service detection.

Default: `false` (require explicit opt-in per record).

### `service_tag`

The single Tailscale identity tag applied to every new Service the
connector registers. When set, this **replaces the entire R35
greedy-with-fallback chain** for new upserts on the next reconcile
cycle: the connector skips the multi-level retry and writes services
under exactly this tag. Use to pin a custom service-level tag (e.g.
`tag:my-team-services`) without modifying the OAuth-derived chain.

Type: string. **Must start with `tag:`**. Empty (or absent) = use the
existing fallback chain unchanged.

This is **service-level**, not connector-level: the connector node's
own tag is unaffected. Existing Services keep their old tags until
the connector re-PUTs them (which happens automatically on the next
cycle since `EligibilityInputsChanged` fires and clears the
fingerprint cache).

### `custom_tags`

Per-FQDN-glob map of additional Service identity tags. Each glob is
matched against the discovered FQDN with `path.Match` semantics; on
match, every tag in the value list is appended (deduped) to the
service's identity tag set during the next Upsert.

Type: `map[string][]string`. Each glob key validated via
`path.Match`. Each tag value must start with `tag:`. **Bad globs or
malformed tags rejected**, valid entries apply.

Use `custom_tags` to scope ACLs by environment, owner, or compliance
boundary without changing the global service tag:

```jsonc
{
  "custom_tags": {
    "*.prod.example":    ["tag:env-prod", "tag:pii-allowed"],
    "*.staging.example": ["tag:env-staging"],
    "kafka.*.example":   ["tag:eventing"]
  }
}
```

A FQDN matched by multiple globs collects the union of all matching
tag lists, sorted+deduped so the upsert fingerprint stays stable across
NetMap pushes (no spurious diffs from map iteration order).

`service_tag` and `custom_tags` compose: the upserted tag set is
`{service_tag-or-fallback-chain-level} ∪ {custom_tags-matches}`.

### `connector_tag`

The connector node's own primary tsnet identity tag — the tag the
connector advertises on the tailnet, which other nodes (and the
service-host advertisement) see. This is what controls "who is this
connector?" in your ACL.

Type: string. **Must start with `tag:`**. Empty = use the
OAuth-derived tag chain.

**Live with rebuild**: when this transitions, the connector
gracefully shuts down its tsnet host and starts a new one with the
new tag. The Service Host advertisement, the portal Service, and the
health engine are all rebuilt. There is a brief tailnet disconnection
(typically under 2 seconds) during the swap, after which the connector
re-registers all of its Services.

A failure during rebuild (the new tag isn't grantable to this OAuth
client, or the new identity collides) leaves the old tsnet host
still running — the connector logs the error and keeps serving.

### `portal_tag`

The Tailscale identity tag applied to the **admin portal Service**
specifically (the connector's own embedded web UI). Lets operators
gate portal access independently of the data-plane services.

Type: string. **Must start with `tag:`**. Empty = use the
OAuth-derived portal tag chain.

**Live with rebuild**: setting this triggers the same tsnet rebuild
flow as `connector_tag`, since the portal Service is bound to the
connector's tsnet host. After the rebuild, the portal is re-registered
under its existing service name with the new tag.

## Precedence: cap vs portal POST

The portal `POST /api/config/runtime` endpoint and the IPN-bus cap
loop both write through the same `runtimeconfig.Overrides`. The cap is
the policy-file source of truth: it overwrites portal changes on every
NetMap push.

Practically this means: if an operator clicks "Save" on the portal
Config page, that value is live until the next NetMap push. If a cap
grant is also active, the next push overwrites the portal value.
Operators using the portal for live experiments should know that any
cap-driven field will revert. The portal Config page surfaces a banner
when the cap is active; see `internal/portal/assets/portal.js`.

To make a portal change durable: set the matching env var, Terraform
variable, CloudFormation parameter, or CLI flag and redeploy.

## Persistence

None. A connector restart re-seeds the overrides from `appconfig.Config`;
deployment-time configuration is the floor. The cap is the live
override layer; the portal is the live override layer; neither
persists across restart.

## Audit events

- `ConfigChangedFromNodeAttr` — fires when at least one field
  transitioned to a new value. Includes the changed field names plus
  prior and next snapshots. Will not fire on redundant NetMap pushes
  carrying the same value.
- `NodeAttrFetchFailed` — fires on decode error, validation rejection,
  or IPN-bus read failure. Non-fatal: connector keeps running on
  prior state. Reason string is human-readable
  (e.g. `"nodeattr validation: domains: invalid glob \"[\""`).

Both events surface in `/api/events` and the portal Events panel.

## Example grants

The cap is delivered via the `grants` block in your tailnet policy file.
Match `dst` to the connector node's tag.

### Tighten the refresh rate for a single connector

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "refresh_rate": "30s",
    }],
  },
},
```

### Restrict eligible records to a domain set

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "domains": ["*.prod.example", "*.staging.example"],
      "discover_all_from_zone": false,
    }],
  },
},
```

### Cap the connector at 50 services and block port 22

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "max_services": 50,
      "port_blocklist": [22, 3389],
      "default_port": 5432,
    }],
  },
},
```

### Differentiate prod and staging connectors with one grant block

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws-prod"],
  "app": {
    "tsaws.com/config": [{
      "refresh_rate": "1m",
      "max_services": 200,
    }],
  },
},
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws-staging"],
  "app": {
    "tsaws.com/config": [{
      "refresh_rate": "10s",
      "max_services": 25,
    }],
  },
},
```

### Pin a custom service-level tag

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "service_tag": "tag:platform-services",
    }],
  },
},
```

### Tag services by environment with custom_tags

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "custom_tags": {
        "*.prod.example":    ["tag:env-prod", "tag:pii-allowed"],
        "*.staging.example": ["tag:env-staging"],
        "*.dev.example":     ["tag:env-dev"]
      }
    }],
  },
},
```

The downstream effect: an ACL grant `tag:audit -> tag:pii-allowed:*`
suddenly applies to every service matching `*.prod.example` without
modifying every record's AWS `tailscale:tags` resource tag.

### Migrate the connector to a new identity tag

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "connector_tag": "tag:tsaws-2026q2",
      "portal_tag":    "tag:tsaws-portal-2026q2"
    }],
  },
},
```

When this lands, the connector rebuilds its tsnet host with the new
tag set. Brief disconnection (typically under 2 seconds), then the
connector re-registers all services and the portal under the new
identity. The audit event `ConfigChangedFromNodeAttr` records the
transition with `connector_tag` and `portal_tag` in the changed list.

## Wire-up surface

The cap is consumed in `internal/nodeattr` and routed to
`internal/runtimeconfig.Overrides`. The reconciler snapshots overrides
once per cycle and uses the snapshot for every read in that cycle, so
mid-cycle writes (cap or portal) take effect on the next cycle.

Three coalesced channels signal the main loop:

- `IntervalChanged`: ticker reset.
- `EligibilityInputsChanged`: fingerprint cache cleared so already-
  registered services re-evaluate against the new criteria. Fires on
  domains, discover_all_from_zone, default_port, port_blocklist,
  service_tag, custom_tags transitions.
- `IdentityChanged`: tsnet host rebuild. Fires on connector_tag or
  portal_tag transitions.

See also: `internal/nodeattr/nodeattr.go` for the decoder, validator,
and `Apply` entry point; `internal/runtimeconfig/runtimeconfig.go` for
the live state and channel semantics; `cmd/tsaws/main.go` for the
IPN-bus subscriber and the rebuild flow.

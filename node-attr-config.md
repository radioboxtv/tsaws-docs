# Node-attribute runtime config

The Connector subscribes to the IPN bus and applies a Tailscale node-attribute
capability on every NetMap push. Six reconciler tunables can be updated by
ACL grant without restarting the Connector.

## Capability key

```
tsaws.com/config
```

The Connector reads this exact key from `nm.SelfNode.CapMap()`. If multiple
grants target the Connector node, the first one is used and a warning is
logged once per watcher session.

## Schema

The cap value is a JSON object. All fields are optional; absent fields leave
the existing override untouched.

| Field | Type | Notes |
|----|----|----|
| `reconcile_interval` | string | Go duration (e.g. `"30s"`, `"5m"`). Min `1s`. |
| `max_services` | int | `0` = unlimited. |
| `default_port` | uint16 | `0` = no fallback. |
| `port_blocklist` | uint16[] | Each `1..65535`, no duplicates. |
| `globs` | string[] | FQDN allowlist; each entry must compile via `path.Match`. |
| `zone_opt_in` | bool | If `true`, all records from configured zones pass the filter. |

Validation is **skip-not-fail**: invalid fields are dropped from the update,
valid sibling fields still land, and a `NodeAttrFetchFailed` event records
the rejected field names. The Connector keeps running on prior state.

Tag-config fields (`connector_tag`, `service_tag`, `portal_tag`) are not in
the M1 schema. Applying a tag change requires rebuilding the tsnet identity
(disconnecting and reconnecting the node), so accepting them here without
applying them would mislead operators who'd see a successful audit event with
no behavior change. Tag fields will land alongside the rebuild path in a
later milestone.

## Persistence

None. A Connector restart re-seeds the overrides from the deployment-time
configuration (env vars, Terraform variables, CloudFormation parameters,
CLI flags) — those values are the floor. To make a change durable, set the
matching deployment-time value and restart.

## Precedence with the portal

The admin portal exposes the same tunables via `POST /api/config/runtime`.
Both the portal and the node-attribute cap write to the same underlying
overrides. When a cap grant is active, every NetMap push reapplies the cap
values, so a portal change to a field carried by the cap will be silently
overwritten on the next push. Treat the cap as the source of truth when
both are configured: use the portal for transient tweaks during incident
response, and the cap for the standing configuration.

## Audit events

- `ConfigChangedFromNodeAttr` — fires on actual transitions (not on
  redundant NetMap pushes carrying the same value). Includes the changed
  field names plus prior/next snapshots.
- `NodeAttrFetchFailed` — fires on decode error, validation rejection, or
  IPN-bus read failure. Non-fatal; the Connector continues with prior state.

Both surface in `/api/events` and the portal Events panel.

## Example grants

The cap is delivered via the `grants` block in your tailnet policy file.
Match `dst` to the Connector node's tag.

### Tighten the reconcile interval for a single Connector

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "reconcile_interval": "30s",
    }],
  },
},
```

### Restrict eligible records to a glob set

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws"],
  "app": {
    "tsaws.com/config": [{
      "globs": ["*.prod.example", "*.staging.example"],
      "zone_opt_in": false,
    }],
  },
},
```

### Cap the Connector at 50 services and block sensitive ports

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

### Differentiate prod and staging Connectors with the same grant block

```jsonc
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws-prod"],
  "app": {
    "tsaws.com/config": [{
      "reconcile_interval": "1m",
      "max_services": 200,
    }],
  },
},
{
  "src": ["autogroup:admin"],
  "dst": ["tag:tsaws-staging"],
  "app": {
    "tsaws.com/config": [{
      "reconcile_interval": "10s",
      "max_services": 25,
    }],
  },
},
```

## Operational notes

- Changes propagate sub-second via the IPN bus; no polling.
- The reconciler reads a per-cycle snapshot, so a cap update that lands
  mid-cycle takes effect on the next cycle — never partially.
- A reconcile-interval change resets the ticker (the next tick fires after
  the new interval, not after the old one).
- An eligibility-input change (`globs`, `zone_opt_in`, `default_port`,
  `port_blocklist`) clears the Connector's fingerprint cache so already-
  registered Services are re-evaluated on the next cycle.
- Revoking the cap grant does not roll back applied values — the override
  retains the last-applied value until the Connector restarts.

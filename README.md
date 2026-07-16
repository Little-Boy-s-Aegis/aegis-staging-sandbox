# Aegis Defensive Staging Sandbox

[![Git Clones](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/Little-Boy-s-Aegis/aegis-bank-deployment/main/aegis-staging-sandbox-clone-badge.json)](https://github.com/Little-Boy-s-Aegis/aegis-bank-deployment)
[![Unique Cloners](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/Little-Boy-s-Aegis/aegis-bank-deployment/main/aegis-staging-sandbox-uniques-badge.json)](https://github.com/Little-Boy-s-Aegis/aegis-bank-deployment)
[![Release Downloads](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/Little-Boy-s-Aegis/aegis-bank-deployment/main/downloads-badge.json)](https://github.com/Little-Boy-s-Aegis/aegis-bank-deployment/releases)
[![Stars](https://img.shields.io/github/stars/Little-Boy-s-Aegis/aegis-staging-sandbox?style=flat&color=F59E0B&logo=github&label=stars)](https://github.com/Little-Boy-s-Aegis/aegis-staging-sandbox/stargazers)

Lightweight HTTP simulator for testing Aegis SOAR response playbooks without
changing real firewall, identity, or endpoint-security systems. It models a
small set of Fortinet, Microsoft Graph/Entra, CrowdStrike, and asset-inventory
interfaces and displays the in-memory state in a browser UI.

> The simulator is for local development and controlled integration tests. It
> is not a security appliance, identity provider, or production-grade mock
> server.

## What It Simulates

| System | Supported behavior |
|---|---|
| Fortinet FortiGate | Create/delete address objects and add them to address groups |
| Entra ID / Graph | Enable or disable a user account |
| CrowdStrike Falcon | Resolve a hostname to AID, contain a host, or lift containment |
| Asset inventory | Return a fixed list of critical IPs, hosts, and domains |
| AWS WAF | Display seeded WAF state in the UI; no WAF mutation endpoint is implemented here |

Every mutation updates process-local state and prepends an entry to the UI
action log. Restarting the process resets all changes.

## Endpoints

| Method | Path | Authentication | Purpose |
|---|---|---|---|
| `GET` | `/` | Basic auth | State dashboard |
| `GET` | `/api/v1/assets/critical` | none | Fixed critical-asset inventory |
| `POST` | `/api/v2/cmdb/firewall/address` | Fortinet token | Create address object |
| `POST` | `/api/v2/cmdb/firewall/addrgrp/{group}` | Fortinet token | Add group members |
| `DELETE` | `/api/v2/cmdb/firewall/address/{name}` | Fortinet token | Remove address object and memberships |
| `POST` | `/oauth2/token` | Client secret | CrowdStrike-style token response |
| `POST` | `/oauth2/v2.0/token` | Client secret | Entra-style token response |
| `GET` | `/devices/queries/devices/v1` | Bearer token | Resolve hostname to device AID |
| `POST` | `/devices/entities/devices-actions/v2` | Bearer token | `contain` or `lift` a device |
| `PATCH` | `/v1.0/users/{username}` | Bearer token | Set `accountEnabled` |

## Ports

Running `app.py` starts the same handler on two ports:

- `SANDBOX_PORT` (default `8095`) for the simulator and browser UI
- `8083` for the asset-inventory alias used by the SOAR policy evaluator

The standalone Dockerfile declares only `8095`, although both listeners run in
the container and peers on the same Docker network can address `8083`. Publish
`8083` explicitly only when host access is required.

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `SANDBOX_PORT` | `8095` | Primary HTTP listener |
| `AEGIS_SECURITY_SYNC_TOKEN` | `admin123` | Password for UI user `admin` |
| `FORTINET_API_TOKEN` | `aegis-fortinet-token-123456` | Fortinet `access_token` value |
| `CROWDSTRIKE_CLIENT_SECRET` | `mock-crowdstrike-client-secret` | Accepted OAuth client secret |
| `ENTRA_CLIENT_SECRET` | `mock-client-secret` | Accepted Entra client secret |

The source contains additional compatibility tokens for integration fixtures.
Do not treat any simulator token as secret or reuse it elsewhere.

## Run Locally

No third-party Python packages are required:

```bash
python3 app.py
```

Open <http://localhost:8095> and authenticate as `admin` with the configured
`AEGIS_SECURITY_SYNC_TOKEN`.

### Exercise a Fortinet action

```bash
curl -sS -X POST \
  'http://localhost:8095/api/v2/cmdb/firewall/address?access_token=aegis-fortinet-token-123456' \
  -H 'Content-Type: application/json' \
  -d '{"name":"blocked_ip_203_0_113_10","type":"ipmask","subnet":"203.0.113.10 255.255.255.255"}'
```

Refresh the browser UI to see the state and action log.

### Exercise CrowdStrike containment

```bash
curl -sS -X POST \
  'http://localhost:8095/devices/entities/devices-actions/v2?action_name=contain' \
  -H 'Authorization: Bearer simulated-token' \
  -H 'Content-Type: application/json' \
  -d '{"ids":["mock-aid-WEB-PROD-01"]}'
```

## Docker

```bash
docker build -t aegis-staging-sandbox .
docker run --rm -p 127.0.0.1:8095:8095 \
  -e AEGIS_SECURITY_SYNC_TOKEN='<local-token>' \
  aegis-staging-sandbox
```

When integrating this standalone repository with the SOAR engine, attach both
containers to the same network and set `ASSET_INVENTORY_API_URL` to this
container's port `8083` listener.

## Implementation Notes

- `app.py` uses only Python's standard library and a single-process in-memory
  state dictionary.
- `address_string()` returns the raw client IP to avoid slow reverse-DNS lookups
  in some Windows/container environments.
- The handler deliberately bypasses authentication when `SANDBOX_PORT` is not
  `8095` for integration fixtures. Do not bind an alternate-port instance to an
  untrusted network.
- There is no persistence, concurrency control, TLS, rate limiting, or complete
  vendor API fidelity.

## Validation

```bash
python3 -m py_compile app.py
docker build -t aegis-staging-sandbox .
```

For integration validation, exercise create, group membership, delete, identity
disable/enable, host contain/lift, invalid token, and unknown-route cases.

## Repository Layout

```text
app.py       # HTTP handlers, UI template, state, and listeners
Dockerfile   # Python 3.11 Alpine runtime
README.md
```

## Related Repositories

- [`aegis-soar-engine`](https://github.com/Little-Boy-s-Aegis/aegis-soar-engine) — playbook execution
- [`aegis-bank-deployment`](https://github.com/Little-Boy-s-Aegis/aegis-bank-deployment) — integrated network and secrets
- [`dashboard`](https://github.com/Little-Boy-s-Aegis/dashboard) — response visibility

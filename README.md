# Aegis Defensive Staging Sandbox (Defensive Simulator)

The **Aegis Defensive Staging Sandbox** is an API mock simulator that emulates enterprise cybersecurity infrastructure interfaces. It provides realistic HTTP endpoints for firewall blocks, active directory account status modifications, endpoint isolation (EDR), and WAF rules. 

This sandbox enables the **SOAR Engine (Layer 2)** to execute its automated playbooks and containment decisions in a safe staging sandbox without altering actual production cloud environments.

---

## Simulated Systems & Endpoints

The simulator mock-handles the following security systems:

### 1. Fortinet FortiGate Firewall
* **Create Address Object**: `POST /api/v2/cmdb/firewall/address`
  * Deploys custom subnets and blocks dynamically.
* **Group Management**: `POST /api/v2/cmdb/firewall/addrgrp/<group_name>`
  * Associates address objects with target rule groups (e.g., `Blocked_IPs_Group`).
* **Delete Address Object**: `DELETE /api/v2/cmdb/firewall/address/<name>`
  * Used during rollback procedures to undo IP blocks.

### 2. Active Directory / Entra ID (Graph API)
* **Account Status Modification**: `PATCH /v1.0/users/<username>`
  * Simulates enabling or disabling user accounts (e.g., setting `accountEnabled` to `true` or `false` in AD/Entra).

### 3. CrowdStrike Falcon EDR
* **Device Query**: `GET /devices/queries/devices/v1`
  * Matches hostnames to CrowdStrike Agent IDs (AIDs).
* **Host Isolation & Reconnection**: `POST /devices/entities/devices-actions/v2`
  * Performs containment actions (`action_name=contain` or `action_name=lift`) to isolate compromised systems.

### 4. AWS Web Application Firewall (WAFv2)
* **IP Block Lists**: Updates the simulated `AegisBlockedIPsSet` lists.

---

## Authentication Mechanism

All simulated interfaces enforce authentication checks mirroring production behaviors:
* **Staging Sandbox UI (`/`)**: Requires Basic Authentication (Username: `admin`, Password matches the `AEGIS_SECURITY_SYNC_TOKEN` environment variable).
* **Fortinet Endpoints**: Verifies the `access_token` query parameter against `FORTINET_API_TOKEN` (default: `mock-fortinet-token-123456`).
* **CrowdStrike & Entra ID Endpoints**: Require OAuth token resolution (`POST /oauth2/token` or `POST /oauth2/v2.0/token` validating secrets) and matching `Bearer` headers.

---

## Running the Sandbox

### Prerequisites
* **Python 3.11+**
* Docker Desktop (optional, for containerized running)

### Running on Host
By default, the application runs on port `8095`.

```bash
# Start the simulator
python app.py
```
To run on a custom port, set the `SANDBOX_PORT` environment variable:
```bash
$env:SANDBOX_PORT="8096" # PowerShell
python app.py
```

### Running with Docker
A pre-configured lightweight `Dockerfile` is provided:
```bash
# Build the simulator image
docker build -t aegis-staging-sandbox .

# Run the simulator container
docker run -d -p 8095:8095 --name aegis-sandbox-service aegis-staging-sandbox
```

---

## Performance Optimizations
* **DNS Resolution Bypass**: Under Windows and containerized environments, reverse DNS lookups on incoming request headers can introduce a 20-40 second delay per socket request. The sandbox overrides Python's default socket `address_string()` resolver to immediately return the raw client IP address, ensuring instantaneous API responses during heavy load-testing.

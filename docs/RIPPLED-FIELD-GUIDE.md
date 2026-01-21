# **__THE RIPPLED FIELD GUIDE__**

*A practical companion to [Running an XRP Ledger Validator](https://xrpl.org/blog/2020/running-an-xrp-ledger-validator) from xrpl.org.*

That guide explains the **why**: how consensus works, what validators do, and why you'd want to run one. This guide covers the **how**: practical configuration, hard-won lessons, and operational insights compiled into a single reference.

The goal is simple. Get you from zero to a running, properly configured rippled node without hunting through GitHub issues, mailing lists, and scattered documentation. The recommendations here come from real-world validator operations, core developer input, and lessons learned the hard way.

---

# Deployment Roadmap

Start with the [README](../README.md) to understand the commitment. Then come back here for the implementation details. Running a rippled node or validator is a commitment. This section maps out the logical sequence so you don't miss critical steps or do things out of order.

### The First Decision: Stock Node or Validator?

| Type | Purpose | Commitment |
|------|---------|------------|
| **Stock Node** | Track the ledger, serve API requests, learn operations | Moderate - can tolerate some downtime |
| **Validator** | Participate in consensus, vote on network's future | High - 100% uptime expected, reputation at stake |

> **Recommendation:** Run a stock node first. Gain operational experience before converting to a validator. The official XRPL documentation advises this approach.

### Deployment Phases

**Phase 1: Infrastructure**

- Hardware selection (RAM, CPU, storage)
- Hosting decision (bare metal preferred)
- OS setup (Ubuntu 22.04+)
- Network/firewall configuration

*Field Guide Sections:* [Hardware Requirements](#hardware-requirements), [Hosting Decisions](#hosting-decisions)

**Phase 2: Security**

- Port concepts and configuration
- Key hierarchy understanding
- Key generation (OFFLINE)
- Token management

*Field Guide Sections:* [Port Configuration & Security](#port-configuration--security), [Validator Keys](#validator-keys)

**Phase 3: Identity** *(Validators Only)*

- Domain verification concepts and preparation
- Fee voting philosophy
- xrp-ledger.toml setup

*Field Guide Sections:* [Domain Verification](#domain-verification), [Fee Voting](#fee-voting)

**Phase 4: Installation & Configuration**

- Install rippled package
- Configure rippled.cfg
- Start service and sync

*Field Guide Sections:* [Node Sizing](#node-sizing), [Database Management](#database-management), [Network Configuration](#network-configuration), [Time Synchronization](#time-synchronization), [Operational Settings](#operational-settings), [Putting It All Together](#putting-it-all-together)

**Phase 5: Operations**

- Monitoring setup
- Maintenance procedures
- Upgrading rippled

*Field Guide Sections:* [Monitoring](#monitoring), [Maintenance](#maintenance), [Upgrading Rippled](#upgrading-rippled)

**Phase 6: Community & Reputation** *(Validators Only)*

- Register on validator directories
- Join community channels
- Build reputation (12+ months for UNL consideration)

*Field Guide Sections:* [Community Resources](#community-resources), [Directories](#directories), [Building Reputation](#building-reputation)

### Critical Dependencies

Some steps must happen in order. Getting this wrong causes problems:

| Do This First | Before This | Why |
|---------------|-------------|-----|
| Select hardware | Install rippled | Undersized hardware causes sync failures |
| Understand port security | Configure rippled.cfg | Misconfigured ports expose admin access |
| Generate keys (offline) | Generate token | Token is derived from master key |
| Install rippled | Generate validator keys | Keys tool comes with rippled package |
| Add token to config | Restart rippled | Config must be valid before restart |
| rippled running & synced | Domain verification | Attestation requires running validator |
| Deploy xrp-ledger.toml | Verify domain | TOML must be accessible for verification |
| Run stable for months | Apply for UNL | Publishers require proven track record |

### Common Mistakes (Wrong Order)

| Mistake | Consequence |
|---------|-------------|
| Generate keys on validator server | Master key exposed if server compromised |
| Skip stock node phase | Operational inexperience leads to downtime |
| Configure domain before token | Domain attestation won't match |
| Apply for UNL too early | Rejected - need 12+ months track record |
| Use low `online_delete` values | I/O storms degrade performance |
| Expose admin ports | Remote attacker can stop your server |

### Quick Reference: Stock Node vs Validator

| Aspect | Stock Node | Validator |
|--------|------------|-----------|
| Hardware | 16-32 GB RAM, SSD | 32-64 GB RAM, NVMe |
| `node_size` | medium/large | huge |
| `peer_private` | Not needed | Required (set to 1) |
| Admin ports | Localhost | Localhost only |
| Public WebSocket | Optional | Disabled |
| Validator keys | Not needed | Required (offline) |
| Domain verification | Not needed | Required for UNL |
| Expected `server_state` | `full` | `proposing` |

### Useful Tools

| Tool | Purpose |
|------|---------|
| [XRPL Node Configurator](https://xrplf.github.io/xrpl-node-configurator/) | Generate rippled.cfg interactively |
| [Domain Verifier](https://xrpl.org/resources/dev-tools/domain-verifier) | Verify domain attestation |
| [TOML Checker](https://xrpl.org/resources/dev-tools/xrp-ledger-toml-checker) | Validate xrp-ledger.toml |
| [XRPSCAN Validators](https://xrpscan.com/validators) | View validator registry |

**Sources:** [XRPL Installation Guide](https://xrpl.org/docs/infrastructure/installation), [Run rippled as a Validator](https://xrpl.org/docs/infrastructure/configuration/server-modes/run-rippled-as-a-validator), [Capacity Planning](https://xrpl.org/docs/infrastructure/installation/capacity-planning)

---

# Table of Contents

- [Deployment Roadmap](#deployment-roadmap)
- [Phase 1: Infrastructure](#phase-1-infrastructure)
  - [Hardware Requirements](#hardware-requirements)
  - [Hosting Decisions](#hosting-decisions)
- [Phase 2: Security](#phase-2-security)
  - [Port Configuration & Security](#port-configuration--security)
  - [Validator Keys](#validator-keys)
- [Phase 3: Identity](#phase-3-identity)
  - [Domain Verification](#domain-verification)
  - [Fee Voting](#fee-voting)
- [Phase 4: Installation & Configuration](#phase-4-installation--configuration)
  - [Node Sizing](#node-sizing)
  - [Database Management](#database-management)
  - [Network Configuration](#network-configuration)
  - [Time Synchronization](#time-synchronization)
  - [Operational Settings](#operational-settings)
  - [Putting It All Together](#putting-it-all-together)
- [Phase 5: Operations](#phase-5-operations)
  - [Monitoring](#monitoring)
  - [Maintenance](#maintenance)
  - [Upgrading Rippled](#upgrading-rippled)
- [Phase 6: Community & Reputation](#phase-6-community--reputation)
  - [Community Resources](#community-resources)
  - [Directories](#directories)
  - [Building Reputation](#building-reputation)
- [Contributing](#contributing)

---

# Phase 1: Infrastructure

Before installing rippled, you need the right foundation. This phase covers hardware selection, hosting decisions, and basic network setup.

---

## Hardware Requirements

**The Bottom Line**

**Use x86-64 (Intel/AMD) processors for production.** ARM and Apple Silicon are not officially supported.

| Component | Minimum | Production Recommended |
|-----------|---------|------------------------|
| CPU | 64-bit x86-64, 4+ cores | 8+ cores, 3+ GHz |
| RAM | 16 GB | 32-64 GB |
| Disk | SSD, 50 GB | NVMe, 10,000+ sustained IOPS |
| Network | Gigabit | Gigabit, low latency |

**CPU Architecture**

| Architecture | Examples | Production Support |
|--------------|----------|-------------------|
| x86-64 (AMD64) | Intel Core, AMD Ryzen, Xeon, EPYC | Fully supported |
| ARM | Apple Silicon (M1/M2/M3/M4), AWS Graviton, Raspberry Pi | Development only |

> **Note:** AMD Ryzen and Intel processors use the same x86-64 instruction set and are equally supported. ARM-based processors (including Apple Silicon) can compile rippled for development, but are **not recommended for production validators or nodes**.

**Disk Performance**

Storage speed is critical. Requirements:

- **Type**: SSD or NVMe (spinning disks will not work)
- **IOPS**: 10,000+ sustained (not burst)
- **Capacity**: 50 GB minimum for database partition

**Source**

See [XRPL System Requirements](https://xrpl.org/docs/infrastructure/installation/system-requirements) for official specifications.

---

## Hosting Decisions

**The Bottom Line**

**Prefer bare metal over cloud.** Hyperscalers (AWS, Azure, OCI) introduce multiple issues for validators:

- **Noisy neighbors**: Shared physical hardware means other tenants can spike resource usage, throttling your validator during critical consensus moments
- **Network bandwidth limits**: Cloud VMs have baseline bandwidth with burst credits that deplete under sustained load. For example, AWS c5.large has only 750 Mbps baseline but advertises "up to 10 Gbps" - that burst lasts just 5 minutes before throttling kicks in
- **Storage I/O throttling**: EBS and equivalent block storage have IOPS burst credits. EC2 instances also have aggregate EBS bandwidth limits (e.g., r6i.xlarge drops from 40K to 6K IOPS after 30 minutes of sustained load)
- **Microburst penalties**: Short spikes in demand trigger throttling even when average utilization appears low - CloudWatch metrics aren't granular enough to detect millisecond-level bursts

**If Cloud is Unavoidable**

Use instances with local NVMe storage (AWS i3/i4 series, Azure Lsv2, OCI Dense I/O) and dedicated network bandwidth ("n" suffix instances on AWS like C5n, M5n).

**OS Recommendation**

Ubuntu 22.04+ LTS is the most commonly used and well-tested platform for rippled.

---

# Phase 2: Security

Before you configure rippled, understand the security model. This phase covers port security and validator key management - concepts you need before writing your config file.

---

## Port Configuration & Security

Understanding port configuration is critical for validator security. The sentry architecture diagram below shows how a properly configured validator exposes only the peer port (51235) through its stock nodes, keeping admin ports locked to localhost.

<p align="center">
  <img src="../images/validator-sentry-architecture.png" alt="Validator Sentry Architecture" width="700">
</p>

### Port Types

rippled uses four types of ports, each serving a different purpose:

| Port | Default | Protocol | Purpose |
|------|---------|----------|---------|
| **Peer** | 51235 | peer | Node-to-node communication. The only port that should be publicly accessible. |
| **RPC Admin** | 5005 | http | Administrative JSON-RPC API. Privileged commands like `stop`, `validation_seed`. |
| **WebSocket Admin** | 6006 | ws | Administrative WebSocket API. Same privileged access as RPC. |
| **WebSocket Public** | 5006 | ws | Public WebSocket API for clients. Subscribe to streams, submit transactions. |

<p align="center">
  <img src="../images/validator-port-security.png" alt="Validator Port Security Zones" width="700">
</p>

**The Bottom Line**

**Only port 51235 (peer) should be exposed to the internet.** Admin ports must be restricted to trusted IPs. Public WebSocket is optional and should be disabled on validators unless you have a specific need.

### Understanding ip vs admin

This is the most commonly misunderstood part of rippled configuration. These two parameters do different things:

| Parameter | What It Does | Security Role |
|-----------|--------------|---------------|
| `ip` | Which network interface to bind to | Controls who can connect at the network level |
| `admin` | Which IPs can run privileged commands | Controls who can execute admin commands after connecting |

**The Dangerous Pattern**

```ini
# DON'T DO THIS - exposes admin to the internet
[port_rpc_admin_local]
port = 5005
ip = 0.0.0.0
admin = 0.0.0.0
protocol = http
```

With `admin = 0.0.0.0`, anyone who can reach port 5005 can run `stop`, dump your validator keys, or worse.

**The Safe Patterns**

**Pattern 1: Localhost only (simplest)**
```ini
[port_rpc_admin_local]
port = 5005
ip = 127.0.0.1
admin = 127.0.0.1
protocol = http
```
Only local processes can connect. Use this if monitoring runs on the same machine.

**Pattern 2: Docker-friendly (single host)**
```ini
[port_rpc_admin_local]
port = 5005
ip = 0.0.0.0
admin = 127.0.0.1, 172.17.0.0/16, 172.18.0.0/16
protocol = http
```
Binds to all interfaces but restricts admin commands to localhost and Docker bridge networks. Use this when monitoring runs in Docker containers on the same host.

**Pattern 3: Hardened multi-host**
```ini
[port_rpc_admin_local]
port = 5005
ip = 10.0.0.10
admin = 10.0.0.10, 10.0.0.20
protocol = http
```
Binds only to the private network interface. Admin restricted to the validator itself and a specific monitoring host. Use this for production validators with separate monitoring infrastructure.

### send_queue_limit

**The Bottom Line**

**100 for admin WebSockets, 500 for public WebSockets.**

**What It Does**

Controls how many outbound messages can queue per WebSocket connection before rippled starts dropping messages or disconnecting slow clients.

| Port Type | Recommended | Rationale |
|-----------|-------------|-----------|
| Admin WS | 100 | Local connections are fast |
| Public WS | 500 | Remote clients may be slower |

Higher values tolerate slower clients but use more memory per connection.

### Security Patterns

**Validator Configuration (recommended)**

Validators should minimize attack surface. Disable public WebSocket unless needed:

```ini
[port_rpc_admin_local]
port = 5005
ip = 127.0.0.1
admin = 127.0.0.1
protocol = http

[port_ws_admin_local]
port = 6006
ip = 127.0.0.1
admin = 127.0.0.1
protocol = ws
send_queue_limit = 100

[port_peer]
port = 51235
ip = 0.0.0.0
protocol = peer
```

No public WebSocket. Admin only on localhost. Only peer port exposed.

**Stock Node with Public API**

If you're running a public API node:

```ini
[port_rpc_admin_local]
port = 5005
ip = 127.0.0.1
admin = 127.0.0.1
protocol = http

[port_ws_admin_local]
port = 6006
ip = 127.0.0.1
admin = 127.0.0.1
protocol = ws
send_queue_limit = 100

[port_ws_public]
port = 5006
ip = 0.0.0.0
protocol = ws
send_queue_limit = 500

[port_peer]
port = 51235
ip = 0.0.0.0
protocol = peer
```

**Docker with External Monitoring**

When monitoring tools (like [XRPL Validator Dashboard](https://github.com/realgrapedrop/xrpl-validator-dashboard)) run in Docker:

```ini
[port_rpc_admin_local]
port = 5005
ip = 0.0.0.0
admin = 127.0.0.1, 172.17.0.0/16, 172.20.0.0/16
protocol = http

[port_ws_admin_local]
port = 6006
ip = 0.0.0.0
admin = 127.0.0.1, 172.17.0.0/16, 172.20.0.0/16
protocol = ws
send_queue_limit = 100

[port_peer]
port = 51235
ip = 0.0.0.0
protocol = peer
```

The `172.x.x.x` ranges cover Docker's default bridge networks. Adjust if you use custom Docker networks.

### Firewall Rules

Defense in depth. Even with correct `ip` and `admin` settings, use firewall rules:

```bash
# Allow peer protocol from anywhere
sudo ufw allow 51235/tcp

# Allow admin ports only from specific IPs (if needed)
sudo ufw allow from 10.0.0.20 to any port 5005
sudo ufw allow from 10.0.0.20 to any port 6006

# Deny admin ports from everywhere else (implicit with ufw, explicit with iptables)
```

**What to Expose**

| Port | Public Internet | Private Network | Localhost |
|------|-----------------|-----------------|-----------|
| 51235 (peer) | Yes | Yes | Yes |
| 5005 (RPC admin) | **Never** | If needed | Yes |
| 6006 (WS admin) | **Never** | If needed | Yes |
| 5006 (WS public) | Optional | Optional | Yes |

**Source**

For hardened multi-host architectures, see [Hardened Architecture Guide](https://github.com/realgrapedrop/xrpl-validator-dashboard/blob/main/docs/HARDENED_ARCHITECTURE.md).

---

## Validator Keys

Validator key management is the most critical security aspect of running a validator. Understanding the key hierarchy and keeping your master key offline protects your validator identity.

<p align="center">
  <img src="../images/validator-key-hierarchy.png" alt="Validator Key Hierarchy" width="600">
</p>

### Key Hierarchy

**The Bottom Line**

**Keep your master key offline. Always.** Use validator tokens for day-to-day operations.

The XRP Ledger uses a three-tier key architecture:

| Key Type | Purpose | Storage | Used For |
|----------|---------|---------|----------|
| **Master Key** | Permanent identity | Offline (encrypted USB, air-gapped) | Signing manifests and tokens |
| **Ephemeral Key** | Daily signing | Inside validator token | Signing validations |
| **Validator Token** | Authorization | rippled.cfg on server | Authorizes rippled to validate |

**How It Works**

1. Your **master key** defines your permanent public identity (the `nH...` address that appears on UNLs)
2. You use the master key to sign a **manifest** that delegates authority to an **ephemeral key**
3. The **validator token** contains the manifest plus the ephemeral private key
4. rippled uses the ephemeral key to sign validations during consensus
5. If the token is compromised, generate a new one - your identity survives

> **Security Model:** By keeping the master key offline, a server compromise only exposes the ephemeral key. You can rotate tokens without changing your validator identity or requiring UNL updates.

### Key Generation

**Generate Keys (Offline)**

On an air-gapped machine or secure workstation (not your validator):

```bash
validator-keys create_keys
```

Output:
```
Validator keys stored in ~/.ripple/validator-keys.json

This file should be stored securely and not shared.
```

**The validator-keys.json File**

```json
{
  "key_type": "ed25519",
  "public_key": "nHD9jtA9y1nWC2Fs1HeRkEisqV3iFpk12wHmHi3mQxQwUP1ywUKs",
  "secret_key": "paLsUUm9bRrvNBPpvJQ4nF7vdRTZyDNofGMMYs9EDeEKeNJa99q",
  "revoked": false,
  "token_sequence": 0
}
```

| Field | Description | Share? |
|-------|-------------|--------|
| `public_key` | Your validator's permanent identity | Yes - this is public |
| `secret_key` | Master private key | **NEVER** |
| `token_sequence` | Token counter (must always increase) | No |
| `revoked` | Whether key has been revoked | N/A |

**Generate a Token (Offline)**

```bash
validator-keys create_token --keyfile ~/.ripple/validator-keys.json
```

Output:
```
[validator_token]
eyJtYW5pZmVzdCI6IkFBQUFBVUZ4R0Z3S3RjdTlLS0E4Y0FRVEhWVGJkVG9...
```

Copy this token to your validator's `rippled.cfg`.

### Token Management

**When to Rotate Tokens**

- A sysadmin with config access leaves your organization
- You suspect the token may have been exposed
- After any security incident on the validator server
- Routine security hygiene (periodic rotation)

**How to Rotate**

1. Generate new token (offline):
   ```bash
   validator-keys create_token --keyfile ~/.ripple/validator-keys.json
   ```

2. Update rippled.cfg on validator with new `[validator_token]`

3. Restart rippled:
   ```bash
   sudo systemctl restart rippled
   ```

4. **Update your backup** of validator-keys.json (token_sequence incremented)

**What Happens on Rotation**

- New token invalidates all previous tokens automatically
- Other servers adapt automatically when they see the new manifest
- **No UNL updates required** - your master public key stays the same
- Your validator continues operating with the same identity

> **Critical:** Always update your backup after generating a token. The `token_sequence` must always increase. If you restore an old backup and generate a token with a lower sequence number, the network ignores it.

### Key Compromise Response

**Scenario A: Token Compromised (Server Breach)**

Severity: Recoverable

1. Generate new token immediately (from offline master key)
2. Update rippled.cfg
3. Restart rippled
4. New token invalidates the compromised one

**Impact:** None to others. Your identity survives.

**Scenario B: Master Key Compromised**

Severity: **Critical - Permanent Identity Loss**

1. Generate revocation (offline):
   ```bash
   validator-keys revoke_keys --keyfile ~/.ripple/validator-keys.json
   ```

2. Add to rippled.cfg:
   ```ini
   [validator_key_revocation]
   JP////9xIe0hvssbqmgzFH4/NDp1z...
   ```

3. Restart rippled
4. Generate entirely new keys
5. **Contact UNL publishers** - everyone must update to your new public key

**Impact:** Your old validator identity is permanently dead. Requires coordination with all parties who trust you.

**Comparison**

| Aspect | Token Compromise | Master Key Compromise |
|--------|------------------|----------------------|
| Retain identity? | Yes | No |
| Others must act? | No | Yes - all UNL updates |
| Recovery time | Minutes | Days/weeks |
| Preventable? | Minimize server exposure | Keep master key offline |

### Common Mistakes

**1. Storing Master Key on Validator**

If the server is compromised, attacker owns your identity forever.

**2. Not Backing Up Keys**

If you lose validator-keys.json, you lose your identity permanently. Store encrypted backups in multiple secure locations.

**3. Not Updating Backups After Token Generation**

If you restore an old backup with lower `token_sequence`, new tokens are ignored by the network.

**4. Using Legacy validation_seed**

The old `[validation_seed]` method has no token rotation capability. Migrate to `[validator_token]`.

**Command Reference**

```bash
# Generate new keys (OFFLINE)
validator-keys create_keys

# Generate token (OFFLINE)
validator-keys create_token --keyfile ~/.ripple/validator-keys.json

# Set domain for verification (OFFLINE)
validator-keys set_domain example.com --keyfile ~/.ripple/validator-keys.json

# Revoke keys permanently (OFFLINE)
validator-keys revoke_keys --keyfile ~/.ripple/validator-keys.json
```

**Source**

See [validator-keys-tool guide](https://github.com/ripple/validator-keys-tool/blob/master/doc/validator-keys-tool-guide.md) for complete documentation.

---

# Phase 3: Identity

For validators, identity matters. This phase covers domain verification and fee voting - concepts to understand before your validator goes live. Note that domain verification can only be completed after your validator is running (Phase 4), but you should understand the process and prepare your domain beforehand.

---

## Domain Verification

Domain verification establishes a cryptographic link between your validator and a domain you control. This is **required** for UNL consideration and helps the community identify legitimate validators.

<p align="center">
  <img src="../images/validator-domain-verification.png" alt="Validator Domain Verification Flow" width="650">
</p>

### xrp-ledger.toml

**The Bottom Line**

Host a `xrp-ledger.toml` file at `https://YOUR_DOMAIN/.well-known/xrp-ledger.toml` with your validator's public key and attestation.

**What It Does**

The `xrp-ledger.toml` file creates a two-way trust link:
1. Your validator announces "I belong to this domain" (via rippled config)
2. Your domain announces "This validator belongs to me" (via the TOML file)

Verification tools check both directions to confirm the link is legitimate.

**File Structure**

```toml
# xrp-ledger.toml

[METADATA]
modified = 2024-01-15T00:00:00.000Z

[[VALIDATORS]]
public_key = "nHUjYWSeiAepbkkrmwmQXFnohhr6Vy5oFK9Gk6TUF6MAT4qSiXSK"
attestation = "9A379A630BA161A3E7A0969E472FAA82C1542FF24C412250C86AAEF55209DD35..."
network = "main"
owner_country = "US"
server_country = "DE"
unl = "https://vl.ripple.com"

[[PRINCIPALS]]
name = "Your Name or Organization"
email = "validator@example.com"
```

**Field Reference**

| Field | Required | Description |
|-------|----------|-------------|
| `public_key` | Yes | Master public key (starts with `n`) |
| `attestation` | Yes | Hex signature linking validator to domain |
| `network` | Yes | `main`, `testnet`, or custom chain ID |
| `owner_country` | No | Two-letter ISO country code (owner jurisdiction) |
| `server_country` | No | Two-letter ISO country code (server location) |
| `unl` | No | URL of validator list you follow |

> **Note:** Use `[[VALIDATORS]]` with double brackets. This is TOML array-of-tables syntax. Single brackets `[VALIDATORS]` will fail.

### Common Mistakes

**1. Domain Mismatch (Most Common)**

The domain in your attestation must **exactly match** where the TOML is served:

| TOML Location | Domain Must Be |
|---------------|----------------|
| `https://example.com/.well-known/xrp-ledger.toml` | `example.com` |
| `https://www.example.com/.well-known/xrp-ledger.toml` | `www.example.com` |
| `https://validator.example.com/.well-known/xrp-ledger.toml` | `validator.example.com` |

If you run the `set_domain` command with `example.com` but serve the TOML from `www.example.com`, verification will fail.

**2. Missing CORS Headers**

Verification tools run in browsers and need CORS enabled. Without it, requests are blocked.

**Nginx:**
```nginx
location /.well-known/xrp-ledger.toml {
    default_type application/toml;
    add_header 'Access-Control-Allow-Origin' '*';
}
```

**Apache:**
```apache
<Location "/.well-known/xrp-ledger.toml">
    Header set Access-Control-Allow-Origin "*"
</Location>
```

**3. SSL/TLS Issues**

- Must use HTTPS (not HTTP)
- Must use a valid certificate from a recognized CA
- Self-signed certificates will fail
- Expired certificates will fail

**4. Wrong Path**

The path `/.well-known/xrp-ledger.toml` is case-sensitive. Common mistakes:
- `/.well-known/XRP-Ledger.toml` (wrong case)
- `/xrp-ledger.toml` (missing .well-known)
- `/.well_known/xrp-ledger.toml` (underscore instead of hyphen)

**5. Hosting TOML on Validator**

> **Security Warning:** Do NOT run a web server on your validator. Host the TOML file on a separate machine.

**6. Confusing Public Key with Token**

| Safe to Share | NEVER Share |
|---------------|-------------|
| `public_key` (starts with `n`) | `validator_token` |
| `attestation` | `validator-keys.json` contents |
| Contact info | Master private key |

### Step-by-Step Setup

**Prerequisites**
- Domain with HTTPS web server (not on your validator)
- Valid SSL certificate from a recognized CA
- Access to your `validator-keys.json` (stored offline)

**Step 1: Generate Attestation**

On a secure machine with your `validator-keys.json`:

```bash
validator-keys set_domain YOUR_DOMAIN.COM
```

Output:
```
The domain attestation for validator nHDG5CRU... is:

attestation="A59AB577E14A7BEC053752FBFE78C3DE..."

Update rippled.cfg with this validator token:

[validator_token]
eyJ2YWxpZGF0...
```

**Step 2: Create TOML File**

```toml
[METADATA]
modified = 2024-01-15T00:00:00.000Z

[[VALIDATORS]]
public_key = "nHDG5CRUHp17ShsEdRweMc7WsA4csiL7qEjdZbRVTr74wa5QyqoF"
attestation = "A59AB577E14A7BEC053752FBFE78C3DED6DCEC81A7C41DF1931BC61742BB4FAE..."
network = "main"
owner_country = "US"

[[PRINCIPALS]]
name = "Your Organization"
email = "validator@yourdomain.com"
```

**Step 3: Deploy TOML**

Upload to your web server at `/.well-known/xrp-ledger.toml`

**Step 4: Configure CORS**

Add CORS headers (see Nginx/Apache examples above).

**Step 5: Update rippled.cfg**

On your validator, update the token from Step 1:

```ini
[validator_token]
eyJ2YWxpZGF0aW9uX3NlY3JldF9r...
```

Restart rippled:
```bash
sudo systemctl restart rippled
```

**Step 6: Verify**

1. Browser check: Visit `https://YOUR_DOMAIN/.well-known/xrp-ledger.toml`
2. [TOML Checker](https://xrpl.org/resources/dev-tools/xrp-ledger-toml-checker)
3. [Domain Verifier](https://xrpl.org/resources/dev-tools/domain-verifier)

> **Note:** Third-party sites like [Bithomp](https://bithomp.com/en/domains) and [XRPSCAN](https://xrpscan.com/validators) may take up to 24 hours to reflect your verified domain.

**Source**

See [xrp-ledger.toml specification](https://xrpl.org/docs/references/xrp-ledger-toml) for complete field reference.

---

## Fee Voting

**The Bottom Line**

**Use the network defaults unless you have strong governance reasons to deviate.**

```ini
[voting]
reference_fee = 10
account_reserve = 1000000
owner_reserve = 200000
```

**What These Values Mean**

| Setting | Value | Human Readable | Purpose |
|---------|-------|----------------|---------|
| reference_fee | 10 | 0.00001 XRP | Base transaction cost |
| account_reserve | 1000000 | 1 XRP | Minimum balance to activate account |
| owner_reserve | 200000 | 0.2 XRP | Cost per owned object (trustline, offer, etc.) |

**How Fee Voting Works**

Validators vote on these values. The network uses the **median** of all validator votes. This means:

- Your vote alone won't change anything
- Changing network fees requires coordinated validator consensus
- Outlier votes are ignored

**Philosophy Behind Current Values**

- **reference_fee (10 drops)**: Low enough for normal use, high enough to make spam expensive at scale
- **account_reserve (1 XRP)**: Prevents ledger bloat from millions of empty accounts
- **owner_reserve (0.2 XRP)**: Makes users think twice before creating objects that live forever in the ledger

**When to Vote Differently**

Rarely. Fee changes are network governance decisions. Only deviate if:
- You're participating in a coordinated network-wide fee adjustment
- You have data showing current fees are causing problems
- You've discussed with other validators

---

# Phase 4: Installation & Configuration

Now that you understand the infrastructure, security, and identity requirements, it's time to install and configure rippled.

---

## Node Sizing

### node_size

**The Bottom Line**

**Match `node_size` to your available RAM.** This is the single most important sizing decision.

| Value | RAM Needed | Use Case |
|-------|------------|----------|
| tiny | 1-2 GB | Testing only |
| small | 4-8 GB | Light stock node |
| medium | 8-16 GB | Standard stock node |
| large | 16-32 GB | Busy stock node |
| huge | 32-64 GB | Validators, high-traffic nodes |

**What It Controls**

The `node_size` parameter is a macro that tunes multiple internal settings:

- Memory allocation for caches
- Number of worker threads
- Database buffer sizes
- Fetch pack sizes for historical data

**Why It Matters**

- **Undersized**: Node falls behind, misses ledgers, poor sync performance
- **Oversized**: Wastes memory, no performance benefit
- **Just right**: Smooth operation with efficient resource use

**Configuration**

```ini
[node_size]
huge
```

No other parameters needed - rippled auto-tunes everything else based on this value.

**Common Mistake**

Don't try to manually tune thread counts or cache sizes. rippled's auto-tuning based on `node_size` is well-tested. Manual overrides often make things worse.

---

## Database Management

### online_delete Tuning

**The Bottom Line**

**Set `online_delete` to at least 16384, preferably 32768.** The default value (512) causes I/O storms that hurt validator performance.

| Scenario | online_delete | Rationale |
|----------|---------------|-----------|
| Default (not recommended) | 512-2000 | Causes I/O storms |
| Minimum stable | 8192 | ~8 hours between deletes |
| **Recommended** | **16384-32768** | **14-36 hours between deletes** |
| Conservative | 50000+ | Days between deletes, if disk permits |

**The Problem: I/O Storms**

Even on high-spec hardware (150k IOPS NVMe, 80 cores, 128 GB RAM), validators experience dropped ledgers due to I/O storms triggered by low `online_delete` values. The pattern is distinctive:

**Sawtooth Pattern (Low online_delete = 512):**
```
IOPS
30k ┤        ╭─╮        ╭─╮        ╭─╮        ╭─╮
    │       ╱   ╲       ╱  ╲       ╱  ╲       ╱  ╲    ← Delete storms
15k ┤      ╱     ╲     ╱    ╲     ╱    ╲     ╱    ╲
    │     ╱       ╲   ╱      ╲   ╱      ╲   ╱      ╲
 0k ┼────╱─────────╲─╱────────╲─╱────────╲─╱────────╲──
    └────┴──────────┴──────────┴──────────┴──────────┴────► Time
         ~30min    ~30min    ~30min    ~30min
```

**Smooth Pattern (High online_delete = 16384+):**
```
IOPS
30k ┤
    │
15k ┤
    │  ────────────────────────────────────────────  ← Steady state
 0k ┼
    └──────────────────────────────────────────────────► Time
```

**Why NuDB Makes This Worse**

NuDB (rippled's ledger database) is optimized for append-mostly workloads:

- **Writes are fast:** Sequential appends are efficient
- **Deletes are expensive:** Requires compaction and reorganization
- **Frequent small deletes = worst case for NuDB**

By increasing `online_delete`, you allow NuDB to operate in its optimal mode (appending) for longer periods before incurring deletion overhead.

**The Rotation Mechanism**

rippled maintains two NuDB databases simultaneously:
- **"Writable" DB** - receives new/modified nodes
- **"Archive" DB** - the older database being phased out

During rotation, rippled copies all nodes that **weren't modified** since the last rotation from archive to writable. This copy operation is the expensive step.

**The math:**
- Lower `online_delete` → fewer ledgers between rotations → fewer transactions → fewer modified nodes
- Fewer modified nodes → **more nodes must be copied** during rotation
- More copying → I/O spikes → potential missed consensus rounds

**Extreme example:** If you rotated every ledger and only 6 accounts changed, you'd copy millions of unchanged nodes. With 32,768 ledgers between rotations, far more nodes have been naturally modified, so fewer need copying.

**Impact on Validator Performance**

| Metric | During Delete Storm | Normal Operation |
|--------|---------------------|------------------|
| I/O latency | Spikes to 10-100+ ms | 1-2 ms |
| Consensus participation | May miss rounds | Full participation |
| Agreement % | Drops periodically | Stable |
| server_state | May flicker | Steady proposing |

A validator can have perfect agreement scores most of the time but experience periodic drops during I/O storms.

**The Trade-off**

| online_delete | Disk Space | I/O Pattern | Delete Frequency |
|---------------|------------|-------------|------------------|
| 512 | ~250 MB | Sawtooth (bad) | Every 30 min |
| 2000 | ~1 GB | Spiky | Every 2 hours |
| 16384 | ~8-12 GB | Smooth | Every 14-18 hours |
| 32768 | ~16-24 GB | Very smooth | Every 36 hours |

**The trade-off is minimal:** Even at 16384, you're only using ~8-12 GB more disk space in exchange for dramatically smoother I/O and more stable validation.

**Configuration**

```ini
[node_db]
type=NuDB
path=/var/lib/rippled/db/nudb
online_delete=32768
advisory_delete=0

[ledger_history]
32768
```

> **Note:** `ledger_history` must be ≤ `online_delete`.

**Verifying the Change**

After changing `online_delete` and restarting rippled:

1. The first delete cycle will still occur at the previous threshold
2. The smooth I/O pattern becomes visible after ledger count exceeds the new threshold
3. Monitor your disk I/O - the sawtooth pattern should disappear

**Additional Tuning Parameters**

If you still experience issues, these parameters in `[node_db]` may help:
- `age_threshold_seconds` - minimum age before deletion eligible
- `recovery_wait_seconds` - delay before rotation resumes after interruption

**The One Rule**

**Never use low values to "save disk space."** You'll pay for it in I/O storms and degraded validator performance. Disk is cheap; validator reputation isn't.

**Source**

This issue was identified by [@shortthefomo](https://github.com/shortthefomo) and documented in [rippled issue #6202](https://github.com/XRPLF/rippled/issues/6202), with technical explanation from Ripple engineer [@ximinez](https://github.com/ximinez).

### advisory_delete

**The Bottom Line**

**Use `advisory_delete=0` (the default).** Let rippled manage deletion automatically.

**What It Does**

| Value | Behavior |
|-------|----------|
| 0 | rippled decides when to delete old ledgers (automatic) |
| 1 | rippled waits for an external signal before deleting |

**When to Use Each**

- **`0` (recommended)**: Standard deployments. rippled handles everything.
- **`1`**: Only if you have external tooling that needs to process ledger data before deletion, or you're running a specialized archival integration.

**Configuration**

```ini
[node_db]
type=NuDB
path=/var/lib/rippled/db/nudb
advisory_delete=0
online_delete=32768
```

---

## Network Configuration

### compression

**The Bottom Line**

**Always enable compression.** There's no good reason to disable it.

```ini
[compression]
true
```

**What It Does**

Compresses peer protocol messages using LZ4. Reduces bandwidth usage by 60-80% with negligible CPU overhead.

**When to Disable**

Almost never. The only scenario: you're on a local network with unlimited bandwidth and you're chasing microseconds of latency. Even then, the savings are marginal.

### peers_max

**The Bottom Line**

**21-30 for validators, 15-21 for stock nodes.**

| Node Type | Recommended | Notes |
|-----------|-------------|-------|
| Validator | 21-30 | Enough for reliable propagation |
| Stock node | 15-21 | Balance between connectivity and resources |
| Private/minimal | 10-15 | Minimum viable connectivity |

**The Trade-off**

- **More peers**: Faster transaction/ledger propagation, more redundancy if peers drop
- **Fewer peers**: Less bandwidth, CPU, and memory usage

**Why Validators Don't Need Huge Counts**

Validators receive transactions from the network and propagate validations. They don't need 100 peers - they need *enough* peers to stay reliably connected. 21 well-connected peers is plenty.

**Real-World Bandwidth**

For reference, a production validator running `node_size=huge` with `peers_max=21` generates approximately 3.5 TB of network traffic per month. Plan your hosting accordingly.

**Configuration**

```ini
[peers_max]
21
```

---

## Time Synchronization

### sntp_servers

**The Bottom Line**

**Configure multiple diverse time sources.** Consensus depends on accurate time.

```ini
[sntp_servers]
time.nist.gov
pool.ntp.org
time.cloudflare.com
time.google.com
```

**Why Multiple Servers**

- **Redundancy**: If one server is unreachable, others provide time
- **Cross-checking**: rippled can detect if one source is wrong
- **Low latency**: Different providers have different geographic coverage

**Good Server Choices**

| Server | Type | Notes |
|--------|------|-------|
| time.nist.gov | Government | US-based, authoritative |
| pool.ntp.org | Community pool | Anycast, global |
| time.cloudflare.com | CDN | Low latency globally |
| time.google.com | CDN | Smeared leap seconds |

**Why Time Matters**

The XRP Ledger consensus protocol uses timestamps. Clock drift can cause:
- Validation timing issues
- Transactions appearing to be in the future/past
- Sync problems with other validators

Your system should also run ntpd or chronyd for OS-level time sync.

---

## Operational Settings

### log_level

**The Bottom Line**

**Run `warning` in production.** Only increase for troubleshooting.

| Level | Verbosity | Disk Impact | Use Case |
|-------|-----------|-------------|----------|
| trace | Extreme | GB/hour | Deep debugging only |
| debug | High | Large | Development, troubleshooting |
| info | Medium | Moderate | Default (too verbose for production) |
| **warning** | **Low** | **Minimal** | **Production recommended** |
| error | Very low | Very low | Only problems |
| fatal | Minimal | Negligible | Almost nothing |

**Configuration**

Set via `[rpc_startup]` to apply at boot:

```ini
[rpc_startup]
{ "command": "log_level", "severity": "warning" }
```

**Runtime Adjustment**

Temporarily increase for debugging without restart:

```bash
rippled log_level debug
```

Then set back:

```bash
rippled log_level warning
```

**Why Not `info`?**

The default `info` level logs every ledger close, peer connection, and many routine operations. On a busy validator, this generates significant disk I/O. `warning` logs only things that might need attention.

### ssl_verify

**The Bottom Line**

**Always `1` in production.** Never disable SSL verification.

```ini
[ssl_verify]
1
```

**What It Does**

| Value | Behavior |
|-------|----------|
| 1 | Verify SSL certificates for outbound connections |
| 0 | Skip verification (INSECURE) |

**Why It Matters**

rippled makes HTTPS connections to validator list publishers (vl.ripple.com, unl.xrplf.org). SSL verification ensures:

- You're actually talking to the real publisher
- No man-in-the-middle can inject a malicious validator list
- Your node won't trust fake validators

**When `0` is Acceptable**

- Local development with self-signed certificates
- Isolated test networks
- Never in production

---

## Putting It All Together

This section provides a complete, production-ready `rippled.cfg` example for a validator. Use this as a starting point and adjust for your environment.

**Assumptions**

| Component | Specification |
|-----------|---------------|
| Server | Bare metal, dedicated |
| CPU | 8+ cores, x86-64 (Intel/AMD) |
| RAM | 64 GB |
| Storage | NVMe SSD, 500 GB, 10,000+ IOPS |
| Network | Gigabit, low latency |
| OS | Ubuntu 22.04+ LTS |
| Deployment | Native install (no Docker on validator) |
| Monitoring | Separate host or localhost (see note below) |

**Monitoring Architecture Note**

> For maximum security, run monitoring on a **separate host** from your validator. This eliminates Docker from the validator entirely, reducing attack surface and preventing resource contention during consensus. The validator exposes admin ports only to the monitoring host's private IP.
>
> For simpler setups, monitoring on localhost is acceptable. See [Hardened Architecture Guide](https://github.com/realgrapedrop/xrpl-validator-dashboard/blob/main/docs/HARDENED_ARCHITECTURE.md) for the multi-host approach.

**Complete Validator Configuration**

```ini
#-----------------------------------------------------------------------
# rippled.cfg - Production Validator Configuration
#-----------------------------------------------------------------------

#-----------------------------------------------------------------------
# Node Identity
#-----------------------------------------------------------------------

[node_size]
huge

[ledger_history]
32768

#-----------------------------------------------------------------------
# Database
#-----------------------------------------------------------------------

[node_db]
type=NuDB
path=/var/lib/rippled/db/nudb
online_delete=32768
advisory_delete=0

[database_path]
/var/lib/rippled/db

[shard_db]
type=NuDB
path=/var/lib/rippled/db/shards
max_historical_shards=1

#-----------------------------------------------------------------------
# Network - Peers
#-----------------------------------------------------------------------

[ips_fixed]
r.ripple.com 51235
zaphod.alloy.ee 51235

[peer_private]
1

[peers_max]
21

[compression]
true

#-----------------------------------------------------------------------
# Network - Ports
#-----------------------------------------------------------------------

[server]
port_rpc_admin_local
port_ws_admin_local
port_peer

[port_rpc_admin_local]
port = 5005
ip = 127.0.0.1
admin = 127.0.0.1
protocol = http

[port_ws_admin_local]
port = 6006
ip = 127.0.0.1
admin = 127.0.0.1
protocol = ws
send_queue_limit = 100

[port_peer]
port = 51235
ip = 0.0.0.0
protocol = peer

#-----------------------------------------------------------------------
# Validator Identity
#-----------------------------------------------------------------------

# Generate with: validator-keys create_keys
# Store validator-keys.json OFFLINE in a secure location
[validator_token]
# Paste your validator token here (generated from validator-keys create_token)

[validation_public_key]
# Your validator's public key (starts with nH...)

#-----------------------------------------------------------------------
# Trust - Validator Lists
#-----------------------------------------------------------------------

[validator_list_sites]
https://vl.ripple.com
https://unl.xrplf.org

[validator_list_keys]
ED2677ABFFD1B33AC6FBC3062B71F1E8397C1505E1C42C64D11AD1B28FF73F4734
EDA8BFE0E9C6B09E867B321B64E8E272C6E93A1EC02FDB5CB1070EA01F6C67CBC6

#-----------------------------------------------------------------------
# Voting
#-----------------------------------------------------------------------

[voting]
reference_fee = 10
account_reserve = 1000000
owner_reserve = 200000

#-----------------------------------------------------------------------
# Time Synchronization
#-----------------------------------------------------------------------

[sntp_servers]
time.nist.gov
pool.ntp.org
time.cloudflare.com
time.google.com

#-----------------------------------------------------------------------
# SSL
#-----------------------------------------------------------------------

[ssl_verify]
1

#-----------------------------------------------------------------------
# Logging
#-----------------------------------------------------------------------

[rpc_startup]
{ "command": "log_level", "severity": "warning" }

[debug_logfile]
/var/log/rippled/debug.log

#-----------------------------------------------------------------------
# Amendment Voting (optional - add amendments you want to veto)
#-----------------------------------------------------------------------

# [veto_amendments]
# Amendment_ID_Here

# [amendments]
# Amendment_ID_Here
```

**What This Configuration Does**

| Setting | Value | Why |
|---------|-------|-----|
| `node_size=huge` | 64 GB RAM allocation | Matches my hardware |
| `online_delete=32768` | ~36 hours between deletes | Prevents I/O storms |
| `peer_private=1` | Hides validator IP | Security best practice |
| `peers_max=21` | 21 peer connections | Sufficient for reliable propagation |
| `compression=true` | LZ4 compression | 60-80% bandwidth savings |
| Admin ports on `127.0.0.1` | Localhost only | No external admin access |
| No public WebSocket | Disabled | Validators don't need it |
| `ssl_verify=1` | Verify certificates | Prevents MITM on UNL fetches |
| `log_level=warning` | Minimal logging | Reduces disk I/O |

**Post-Configuration Checklist**

1. Generate validator keys: `validator-keys create_keys`
2. Store `validator-keys.json` offline (encrypted USB)
3. Generate token: `validator-keys create_token --keyfile validator-keys.json`
4. Paste token into config
5. Set file permissions: `chmod 600 /etc/opt/ripple/rippled.cfg`
6. Configure firewall: only allow port 51235 inbound
7. Set up domain verification via `.well-known/xrp-ledger.toml`
8. Start rippled and verify sync: `rippled server_info`
9. Monitor agreement percentage over time

**Adapting for Docker**

If running rippled in Docker with monitoring tools (like [XRPL Validator Dashboard](https://github.com/realgrapedrop/xrpl-validator-dashboard)) on the same host, you need to understand Docker networking to configure admin ports securely.

<p align="center">
  <img src="../images/validator-monitoring-docker.png" alt="Docker Networking for Validator Monitoring" width="650">
</p>

**Why `ip = 0.0.0.0` is Required**

Docker containers run in isolated network namespaces. Each container has its own `127.0.0.1` that other containers cannot reach:

| Binding | Result |
|---------|--------|
| `ip = 127.0.0.1` | Only processes *inside the rippled container* can connect. Monitoring container is blocked. |
| `ip = 0.0.0.0` | Listens on all container interfaces. Other containers on the same Docker network can connect. |

Since `127.0.0.1` inside the rippled container is not the same as `127.0.0.1` inside the monitoring container, you must use `0.0.0.0` for container-to-container communication.

**The Security Model**

With `ip = 0.0.0.0`, the `admin` parameter becomes your security control. It whitelists which IP addresses can execute privileged commands like `stop`, `validation_seed`, or `peers`.

**Basic Pattern (Subnet Whitelist)**

This allows any container on the Docker bridge networks:

```ini
[port_rpc_admin_local]
port = 5005
ip = 0.0.0.0
admin = 127.0.0.1, 172.17.0.0/16, 172.20.0.0/16
protocol = http

[port_ws_admin_local]
port = 6006
ip = 0.0.0.0
admin = 127.0.0.1, 172.17.0.0/16, 172.20.0.0/16
protocol = ws
send_queue_limit = 100
```

> **Risk:** Any container on those subnets can run admin commands. If another container is compromised, it has admin access to rippled.

**Secure Pattern (Static IP Whitelist)**

For better security, assign static IPs to your containers and whitelist only those specific addresses.

First, create a Docker Compose file with a custom network and static IPs:

```yaml
# docker-compose.yml
networks:
  validator-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/24

services:
  rippled:
    image: your-rippled-image
    networks:
      validator-net:
        ipv4_address: 172.28.0.10
    # ... other config

  vmagent:
    image: victoriametrics/vmagent
    networks:
      validator-net:
        ipv4_address: 172.28.0.20
    # ... other config

  grafana:
    image: grafana/grafana
    networks:
      validator-net:
        ipv4_address: 172.28.0.30
    # ... other config
```

Then configure rippled to allow only those specific IPs:

```ini
[port_rpc_admin_local]
port = 5005
ip = 0.0.0.0
admin = 127.0.0.1, 172.28.0.10, 172.28.0.20
protocol = http

[port_ws_admin_local]
port = 6006
ip = 0.0.0.0
admin = 127.0.0.1, 172.28.0.10, 172.28.0.20, 172.28.0.30
protocol = ws
send_queue_limit = 100
```

Now only the rippled container itself (172.28.0.10), vmagent (172.28.0.20), and grafana (172.28.0.30) can run admin commands. Any other container on the network is blocked.

**Why This Is More Secure**

| Pattern | Allowed Admin Access |
|---------|---------------------|
| Subnet whitelist (`172.17.0.0/16`) | ~65,000 potential IPs |
| Static IP whitelist | Only your specific containers (2-3 IPs) |

The static IP pattern follows the principle of least privilege.

**Important Notes**

1. **Static IPs require a user-defined network.** The default Docker bridge network doesn't support static IP assignment.

2. **Keep IPs in sync.** If you change a container's IP in docker-compose.yml, update rippled.cfg to match.

3. **Never publish admin ports.** Even with this configuration, don't use `-p 5005:5005` to expose admin ports to the host. Only publish port 51235 (peer protocol).

4. **Consider separate hosts.** For maximum security, run monitoring on a separate host entirely. See [Hardened Architecture Guide](https://github.com/realgrapedrop/xrpl-validator-dashboard/blob/main/docs/HARDENED_ARCHITECTURE.md).

**Resources**

- [Docker Networking Overview](https://docs.docker.com/network/)
- [Docker Compose Networking](https://docs.docker.com/compose/networking/)
- [Assign Static IP to Container](https://docs.docker.com/compose/compose-file/05-services/#ipv4_address-ipv6_address)

---

# Phase 5: Operations

Your validator is running. Now you need to keep it running well. This phase covers monitoring, ongoing maintenance, and upgrades.

---

## Monitoring

You can't fix what you can't see. Monitor:

- **Server state**: Should be `proposing` (validator) or `full` (stock node). Alert on `disconnected`, `connected`, or `syncing`.
- **Agreement percentage**: Your votes vs. consensus outcome.
- **Resource utilization**: CPU, memory, disk I/O, network.
- **Peer connections**: Healthy count, no sudden drops.
- **Ledger sync**: Are you keeping up with the network?

For a turnkey solution, the [XRPL Validator Dashboard](https://github.com/realgrapedrop/xrpl-validator-dashboard) provides full-service monitoring built specifically for validator operators. It tracks 40+ rippled metrics via WebSocket and HTTP APIs *plus* system metrics (CPU, memory, disk I/O, network). Includes pre-configured alerts for critical states and supports Discord, Slack, PagerDuty, and other notification channels. Deploys as a single Docker stack - no libraries or dependencies on your rippled machine. Can run on a separate machine entirely for better security (see [Hardened Architecture](https://github.com/realgrapedrop/xrpl-validator-dashboard/blob/main/docs/HARDENED_ARCHITECTURE.md)).

Prometheus + Grafana is an alternative for system metrics, but you'll need additional tooling to monitor rippled-specific metrics like server state, agreement percentage, or peer connections.

**Quick Diagnostics**

```bash
# Check server status
rippled server_info

# Health check
curl http://localhost:51235/health

# Check if amendment blocked
rippled server_info | grep amendment_blocked
```

---

## Maintenance

Regular maintenance prevents small issues from becoming big problems. This section covers log management, disk space, database health, backups, and disaster recovery.

### Log Management

**Log File Location**

By default, rippled writes logs to `/var/log/rippled/debug.log`. The verbosity is controlled by `log_level` in your config.

**Log Levels**

| Level | Verbosity | Production Use |
|-------|-----------|----------------|
| trace | Extreme | Never |
| debug | High | Troubleshooting only |
| info | Medium | Too verbose |
| **warning** | **Low** | **Recommended** |
| error | Very low | Minimal logging |

Set the log level in `rippled.cfg`:

```ini
[rpc_startup]
{ "command": "log_level", "severity": "warning" }
```

You can temporarily increase verbosity for debugging without restarting:

```bash
rippled log_level debug
# ... investigate issue ...
rippled log_level warning
```

**Log Rotation**

The official rippled packages for Ubuntu/Debian and CentOS/RHEL include a logrotate configuration at `/etc/logrotate.d/rippled`. If you need to customize it:

```
/var/log/rippled/*.log {
    daily
    minsize 200M
    rotate 7
    nocreate
    missingok
    notifempty
    compress
    compresscmd /usr/bin/nice
    compressoptions -n19 ionice -c3 gzip
    compressext .gz
    postrotate
        /opt/ripple/bin/rippled --conf /opt/ripple/etc/rippled.cfg logrotate
    endscript
}
```

The `postrotate` command tells rippled to close and reopen its log file. Adjust `minsize` and `rotate` based on your disk space and retention needs.

> **Note:** Ensure you have only one logrotate configuration for rippled. Multiple configurations can cause issues.

**Source:** [logrotate API reference](https://xrpl.org/docs/references/http-websocket-apis/admin-api-methods/logging-and-data-management-methods/logrotate)

### Disk Space Monitoring

**What Consumes Disk**

| Component | Location | Growth Rate | Notes |
|-----------|----------|-------------|-------|
| Ledger database (NuDB) | `/var/lib/rippled/db/nudb` | ~12 GB/day (full history) | Controlled by `online_delete` |
| SQLite databases | `/var/lib/rippled/db` | Minimal | wallet.db, transaction.db |
| Logs | `/var/log/rippled` | Depends on log level | Controlled by logrotate |

**Monitoring Commands**

```bash
# Check rippled data directory size
du -sh /var/lib/rippled/db/

# Check log directory size
du -sh /var/log/rippled/

# Check overall disk usage
df -h /var/lib/rippled
```

**The online_delete Trade-off**

As covered in [Database Management](#database-management), higher `online_delete` values (16384-32768) use more disk but prevent I/O storms. Monitor your disk usage and adjust accordingly. With `online_delete=32768`, expect 16-24 GB for the ledger database.

### Database Maintenance

**Ledger Cleaner**

rippled includes an asynchronous maintenance process called the Ledger Cleaner that can find and repair corruption in the ledger database.

```bash
# Check a specific ledger range for corruption
rippled ledger_cleaner check 75000000 75001000

# Repair corruption in a range
rippled ledger_cleaner repair 75000000 75001000

# Stop the cleaner
rippled ledger_cleaner stop
```

This is an admin command requiring access to admin ports.

**When Database Corruption Occurs**

Corruption can result from:
- Unexpected shutdowns or power loss
- Hardware failures (disk, memory)
- Running out of disk space during writes

**Recovery Options**

**Option 1: Test with fresh database paths**

If you have disk space, change the paths temporarily to test if corruption is the issue:

```ini
[node_db]
path=/var/lib/rippled/db/nudb_test

[database_path]
/var/lib/rippled/db_test
```

If the server syncs successfully with fresh paths, corruption was the issue.

**Option 2: Delete and re-download**

It's safe to delete rippled's databases - the server will re-download ledger history from the network:

```bash
# Stop rippled first
sudo systemctl stop rippled

# Remove databases (keep your config!)
sudo rm -rf /var/lib/rippled/db/nudb/*
sudo rm /var/lib/rippled/db/*.db

# Restart - server will sync from network
sudo systemctl start rippled
```

Re-syncing typically takes 5-15 minutes for recent ledger data.

> **Warning:** If you delete `wallet.db`, your node will generate a new identity. For stock nodes this is fine. For validators, your validator identity is in `rippled.cfg` (the token), not in `wallet.db`, so your validator identity is preserved.

**Source:** [rippled Server Doesn't Sync](https://xrpl.org/docs/infrastructure/troubleshooting/server-doesnt-sync)

### Validator Key Backups

**What to Back Up**

| File | Location | Contains | Backup Frequency |
|------|----------|----------|------------------|
| `validator-keys.json` | Offline storage | Master key, token sequence | After every token generation |
| `rippled.cfg` | `/etc/opt/ripple/rippled.cfg` | Current token, config | After config changes |

**Backup Best Practices**

1. **Store `validator-keys.json` on encrypted, offline media** (USB drive stored in a safe)
2. **Multiple copies in different physical locations** - fire, flood, theft protection
3. **Update backups after every token generation** - the `token_sequence` must always increase
4. **Never store master keys on the validator server**
5. **Test your backup restoration process** periodically

**The Critical Sequence Number Rule**

Each token contains a sequence number. The network only accepts tokens with a sequence number higher than any previously seen. If you:

1. Generate token (sequence 1)
2. Generate token (sequence 2)
3. Restore old backup (sequence 1)
4. Generate token (sequence 2 again)

...the network will ignore that token because it's not higher than sequence 2. **Always update your backup after generating tokens.**

**Source:** [validator-keys-tool guide](https://github.com/ripple/validator-keys-tool/blob/master/doc/validator-keys-tool-guide.md)

### Health Checks

**The Health Check Endpoint**

rippled provides a health check endpoint on the peer port:

```bash
curl http://localhost:51235/health
```

**Response Interpretation**

| HTTP Status | Meaning |
|-------------|---------|
| 200 | Healthy - all metrics normal |
| 503 | Warning - some metrics in warning range |
| 500 | Critical - server has serious issues |

A healthy response returns an empty `info` object:

```json
{
  "info": {}
}
```

An unhealthy response lists the problematic metrics:

```json
{
  "info": {
    "server_state": "syncing",
    "validated_ledger": "too old"
  }
}
```

**Metrics Checked**

- `server_state` - should be `full` or `proposing`
- `validated_ledger` - age of latest validated ledger
- `peers` - number of peer connections
- `amendment_blocked` - whether server is amendment blocked

**Monitoring Best Practices**

- Call the health check frequently (every 30-60 seconds)
- Don't alert on every unhealthy status - metrics can fluctuate briefly
- Escalate only for **severe and consistent** problems
- Allow 15 minutes after startup before expecting healthy status

**Special Cases**

- **Private peers**: Health check warns if peers < 7, but private validators intentionally have few peers. Check for your expected peer count instead.
- **Test networks**: Health check warns if validated ledger is > 7 seconds old, but test networks may have longer gaps between ledgers.

**Source:** [Health Check Interventions](https://xrpl.org/docs/infrastructure/troubleshooting/health-check-interventions)

### Disaster Recovery

**Scenario: Server Hardware Failure**

1. Provision new server meeting [Hardware Requirements](#hardware-requirements)
2. Install rippled from packages
3. Copy your `rippled.cfg` from backup (includes validator token)
4. Start rippled - it will sync from the network
5. Verify `server_state` returns to `proposing` (validator) or `full` (stock node)

**Scenario: Configuration Corruption**

1. Stop rippled
2. Restore `rippled.cfg` from backup
3. Verify file permissions: `chmod 600 /etc/opt/ripple/rippled.cfg`
4. Restart rippled

**Scenario: Master Key Compromise**

This is the worst case - see [Key Compromise Response](#key-compromise-response) in the Validator Keys section. You must revoke the key and coordinate with UNL publishers.

**Disaster Recovery Checklist**

- [ ] Encrypted backup of `validator-keys.json` in secure offline location
- [ ] Backup of `rippled.cfg` with current token
- [ ] Documented server provisioning process
- [ ] Tested restoration procedure (at least annually)
- [ ] Contact information for UNL publishers (in case of master key compromise)

---

## Upgrading Rippled

Keeping rippled updated is essential. Outdated servers can become amendment blocked and unable to participate in consensus. This section covers notification channels, upgrade procedures, and troubleshooting.

### Staying Notified

**Subscription Options (Use Multiple)**

| Channel | URL | What You Get |
|---------|-----|--------------|
| **ripple-server mailing list** | [groups.google.com/g/ripple-server](https://groups.google.com/g/ripple-server) | Release announcements, urgent upgrade notices, amendment alerts. **Essential.** |
| **GitHub Releases** | [github.com/XRPLF/rippled/releases](https://github.com/XRPLF/rippled/releases) | Detailed release notes, changelogs |
| **GitHub Watch** | Click "Watch" → "Custom" → "Releases" | Email notifications for new releases |
| **RSS Feed** | `github.com/XRPLF/rippled/releases.atom` | For RSS readers |

**Built-in Upgrade Notification**

rippled itself monitors the network and warns you when most validators are running a newer version:

```
Majority of your trusted validators run a higher version of rippled
server software. Please upgrade your rippled server software.
```

This message appears when >60% of validators on your UNL run a newer version. By the time you see this, upgrading is urgent.

**Amendment Voting Timeline**

When new amendments are introduced:

1. **Release**: New rippled version with amendment code is published
2. **Voting begins**: Validators vote for/against (requires 80% support for 2 weeks)
3. **Activation**: Amendment becomes active on the network
4. **Blocking**: Servers without the amendment code become amendment blocked

The window between voting start and activation is typically 2 weeks. Upgrade before activation to avoid being blocked.

**Source:** [XRPL Blog](https://xrpl.org/blog)

### Pre-Upgrade Checklist

Before upgrading:

1. **Check current version**
   ```bash
   rippled --version
   # or
   rippled server_info | grep build_version
   ```

2. **Read the release notes** - [GitHub Releases](https://github.com/XRPLF/rippled/releases)
   - Are there breaking changes?
   - Does this release require a config change?
   - Are there new amendments that will begin voting?

3. **Check for GPG key updates** - Some releases require re-trusting the package signing key

4. **Verify disk space** - Ensure sufficient space for the upgrade

5. **Plan timing** - Upgrade during low-activity periods if possible, though rippled restarts are fast

### Upgrade Procedures

**Method 1: Automatic Updates (Convenience)**

Enable automatic updates using the provided cron script:

```bash
# Check if the cron config exists
ls /opt/ripple/etc/update-rippled-cron

# Enable automatic updates
sudo ln -s /opt/ripple/etc/update-rippled-cron /etc/cron.d/
```

> **Warning:** Automatic updates carry risk. If the repository were compromised, malicious code could be deployed automatically. Not recommended for high-stakes validators.

**Method 2: Manual Update Script (Recommended)**

Use the provided update script:

```bash
sudo /opt/ripple/bin/update-rippled.sh
```

This script handles the full update process including daemon reload and restart.

**Method 3: Manual apt Commands (Most Control)**

```bash
# Update package lists
sudo apt update

# Check available version
apt-cache policy rippled

# Upgrade rippled
sudo apt upgrade rippled

# Reload systemd (picks up any unit file changes)
sudo systemctl daemon-reload

# Restart rippled
sudo systemctl restart rippled
```

**Method 4: Graceful Shutdown Before Upgrade**

For the smoothest upgrade with fastest re-sync:

```bash
# Install the new package (doesn't restart yet)
sudo apt update && sudo apt upgrade rippled

# Gracefully stop rippled
/opt/ripple/bin/rippled stop

# Wait for full stop (can take a minute)
while systemctl is-active rippled >/dev/null 2>&1; do
    sleep 2
done

# Reload and start
sudo systemctl daemon-reload
sudo systemctl start rippled
```

> **Note:** There's a known issue where `rippled stop` sometimes causes a restart instead of stopping. If this happens, use `sudo systemctl stop rippled` instead.

**GPG Key Updates**

Occasionally, the package signing key is renewed. If `apt update` fails with GPG errors:

```bash
# Download and trust the new key
wget -q -O - "https://repos.ripple.com/repos/api/gpg/key/public" | \
  sudo gpg --dearmor -o /etc/apt/keyrings/ripple.gpg

# Retry update
sudo apt update
```

**Source:** [Update Manually on Ubuntu](https://xrpl.org/update-rippled-manually-on-ubuntu.html)

### Post-Upgrade Verification

After upgrading, verify the server is healthy:

```bash
# Check version
rippled server_info | grep build_version

# Check server state (should return to proposing/full within minutes)
rippled server_info | grep server_state

# Check for amendment blocked status
rippled server_info | grep amendment_blocked

# Check health endpoint
curl http://localhost:51235/health

# Monitor logs for errors
tail -f /var/log/rippled/debug.log
```

**Expected Behavior**

- Server will sync with the network (5-15 minutes typical)
- `server_state` returns to `full` (stock node) or `proposing` (validator)
- `amendment_blocked` should be `false`
- Health check returns 200 with empty `info` object

### Amendment Blocking

**What It Means**

When the network enables a protocol amendment your server doesn't understand, your server becomes "amendment blocked." This is a safety feature - rather than misinterpret transactions under rules it doesn't know, the server stops processing.

**Symptoms**

- `server_state` shows `connected` instead of `full` or `proposing`
- `amendment_blocked: true` in `server_info`
- Error when submitting transactions: `amendmentBlocked` (error code 14)
- Health check returns critical status

**How to Check**

```bash
# Check if amendment blocked
rippled server_info | grep amendment_blocked

# See which features are blocking
rippled feature | grep -A1 '"blocked"'
```

**How to Resolve**

Upgrade to the latest rippled version:

```bash
sudo apt update && sudo apt upgrade rippled
sudo systemctl restart rippled
```

In most cases, upgrading to the latest version resolves the block. Check [Known Amendments](https://xrpl.org/resources/known-amendments) to see which version introduced support for specific amendments.

**Prevention**

- Subscribe to ripple-server mailing list for advance notice
- Upgrade promptly when new versions are released
- Don't wait for the built-in >60% warning - by then you're behind

**Source:** [rippled Server is Amendment Blocked](https://xrpl.org/docs/infrastructure/troubleshooting/server-is-amendment-blocked)

### Version History Context

As of January 2026:

- **Current version**: 3.0.0
- **Minimum for recent amendments**: 2.4.0+
- **Amendment blocked if older than**: Depends on which amendments have activated

Recent significant releases:
- **3.0.0** (December 2025): Five new amendments including fixPriceOracleOrder, fixTokenEscrowV1, fixAMMClawbackRounding
- **2.4.0**: Minimum version for many 2024-2025 amendments

Check [GitHub Releases](https://github.com/XRPLF/rippled/releases) for the complete changelog.

---

# Phase 6: Community & Reputation

For validators, building trust takes time. This phase covers community engagement, directory registration, and the long path to UNL inclusion.

---

## Community Resources

After deployment, you'll want to verify your node is visible, stay informed about network changes, and know where to get help.

### Network Explorers

Use these to verify your node's visibility and check network status:

| Resource | URL | What It Shows |
|----------|-----|---------------|
| **XRPL Explorer** | [livenet.xrpl.org](https://livenet.xrpl.org) | Official explorer. Account lookups, transactions, validator list at `/network/validators` |
| **Bithomp** | [bithomp.com](https://bithomp.com) | Explorer with validator tracking, domain verification status, and account analytics |
| **XRPL Cluster** | [xrplcluster.com](https://xrplcluster.com) | Community WebSocket endpoint with geographic routing. Stats at [xrpl.ws-stats.com](https://xrpl.ws-stats.com) |

### Staying Informed

**Mailing Lists (Essential)**

| List | URL | Content |
|------|-----|---------|
| **ripple-server** | [groups.google.com/g/ripple-server](https://groups.google.com/g/ripple-server) | Release announcements, upgrade warnings, amendment alerts. **Subscribe to this.** |
| **xrpl-announce** | [groups.google.com/g/xrpl-announce](https://groups.google.com/g/xrpl-announce) | Client library updates (~1 email/week) |

**Discord Servers**

| Server | Invite | Focus |
|--------|--------|-------|
| **XRP Ledger (Official)** | [discord.com/invite/xrpl](https://discord.com/invite/xrpl) | Community discussions, technical help, announcements |
| **XRPL Developers** | [discord.com/invite/CVG6Q2S3R8](https://discord.com/invite/CVG6Q2S3R8) | Developer-focused discussions |

**Other Channels**

| Resource | URL | Content |
|----------|-----|---------|
| **XRPL Blog** | [xrpl.org/blog](https://xrpl.org/blog) | Release notes, amendment activations, technical updates |
| **XRPChat** | [xrpchat.com](https://xrpchat.com) | Long-running community forum with technical discussions |
| **GitHub Releases** | [github.com/XRPLF/rippled/releases](https://github.com/XRPLF/rippled/releases) | Release notes and changelogs |

**Twitter/X Accounts**

- [@RippleXDev](https://twitter.com/RippleXDev) - Protocol changes, infrastructure updates
- [@XRPLLabs](https://twitter.com/XRPLLabs) - XRPL Labs updates, Hooks amendment news

### Getting Help

**Troubleshooting Resources**

| Resource | URL | Use For |
|----------|-----|---------|
| **Troubleshooting Guide** | [xrpl.org/docs/infrastructure/troubleshooting](https://xrpl.org/docs/infrastructure/troubleshooting) | Diagnosing problems, log messages, health checks |
| **Known Amendments** | [xrpl.org/resources/known-amendments](https://xrpl.org/resources/known-amendments) | Amendment details and voting information |
| **GitHub Issues** | [github.com/XRPLF/rippled/issues](https://github.com/XRPLF/rippled/issues) | Bug reports, feature requests |

**Where to Ask Questions**

1. **XRP Ledger Discord** - #technical-discussions for quick questions
2. **XRPChat Technical Discussion** - For detailed questions with context
3. **ripple-server mailing list** - For upgrade-related questions
4. **GitHub Issues** - For bugs (include logs and config)

> **Tip:** When asking for help, include: rippled version (`rippled --version`), `server_info` output, relevant log entries, and your OS/hardware specs.

---

## Directories

These sites track validator performance and UNL status. Register your validator to build visibility.

| Resource | URL | What It Shows |
|----------|-----|---------------|
| **XRPSCAN Validators** | [xrpscan.com/validators](https://xrpscan.com/validators) | All validators, domain verification, UNL membership, server versions, amendment votes |
| **XRPSCAN Amendments** | [xrpscan.com/amendments](https://xrpscan.com/amendments) | Amendment voting status, percentage support, expected activation |
| **Bithomp Validators** | [bithomp.com/validators](https://bithomp.com/validators) | Validator listings, UNL tracking |

**Post-deployment checklist:**
1. Verify your validator appears on XRPSCAN
2. Confirm domain verification shows correctly
3. Check your amendment votes are recorded
4. Monitor your agreement percentage over time

---

## Building Reputation

Getting on a UNL isn't automatic. It's earned over time.

**What UNL Publishers Look For**

| Criteria | Why It Matters |
|----------|----------------|
| **1+ year of reliable operation** | Anyone can run a validator for a week. Consistency over time demonstrates commitment. |
| **High uptime and agreement** | The numbers don't lie. >99% on both metrics. |
| **Domain verification** | Proves identity and enables accountability. |
| **Separate entity** | Not an employee of an existing validator operator. |
| **Geographic diversity** | Different data centers, regions, and jurisdictions strengthen the network. |
| **Public identity** | The community should know who you are. |

**Steps to Build Reputation**

1. Run reliably for the long haul. There are no shortcuts.
2. Set up domain verification from day one.
3. Publish your validator's public key on your website.
4. Register on validator directories like XRPSCAN.
5. Engage with the XRPL community.
6. Be transparent about your operations.
7. Consider running in an underrepresented region or on less-common infrastructure.

---

# Contributing

This document is maintained by [xrp-validator.grapedrop.xyz](https://xrp-validator.grapedrop.xyz). Contributions, corrections, and additional insights are welcome.

---

*Last updated: 2026-01-19*

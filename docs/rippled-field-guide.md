# **__THE RIPPLED FIELD GUIDE__**

*The following recommendations come from real-world validator operations, GitHub issues, and direct input from rippled engineers. This guidance is based on years of core insights and lessons learned the hard way, your cliff notes to bootstrap your rippled node or validator.*

---

# Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Node Sizing](#node-sizing)
  - [node_size](#node_size)
- [Database Management](#database-management)
  - [online_delete Tuning](#online_delete-tuning)
  - [advisory_delete](#advisory_delete)
- [Network Configuration](#network-configuration)
  - [compression](#compression)
  - [peers_max](#peers_max)
- [Port Configuration & Security](#port-configuration--security)
  - [Port Types](#port-types)
  - [Understanding ip vs admin](#understanding-ip-vs-admin)
  - [send_queue_limit](#send_queue_limit)
  - [Security Patterns](#security-patterns)
  - [Firewall Rules](#firewall-rules)
- [Time Synchronization](#time-synchronization)
  - [sntp_servers](#sntp_servers)
- [Fee Voting](#fee-voting)
- [Operational Settings](#operational-settings)
  - [log_level](#log_level)
  - [ssl_verify](#ssl_verify)
- [Putting It All Together](#putting-it-all-together)
- [Contributing](#contributing)

---

# Hardware Requirements

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

AMD Ryzen and Intel processors use the same x86-64 instruction set and are equally supported. ARM-based processors (including Apple Silicon) can compile rippled for development, but are **not recommended for production validators or nodes**.

**Disk Performance**

Storage speed is critical. Requirements:

- **Type**: SSD or NVMe (spinning disks will not work)
- **IOPS**: 10,000+ sustained (not burst)
- **Capacity**: 50 GB minimum for database partition

**Prefer bare metal over cloud.** Hyperscalers (AWS, Azure, OCI) introduce multiple issues for validators:

- **Noisy neighbors**: Shared physical hardware means other tenants can spike resource usage, throttling your validator during critical consensus moments
- **Network bandwidth limits**: Cloud VMs have baseline bandwidth with burst credits that deplete under sustained load. For example, AWS c5.large has only 750 Mbps baseline but advertises "up to 10 Gbps" - that burst lasts just 5 minutes before throttling kicks in
- **Storage I/O throttling**: EBS and equivalent block storage have IOPS burst credits. EC2 instances also have aggregate EBS bandwidth limits (e.g., r6i.xlarge drops from 40K to 6K IOPS after 30 minutes of sustained load)
- **Microburst penalties**: Short spikes in demand trigger throttling even when average utilization appears low - CloudWatch metrics aren't granular enough to detect millisecond-level bursts

If cloud is unavoidable, use instances with local NVMe storage (AWS i3/i4 series, Azure Lsv2, OCI Dense I/O) and dedicated network bandwidth ("n" suffix instances on AWS like C5n, M5n).

**Source**

See [XRPL System Requirements](https://xrpl.org/docs/infrastructure/installation/system-requirements) for official specifications.

---

# Node Sizing

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

# Database Management

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

# Network Configuration

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

**Configuration**

```ini
[peers_max]
21
```

---

# Port Configuration & Security

### Port Types

rippled uses four types of ports, each serving a different purpose:

| Port | Default | Protocol | Purpose |
|------|---------|----------|---------|
| **Peer** | 51235 | peer | Node-to-node communication. The only port that should be publicly accessible. |
| **RPC Admin** | 5005 | http | Administrative JSON-RPC API. Privileged commands like `stop`, `validation_seed`. |
| **WebSocket Admin** | 6006 | ws | Administrative WebSocket API. Same privileged access as RPC. |
| **WebSocket Public** | 5006 | ws | Public WebSocket API for clients. Subscribe to streams, submit transactions. |

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

# Time Synchronization

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

# Fee Voting

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

# Operational Settings

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

# Putting It All Together

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
| `node_size=huge` | 64 GB RAM allocation | Matches our hardware |
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

If running rippled in Docker with monitoring tools on the same host, change the admin port bindings:

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

This allows Docker containers to access the admin API while still restricting access to the Docker bridge networks.

---

# Contributing

This document is maintained by [xrp-validator.grapedrop.xyz](https://xrp-validator.grapedrop.xyz). Contributions, corrections, and additional insights are welcome.

---

*Last updated: 2026-01-18*

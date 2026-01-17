# The Rippled Field Guide

Operational intelligence for XRP Ledger validators and stock nodes. Compiled from production experience and core developer insights.

---

## Table of Contents

- [Node Sizing](#node-sizing)
  - [node_size](#node_size)
- [Database Management](#database-management)
  - [online_delete Tuning](#online_delete-tuning)
  - [advisory_delete](#advisory_delete)
- [Network Configuration](#network-configuration)
  - [compression](#compression)
  - [peers_max](#peers_max)
  - [send_queue_limit](#send_queue_limit)
- [Time Synchronization](#time-synchronization)
  - [sntp_servers](#sntp_servers)
- [Fee Voting](#fee-voting)
- [Operational Settings](#operational-settings)
  - [log_level](#log_level)
  - [ssl_verify](#ssl_verify)
- [Contributing](#contributing)

---

## Node Sizing

### node_size

#### The Bottom Line

**Match `node_size` to your available RAM.** This is the single most important sizing decision.

| Value | RAM Needed | Use Case |
|-------|------------|----------|
| tiny | 1-2 GB | Testing only |
| small | 4-8 GB | Light stock node |
| medium | 8-16 GB | Standard stock node |
| large | 16-32 GB | Busy stock node |
| huge | 32-64 GB | Validators, high-traffic nodes |

#### What It Controls

The `node_size` parameter is a macro that tunes multiple internal settings:

- Memory allocation for caches
- Number of worker threads
- Database buffer sizes
- Fetch pack sizes for historical data

#### Why It Matters

- **Undersized**: Node falls behind, misses ledgers, poor sync performance
- **Oversized**: Wastes memory, no performance benefit
- **Just right**: Smooth operation with efficient resource use

#### Configuration

```ini
[node_size]
huge
```

No other parameters needed - rippled auto-tunes everything else based on this value.

#### Common Mistake

Don't try to manually tune thread counts or cache sizes. rippled's auto-tuning based on `node_size` is well-tested. Manual overrides often make things worse.

---

## Database Management

### online_delete Tuning

#### The Bottom Line

**Set `online_delete` as high as your disk space allows.**

| Disk Budget for DB | Recommended Setting | History Retained |
|--------------------|---------------------|------------------|
| ~25 GB | 32768 | ~36 hours |
| ~50 GB | 65536 | ~3 days |
| ~100 GB+ | 131072 | ~6 days |

#### Why Higher is Better

Counter-intuitively, **lower `online_delete` values cause more I/O, not less**.

rippled maintains two NuDB databases simultaneously:
- **"Writable" DB** - receives new/modified nodes
- **"Archive" DB** - the older database being phased out

During rotation, rippled must copy all nodes that **weren't modified** since the last rotation from archive to writable. This is the expensive operation.

**The math:**
- Lower `online_delete` → fewer ledgers between rotations → fewer transactions → fewer modified nodes
- Fewer modified nodes → **more nodes must be copied** during rotation
- More copying → I/O spikes → potential missed consensus rounds

**Extreme example:** If you rotated every ledger and only 6 accounts changed, you'd copy millions of unchanged nodes. With 32,768 ledgers between rotations, far more nodes have been naturally modified, so fewer need copying.

#### The Trade-off

| Setting | Rotation Frequency | I/O Pattern | Disk Usage |
|---------|-------------------|-------------|------------|
| 512 | Every ~30 min | Severe sawtooth spikes | Minimal |
| 2000 | Every ~2 hours | Noticeable spikes | Low |
| 32768 | Every ~36 hours | Smooth | Moderate |
| 65536+ | Every ~3+ days | Very smooth | Higher |

#### Configuration

```ini
[node_db]
type=NuDB
path=/var/lib/rippled/db/nudb
online_delete=32768

[ledger_history]
32768
```

**Note:** `ledger_history` must be ≤ `online_delete`.

#### Additional Tuning Parameters

If you still experience issues, these parameters in `[node_db]` may help:
- `age_threshold_seconds` - minimum age before deletion eligible
- `recovery_wait_seconds` - delay before rotation resumes after interruption

#### The One Rule

**Never use low values to "save disk space."** You'll pay for it in I/O storms and degraded validator performance. Disk is cheap; validator reputation isn't.

#### Source

This insight comes from [GitHub discussion XRPLF/rippled#6202](https://github.com/XRPLF/rippled/issues/6202), with explanation from Ripple engineer @ximinez.

### advisory_delete

#### The Bottom Line

**Use `advisory_delete=0` (the default).** Let rippled manage deletion automatically.

#### What It Does

| Value | Behavior |
|-------|----------|
| 0 | rippled decides when to delete old ledgers (automatic) |
| 1 | rippled waits for an external signal before deleting |

#### When to Use Each

- **`0` (recommended)**: Standard deployments. rippled handles everything.
- **`1`**: Only if you have external tooling that needs to process ledger data before deletion, or you're running a specialized archival integration.

#### Configuration

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

#### The Bottom Line

**Always enable compression.** There's no good reason to disable it.

```ini
[compression]
true
```

#### What It Does

Compresses peer protocol messages using LZ4. Reduces bandwidth usage by 60-80% with negligible CPU overhead.

#### When to Disable

Almost never. The only scenario: you're on a local network with unlimited bandwidth and you're chasing microseconds of latency. Even then, the savings are marginal.

### peers_max

#### The Bottom Line

**21-30 for validators, 15-21 for stock nodes.**

| Node Type | Recommended | Notes |
|-----------|-------------|-------|
| Validator | 21-30 | Enough for reliable propagation |
| Stock node | 15-21 | Balance between connectivity and resources |
| Private/minimal | 10-15 | Minimum viable connectivity |

#### The Trade-off

- **More peers**: Faster transaction/ledger propagation, more redundancy if peers drop
- **Fewer peers**: Less bandwidth, CPU, and memory usage

#### Why Validators Don't Need Huge Counts

Validators receive transactions from the network and propagate validations. They don't need 100 peers - they need *enough* peers to stay reliably connected. 21 well-connected peers is plenty.

#### Configuration

```ini
[peers_max]
21
```

### send_queue_limit

#### The Bottom Line

**100 for admin WebSockets, 500 for public WebSockets.**

#### What It Does

Controls how many outbound messages can queue per WebSocket connection before rippled starts dropping messages or disconnecting slow clients.

| Port Type | Recommended | Rationale |
|-----------|-------------|-----------|
| Admin WS | 100 | Local connections are fast |
| Public WS | 500 | Remote clients may be slower |

#### Configuration

```ini
[port_ws_admin_local]
port = 6006
ip = 127.0.0.1
protocol = ws
send_queue_limit = 100

[port_ws_public]
port = 5006
ip = 0.0.0.0
protocol = ws
send_queue_limit = 500
```

Higher values tolerate slower clients but use more memory per connection.

---

## Time Synchronization

### sntp_servers

#### The Bottom Line

**Configure multiple diverse time sources.** Consensus depends on accurate time.

```ini
[sntp_servers]
time.nist.gov
pool.ntp.org
time.cloudflare.com
time.google.com
```

#### Why Multiple Servers

- **Redundancy**: If one server is unreachable, others provide time
- **Cross-checking**: rippled can detect if one source is wrong
- **Low latency**: Different providers have different geographic coverage

#### Good Server Choices

| Server | Type | Notes |
|--------|------|-------|
| time.nist.gov | Government | US-based, authoritative |
| pool.ntp.org | Community pool | Anycast, global |
| time.cloudflare.com | CDN | Low latency globally |
| time.google.com | CDN | Smeared leap seconds |

#### Why Time Matters

The XRP Ledger consensus protocol uses timestamps. Clock drift can cause:
- Validation timing issues
- Transactions appearing to be in the future/past
- Sync problems with other validators

Your system should also run ntpd or chronyd for OS-level time sync.

---

## Fee Voting

#### The Bottom Line

**Use the network defaults unless you have strong governance reasons to deviate.**

```ini
[voting]
reference_fee = 10
account_reserve = 1000000
owner_reserve = 200000
```

#### What These Values Mean

| Setting | Value | Human Readable | Purpose |
|---------|-------|----------------|---------|
| reference_fee | 10 | 0.00001 XRP | Base transaction cost |
| account_reserve | 1000000 | 1 XRP | Minimum balance to activate account |
| owner_reserve | 200000 | 0.2 XRP | Cost per owned object (trustline, offer, etc.) |

#### How Fee Voting Works

Validators vote on these values. The network uses the **median** of all validator votes. This means:

- Your vote alone won't change anything
- Changing network fees requires coordinated validator consensus
- Outlier votes are ignored

#### Philosophy Behind Current Values

- **reference_fee (10 drops)**: Low enough for normal use, high enough to make spam expensive at scale
- **account_reserve (1 XRP)**: Prevents ledger bloat from millions of empty accounts
- **owner_reserve (0.2 XRP)**: Makes users think twice before creating objects that live forever in the ledger

#### When to Vote Differently

Rarely. Fee changes are network governance decisions. Only deviate if:
- You're participating in a coordinated network-wide fee adjustment
- You have data showing current fees are causing problems
- You've discussed with other validators

---

## Operational Settings

### log_level

#### The Bottom Line

**Run `warning` in production.** Only increase for troubleshooting.

| Level | Verbosity | Disk Impact | Use Case |
|-------|-----------|-------------|----------|
| trace | Extreme | GB/hour | Deep debugging only |
| debug | High | Large | Development, troubleshooting |
| info | Medium | Moderate | Default (too verbose for production) |
| **warning** | **Low** | **Minimal** | **Production recommended** |
| error | Very low | Very low | Only problems |
| fatal | Minimal | Negligible | Almost nothing |

#### Configuration

Set via `[rpc_startup]` to apply at boot:

```ini
[rpc_startup]
{ "command": "log_level", "severity": "warning" }
```

#### Runtime Adjustment

Temporarily increase for debugging without restart:

```bash
rippled log_level debug
```

Then set back:

```bash
rippled log_level warning
```

#### Why Not `info`?

The default `info` level logs every ledger close, peer connection, and many routine operations. On a busy validator, this generates significant disk I/O. `warning` logs only things that might need attention.

### ssl_verify

#### The Bottom Line

**Always `1` in production.** Never disable SSL verification.

```ini
[ssl_verify]
1
```

#### What It Does

| Value | Behavior |
|-------|----------|
| 1 | Verify SSL certificates for outbound connections |
| 0 | Skip verification (INSECURE) |

#### Why It Matters

rippled makes HTTPS connections to validator list publishers (vl.ripple.com, unl.xrplf.org). SSL verification ensures:

- You're actually talking to the real publisher
- No man-in-the-middle can inject a malicious validator list
- Your node won't trust fake validators

#### When `0` is Acceptable

- Local development with self-signed certificates
- Isolated test networks
- Never in production

---

## Contributing

This document is maintained by [grapedrop.xyz](https://grapedrop.xyz). Contributions, corrections, and additional insights are welcome.

---

*Last updated: 2026-01-17*

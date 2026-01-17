<img src="images/rippled-field-guide-logo.gif" alt="The Rippled Field Guide" width="400">

Operational intelligence for XRP Ledger validators and stock nodes. Compiled from production experience and core developer insights.

---

## What is rippled?

[rippled](https://github.com/XRPLF/rippled) is the reference server implementation for the XRP Ledger, a decentralized public blockchain built for fast, low-cost, and reliable value transfer. Unlike proof-of-work or proof-of-stake networks, the XRP Ledger uses a unique consensus protocol where a network of independent validators agrees on the order and validity of transactions every 3-5 seconds.

rippled is the software that powers this network. It validates transactions, maintains the ledger history, and participates in the consensus process. You can run rippled as a **stock node** (tracking the ledger, serving API requests) or as a **validator** (actively participating in consensus and voting on the network's future).

The XRP Ledger has been operating continuously since 2012, processing over 2 billion transactions with no downtime. It handles 1,500+ transactions per second with settlement finality in seconds, not minutes or hours.

---

## Running a Validator: The Commitment

Running a validator is a responsibility, not just a technical deployment. You are becoming part of the infrastructure that secures the network and determines its rules. Before you spin up a validator, understand what you're signing up for.

### The Core Duties

A validator must be **online**, **honest**, and **fast**.

| Duty | Target | What It Means |
|------|--------|---------------|
| **Availability** | 100% uptime | Your validator should be online and participating in every consensus round. The network tracks your performance over the last 256 ledgers. |
| **Agreement** | >99% | Your votes must match the consensus outcome. Disagreement may indicate stale software, bugs, or (worst case) intentional misbehavior. |
| **Timeliness** | Votes before round ends | A late vote is a missed vote. Your infrastructure must deliver sub-second response times. |
| **Impartiality** | Process all transactions | You don't get to pick winners. Process every valid transaction without censorship or delay. |

### What Good Citizenship Looks Like

- **Run the latest software.** Always. Without modifications. When a new release drops, upgrade promptly.
- **Announce maintenance.** If you need downtime, tell the community ahead of time.
- **Verify your domain.** Prove you control the identity associated with your validator.
- **Vote responsibly.** Amendments are protocol changes. Understand them before voting.
- **Be reachable.** The community should be able to identify and contact you.
- **Stay informed.** Subscribe to [rippled releases](https://github.com/XRPLF/rippled/releases) and the [XRPL Google Group](https://groups.google.com/g/ripple-server).

### The Stakes

The XRP Ledger doesn't have slashing penalties. If you misbehave or go offline, you won't lose funds. But you will lose something more valuable: **trust**.

Poor performers get added to the **Negative UNL**—a list of validators the network temporarily ignores for consensus calculations. Persistent problems lead to removal from UNL publisher lists entirely. Your reputation takes years to build and moments to destroy.

---

## Deployment: Doing It Right

### Hardware Requirements

Don't cheap out. Validators are latency-sensitive and I/O-intensive.

| Component | Recommended | Notes |
|-----------|-------------|-------|
| **CPU** | 8+ cores, 3+ GHz | x86_64 only. ARM/Apple Silicon not production-ready. |
| **RAM** | 32-64 GB | Match with `node_size=huge` in config. |
| **Storage** | NVMe SSD | 10,000+ sustained IOPS. **No Amazon EBS.** |
| **Network** | Low-latency, redundant | Multiple ISP connections if possible. |

### Security Architecture

Your validator is a target. Design accordingly.

**Infrastructure:**
- Run on **bare metal** or dedicated servers—not VMs, not shared hosting.
- Data center with redundant power, cooling, and physical security.
- Uninterruptible power supply (UPS) and backup generator.

**Network:**
- Enable `peer_private = 1` in config to hide your validator's IP.
- Cluster your validator behind **2+ stock nodes** (sentry architecture).
- Validator connects only to nodes you control.
- Firewall everything; validator makes outbound connections only.
- Your domain should **NOT** resolve to your validator's IP address.

**Keys:**
- Store your master key **offline** (encrypted USB in a secure location).
- Use validator tokens for the running server—these can be rotated if compromised.
- Config file permissions: `chmod 600`.

### Docker vs. Bare Metal

**Official recommendation: bare metal.**

Bare metal gives you deterministic performance, no noisy neighbors, and full hardware control. You also avoid cloud provider policies that may suddenly ban crypto workloads.

If you must use Docker, community-maintained images exist. Use dedicated volume mounts for keys, configure proper health checks, and understand you're accepting additional operational complexity.

---

## Operations: The Daily Grind

### Monitoring

You can't fix what you can't see. Monitor:

- **Server state**: Should be `proposing` (validator) or `full` (stock node). Alert on `disconnected`, `connected`, or `syncing`.
- **Agreement percentage**: Your votes vs. consensus outcome.
- **Resource utilization**: CPU, memory, disk I/O, network.
- **Peer connections**: Healthy count, no sudden drops.
- **Ledger sync**: Are you keeping up with the network?

Prometheus + Grafana is a solid stack. Alert on meaningful thresholds—not every blip.

### Upgrades

- **Subscribe** to release notifications.
- **Upgrade promptly**—especially when amendments are approaching activation.
- **Test configuration changes** on a non-production node first.
- **Amendment-blocked** servers can't participate in consensus. Don't let this happen to you.

### Key Management

- **Rotation**: Generate new validator tokens periodically. Old tokens are automatically invalidated.
- **Revocation**: If your key is compromised, revoke it immediately using the `validator-keys` tool.
- **Domain verification**: Set it up early. It's required for UNL consideration.

### Amendment Voting

Validators on UNLs vote on protocol amendments. An amendment needs >80% support for two consecutive weeks to activate. Once passed, changes are **permanent**.

Understand what you're voting for. Engage with the community. Don't abstain by default—your vote matters.

---

## The Path to Trust: UNL Inclusion

The **Unique Node List (UNL)** is the set of validators a server trusts for consensus. UNL publishers like [Ripple](https://vl.ripple.com) and the [XRPL Foundation](https://unl.xrplf.org) maintain curated lists.

Getting on a UNL isn't automatic. It's earned.

### What Publishers Look For

| Criteria | Why It Matters |
|----------|----------------|
| **1+ year of reliable operation** | Anyone can run a validator for a week. Consistency over time demonstrates commitment. |
| **High uptime and agreement** | The numbers don't lie. >99% on both metrics. |
| **Domain verification** | Proves identity and enables accountability. |
| **Separate entity** | Not an employee of an existing validator operator. |
| **Geographic diversity** | Different data centers, regions, and jurisdictions strengthen the network. |
| **Public identity** | The community should know who you are. |

### Building Your Reputation

1. Run reliably for the long haul—there are no shortcuts.
2. Set up domain verification from day one.
3. Publish your validator's public key on your website.
4. Register on validator directories like [XRPSCAN](https://xrpscan.com/validators).
5. Engage with the XRPL community.
6. Be transparent about your operations.
7. Consider running in an underrepresented region or on less-common infrastructure.

---

## The Field Guide

The [Rippled Field Guide](docs/rippled-field-guide.md) contains detailed configuration guidance for specific rippled settings:

- **Node Sizing** - `node_size` parameter and RAM allocation
- **Database Management** - `online_delete` tuning, I/O storm prevention
- **Network Configuration** - compression, peer limits, queue settings
- **Time Synchronization** - SNTP server configuration
- **Fee Voting** - reference fees and reserve settings
- **Operational Settings** - logging, SSL verification

Each entry follows the same format:
1. **Bottom-line recommendation** - what to do
2. **Technical explanation** - why it works
3. **Configuration example** - how to implement
4. **Source** - where the insight came from

---

## Sources

This guide draws from official documentation, core developer insights, and production experience:

- [XRPL: Run rippled as a Validator](https://xrpl.org/docs/infrastructure/configuration/server-modes/run-rippled-as-a-validator)
- [XRPL: System Requirements](https://xrpl.org/docs/infrastructure/installation/system-requirements)
- [XRPL: Consensus Protocol](https://xrpl.org/docs/concepts/consensus-protocol)
- [XRPL: Amendments](https://xrpl.org/docs/concepts/networks-and-servers/amendments)
- [XRPL Blog: Running an XRP Ledger Validator](https://xrpl.org/blog/2020/running-an-xrp-ledger-validator)
- [XLS-50d: Healthy Validator Infrastructure Distribution](https://github.com/XRPLF/XRPL-Standards/discussions/145)
- [rippled GitHub Repository](https://github.com/XRPLF/rippled)

---

## Contributing

Found something that should be in here? Have a correction or additional insight? Open an issue or PR.

---

## License

[MIT](LICENSE)

---

## Maintainer

[grapedrop.xyz](https://grapedrop.xyz)

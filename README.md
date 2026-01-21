<p align="center">
  <img src="images/rippled-field-guide-logo.gif" alt="The Rippled Field Guide" width="400">
</p>

Operational intelligence for XRP Ledger validators and stock nodes. Compiled from production experience and core developer insights. The reference guide we wish we had when we started running validators.

**This is a community project.** Running a node or validator? [Share what you've learned.](#contributing)

---

## Ready to Get Your Hands Dirty?

We recommend reading through this README to understand the commitment. But if you're the type who reads the manual after the thing's already on fire, head straight to **[The Rippled Field Guide](docs/RIPPLED-FIELD-GUIDE.md)**.

---

## What is rippled?

[rippled](https://github.com/XRPLF/rippled) is the reference server implementation for the XRP Ledger, a decentralized public blockchain built for fast, low-cost, and reliable value transfer. Unlike proof-of-work or proof-of-stake networks, the XRP Ledger uses a unique consensus protocol where a network of independent validators agrees on the order and validity of transactions every 3-5 seconds.

rippled is the software that powers this network. It validates transactions, maintains the ledger history, and participates in the consensus process. You can run rippled as a **stock node** (tracking the ledger, serving API requests) or as a **validator** (actively participating in consensus and voting on the network's future).

The XRP Ledger has been operating continuously since 2012, processing over 3.8 billion transactions with no downtime. It handles 1,500+ transactions per second with settlement finality in seconds, not minutes or hours.

---

## Running a Validator: The Commitment

Running a validator is a responsibility, not just a technical deployment. You are becoming part of the infrastructure that secures the network and determines its rules. Before you spin up a validator, understand what you're signing up for.

### Consensus Duties

Every 3-5 seconds, validators participate in a consensus round to agree on the next ledger. These are your real-time responsibilities:

<p align="center">
  <img src="images/validator-consensus-participation.png" alt="Validator Consensus Participation" width="900">
</p>

| Duty | Target | What It Means |
|------|--------|---------------|
| **Availability** | 100% uptime | Your validator should be online and participating in every consensus round. The network tracks your performance over the last 256 ledgers. |
| **Agreement** | >99% | Your validations must match the consensus outcome. Disagreement may indicate stale software, bugs, or (worst case) intentional misbehavior. |
| **Timeliness** | Validate before round ends | A late validation is a missed validation. Your infrastructure must deliver sub-second response times. |
| **Impartiality** | Process all transactions | You don't get to pick winners. Process every valid transaction without censorship or delay. |

### Governance Duties

Validators on UNLs also participate in network governance through voting:

| Duty | Frequency | What It Means |
|------|-----------|---------------|
| **Amendment Voting** | Ongoing | Protocol changes require >80% validator support for 2 weeks to activate. Understand proposals before voting. Your vote shapes the network's future. |
| **Fee Voting** | Ongoing | Validators vote on base transaction fees and reserve requirements. The network uses the median of all votes. |

Governance voting is configured in `rippled.cfg` or via the admin API. This is different from consensus validation, which happens automatically every few seconds.

### What Good Citizenship Looks Like

- **Run the latest software.** Always. Without modifications. When a new release drops, upgrade promptly.
- **Announce maintenance.** If you need downtime, tell the community ahead of time.
- **Verify your domain.** Prove you control the identity associated with your validator.
- **Participate in governance.** Don't ignore amendment votes. Understand what you're voting for.
- **Be reachable.** The community should be able to identify and contact you.
- **Stay informed.** Subscribe to [rippled releases](https://github.com/XRPLF/rippled/releases) and the [XRPL Google Group](https://groups.google.com/g/ripple-server).

### The Stakes

Unlike proof-of-work or proof-of-stake networks, the XRP Ledger offers **no direct economic incentives** for running a validator. There are no block rewards, no staking yields, no transaction fee distributions. So why do people run validators?

Validators run because they have a stake in the network's success. Exchanges need reliable transaction processing. Businesses building on XRPL need the network to function. Individuals who hold XRP benefit from network health. The lack of direct rewards filters for operators who are genuinely invested in the ecosystem rather than mercenary miners chasing the highest yield.

The XRP Ledger also doesn't have slashing penalties. If you misbehave or go offline, you won't lose funds. But you will lose something more valuable: **trust**.

Poor performers get added to the **Negative UNL**, a list of validators the network temporarily ignores for consensus calculations. Persistent problems lead to removal from UNL publisher lists entirely. Your reputation takes years to build and moments to destroy.

---

## Deployment: Doing It Right

### Hardware Requirements

Don't cheap out. Validators are latency-sensitive and I/O-intensive.

| Component | Recommended | Notes |
|-----------|-------------|-------|
| **CPU** | 8+ cores, 3+ GHz | x86-64 only (Intel/AMD). See note below. |
| **RAM** | 32-64 GB | Match with `node_size=huge` in config. |
| **Storage** | NVMe SSD | 10,000+ sustained IOPS. **No Amazon EBS.** |
| **Network** | Low-latency, redundant | Multiple ISP connections if possible. |

**CPU Architecture Note:** AMD Ryzen, Intel Core, Xeon, and EPYC are all x86-64 and fully supported. ARM processors (Apple Silicon M1/M2/M3/M4, AWS Graviton, Raspberry Pi) can compile rippled for development but are not recommended for production. See [XRPL System Requirements](https://xrpl.org/docs/infrastructure/installation/system-requirements).

### Security Architecture

Your validator is a target. Design accordingly.

<p align="center">
  <img src="images/validator-sentry-architecture.png" alt="Validator Sentry Architecture" width="700">
</p>

**Infrastructure:**
- Run on **bare metal** or dedicated servers. Not VMs, not shared hosting.
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
- Use validator tokens for the running server. These can be rotated if compromised.
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

For a turnkey solution, the [XRPL Validator Dashboard](https://github.com/realgrapedrop/xrpl-validator-dashboard) provides full-service monitoring built specifically for validator operators. It tracks 40+ rippled metrics via WebSocket and HTTP APIs *plus* system metrics (CPU, memory, disk I/O, network). Includes pre-configured alerts for critical states and supports Discord, Slack, PagerDuty, and other notification channels. Deploys as a single Docker stack - no libraries or dependencies on your rippled machine. Can run on a separate machine entirely for better security (see [Hardened Architecture](https://github.com/realgrapedrop/xrpl-validator-dashboard/blob/main/docs/HARDENED_ARCHITECTURE.md)).

Prometheus + Grafana is an alternative for system metrics, but you'll need additional tooling to monitor rippled-specific metrics like server state, agreement percentage, or peer connections.

### Upgrades

- **Subscribe** to release notifications.
- **Upgrade promptly**, especially when amendments are approaching activation.
- **Test configuration changes** on a non-production node first.
- **Amendment-blocked** servers can't participate in consensus. Don't let this happen to you.

### Key Management

- **Rotation**: Generate new validator tokens periodically. Old tokens are automatically invalidated.
- **Revocation**: If your key is compromised, revoke it immediately using the `validator-keys` tool.
- **Domain verification**: Set it up early. It's required for UNL consideration.

### Amendment Voting

Validators on UNLs vote on protocol amendments. An amendment needs >80% support for two consecutive weeks to activate. Once passed, changes are **permanent**.

Understand what you're voting for. Engage with the community. Don't abstain by default. Your vote matters.

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

**Why Identity Matters**

The XRP Ledger's trust model is fundamentally different from proof-of-work or proof-of-stake. There's no mining power or staked capital to lose. Instead, the network relies on **reputation and accountability**.

When you add a validator to your UNL, you're saying "I trust this operator not to collude against the network." That trust requires knowing who you're trusting. Anonymous operators may be technically competent, but anonymity removes accountability. If something goes wrong, there's no reputation at stake, no entity to answer questions, and no way to coordinate on fixes.

This is why UNL publishers prioritize validators with verified domains, public identities, and community presence. Identity is the collateral in a trust-based system.

### Building Your Reputation

1. Run reliably for the long haul. There are no shortcuts.
2. Set up domain verification from day one.
3. Publish your validator's public key on your website.
4. Register on validator directories like [XRPSCAN](https://xrpscan.com/validators).
5. Engage with the XRPL community.
6. Be transparent about your operations.
7. Consider running in an underrepresented region or on less-common infrastructure.

---

## The Field Guide

This is why you came here. While the official [Running an XRP Ledger Validator](https://xrpl.org/blog/2020/running-an-xrp-ledger-validator) explains the why, **[The Rippled Field Guide](docs/RIPPLED-FIELD-GUIDE.md)** covers the how: practical configuration, hard-won lessons, and operational insights compiled into a single reference.

**Getting Started**
- **Deployment Roadmap** - Logical sequence from infrastructure to operations

**Infrastructure & Configuration**
- **Hardware Requirements** - CPU, RAM, storage, bare metal vs cloud
- **Node Sizing** - `node_size` parameter and RAM allocation
- **Database Management** - `online_delete` tuning, I/O storm prevention
- **Network Configuration** - compression, peer limits
- **Port Configuration & Security** - port types, `ip` vs `admin`, firewall rules
- **Time Synchronization** - SNTP server configuration
- **Operational Settings** - logging, SSL verification

**Validator Identity & Security**
- **Domain Verification** - xrp-ledger.toml setup and common mistakes
- **Validator Keys** - master key security, token management, compromise response
- **Fee Voting** - reference fees and reserve settings

**Post-Deployment**
- **Community Resources** - Network explorers, validator directories, staying informed, getting help

**Reference**
- **Putting It All Together** - Complete production rippled.cfg example

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

This is a community driven project. If you run rippled as a stock node or validator, your operational experience is valuable. Hard-won lessons, edge cases, and configuration insights that helped you can help others.

**How to contribute:**

1. **Open a GitHub issue** with your content, insight, or correction
2. Submissions are reviewed for accuracy and fit
3. Accepted contributions are incorporated into the Field Guide

No pull request required, just describe what you learned and I will handle the formatting. Whether it's a single configuration tip or a full section on a topic we haven't covered, it's welcome here.

---

## License

[MIT](LICENSE)

---

## Maintainer

[xrp-validator.grapedrop.xyz](https://xrp-validator.grapedrop.xyz)

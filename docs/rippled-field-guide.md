# The Rippled Field Guide

Operational intelligence for XRP Ledger validators and stock nodes. Compiled from production experience and core developer insights.

---

## Table of Contents

- [Database Management](#database-management)
  - [online_delete Tuning](#online_delete-tuning)
- [Contributing](#contributing)

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

---

## Contributing

This document is maintained by [grapedrop.xyz](https://grapedrop.xyz). Contributions, corrections, and additional insights are welcome.

---

*Last updated: 2026-01-17*

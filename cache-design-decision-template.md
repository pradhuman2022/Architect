# Cache / Distributed Store — Design Decision Template

A repeatable guide for choosing a data-tier model. Answer the gates in order.
The first "yes" that routes you out of the cache section wins — stop there.

---

## GATE 0 — Is this even a cache?

> **The one question:** *If this data vanishes, do I lose truth, or just speed?*

- [ ] **Just speed** (there is a database/source of truth behind it) → it's a **cache**. Go to Gate 1.
- [ ] **Lose truth** (this data IS authoritative — nothing behind it) → **NOT a cache.** Go to the "Not a Cache" section at the bottom.

Quick tells that it's NOT a cache: locks, leader election, counters/quotas that must be exact,
idempotency keys, wallet/inventory balances, config that must never be lost.

---

## CACHE SECTION

### Gate 1 — Capture the workload

| Question | Your answer |
|---|---|
| Read : Write ratio | __________ |
| Absolute write rate (writes/sec, peak) | __________ |
| Do concurrent writes to the **same key** happen? | Y / N |
| If a concurrent write is lost, does it matter? | Y / N |
| Staleness tolerance (how long is stale OK?) | __________ |
| Is the source-of-truth behind it fast/cheap to re-fetch on miss? | Y / N |

### Gate 2 — Pick the model (writes decide, not reads)

Reads are fast in every model (all read local). **The write side picks the model.**

```
Writes light / rare?
   └── YES → LEADERLESS (EVCache-style)          [Model A]

Writes heavy?
   └── Concurrent same-key writes matter (Gate 1)?
         ├── NO  → LEADERLESS + async batched fan-out   [Model A+]
         └── YES → SHARDED LEADER-FOLLOWER              [Model B]

Both read- AND write-heavy?
   └── Do you need write ORDERING / no lost concurrent writes?
         ├── NO  → LEADERLESS + local-ack + async fan-out [Model A+]
         └── YES → SHARDED LEADER-FOLLOWER (read followers)[Model B]
```

### The models

**Model A — Leaderless (EVCache)**
- Full copy per AZ. Client writes to all AZs. Read local AZ, fall back cross-AZ on miss.
- Heal via TTL + overwrites (+ optional background anti-entropy repair).
- ✅ Simple, no leader, AZ-death is a shrug, write-local, self-load-balancing reads.
- ❌ Write amplification (N× per write), no write ordering (LWW → clock-skew risk).
- **Best for: read-heavy, staleness-tolerant.**

**Model A+ — Leaderless tuned for throughput**
- Model A, plus: local-ack writes, **async batched + pipelined** cross-AZ fan-out,
  client near-cache, single-flight on misses, background Merkle repair, decorrelated eviction.
- Pushes ALL coordination off the hot path.
- **Best for: both-heavy where you don't need ordering.**

**Model B — Sharded Leader-Follower (Redis Cluster)**
- Partition keys → each shard has one primary. Write to primary, async replicate to followers.
  Read from followers (slightly stale) or primary (fresh).
- ✅ Cheap ordered writes, high throughput (scale shards), no lost concurrent writes.
- ❌ Failover pauses writes on leader death; may lose un-replicated tail; more ops complexity;
  writes pay a hop to the leader (not local).
- **Best for: write-heavy, or both-heavy needing ordering.**
- ⚠️ ALWAYS shard — a single leader is a write bottleneck.

### Gate 3 — Quorum? (optional dial — NOT a consistency claim)

Only if you want to tune WHERE latency lands. With N=3 (N = replica count):

| Want | Set | Cost |
|---|---|---|
| Cheap reads | R=1, W=N | Writes pay all AZs; strict W=N blocks writes if any AZ down |
| Cheap writes | W=1, R=N | Reads pay all AZs + N× read load; blocks reads if any AZ down |
| Both cheap | W=1, R=1 | Pure eventual, no overlap |

**Rules:**
- Overlap needs `W + R > N` — but this does **NOT** give strong consistency on a cache
  (eviction lets replicas forget → the "guarantee" breaks). It only lowers staleness odds.
- **Never call any setting "strong."** For strong, see "Not a Cache" below.
- For a cache, use **best-effort** (target the quorum, don't *require* it) so an AZ outage
  degrades to "fewer copies," not "request fails." Requiring quorum = database behavior.
- Note: R=1 / W=N-best-effort **is literally EVCache**.

### Gate 4 — Failure & correctness checklist
- [ ] AZ down → do writes degrade (best-effort) rather than fail? (should be YES for a cache)
- [ ] Returning AZ heals lazily via TTL/overwrite (Model A) or catches up from leader (Model B)?
- [ ] TTL set as the staleness backstop?
- [ ] (Model A+) background repair + decorrelated eviction to counter LRU de-replication?
- [ ] LWW clock authority named / bounded skew stated? (or use a logical version)

---

## NOT A CACHE SECTION (Gate 0 = "lose truth")

> Don't bolt consistency onto the cache. Route to the right store.

```
Need a stored VALUE, strongly consistent, and you already run a DB?
   └── Use the DATABASE (SELECT FOR UPDATE / UPDATE WHERE / unique constraint).
       ACID already gives the consistency a cache can't. Don't add a cache or ZK.

Need in-memory SPEED + strong consistency + durability?
   └── MEMORYDB (WAL-backed). It's a database, not a cache.
       (WAL = writes committed to a quorum-replicated log before ack → durable + ordered.)

Need COORDINATION: leader election, auto-releasing locks, watch-on-change?
   └── ETCD / ZOOKEEPER. Ephemeral leases + watches + consensus.
```

---

## ONE-LINE SUMMARY

- **Cache = you can lose it** → keep it simple & eventual.
  - Read-heavy → **Leaderless (EVCache)**
  - Write-heavy → **Sharded Leader-Follower**
  - Both → Leaderless+async (simple/available) OR Leader-Follower (ordering/throughput)
- **Reads don't pick the model — writes do.**
- **Can't-lose-it or must-be-correct → it's not a cache** → DB / MemoryDB / etcd.
- **Quorum is a latency dial, not a strong-consistency switch. Best-effort, never strict, on a cache.**

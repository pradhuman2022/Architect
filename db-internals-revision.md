# Database Internals — Complete Revision Notes
*Prady ka personal revision doc — Storage Engines, LSM, B-Tree, System Design*

---

## 1. Uber's InnoDB Problem — Postgres vs MySQL

### Root Cause
Postgres ka row update = nayi copy likhta hai (MVCC). Old row physically wahan hi padhi rehti hai — sirf `t_xmax` set hota hai.

### Postgres Update Problem
```
1 UPDATE → 
  WAL mein likha
  Heap File mein nayi row copy
  Saare indexes update (random disk writes)
  Replication → physical WAL pages bheje
```

### MySQL InnoDB Solution
```
1 UPDATE →
  WAL mein likha
  Buffer Pool (RAM) mein in-place update
  Old version → Undo Log mein
  Replication → logical binlog row change
```

### Key Difference
| | Postgres | MySQL InnoDB |
|---|---|---|
| Update | Nayi copy likhta hai | In-place, old → undo log |
| Replication | Physical WAL pages | Logical binlog |
| Replica load | Heavy | Light |
| VACUUM needed | Haan | Nahi |

### Side Notes
- Postgres mein dead tuples VACUUM saaf karta hai
- WAL aur Binlog alag cheezein hain — MySQL mein crash recovery ke liye redo log, replication ke liye binlog
- Write Amplification = 1 logical write ke liye N physical disk writes

---

## 2. InnoDB Internals

### Buffer Pool
- RAM mein bada area — disk pages cache karta hai
- Write aaya → Buffer Pool (RAM) update, disk nahi
- Background mein dirty pages flush hoti hain
- Clean page = RAM aur disk same. Dirty page = RAM mein update, disk pe nahi

### Redo Log (WAL)
```
Write aaya →
  1. Redo Log pe likha (sequential, disk)
  2. Buffer Pool update (RAM)
  3. User ko success

Crash hua?
  Redo Log dekho → jo miss hua apply karo
```
- Circular files: `ib_logfile0`, `ib_logfile1`
- Checkpoint = "yahan tak disk pe flush ho gayi"
- Log file bada rakhne se fewer forced checkpoints

### Files Overview
| File | Kaam |
|---|---|
| `*.ibd` | Actual data + indexes |
| `ib_logfile*` | Crash recovery (redo log) |
| `ibdata1` | Undo log, metadata |
| `binlog.*` | Replication ke liye |

---

## 3. LSM Tree — Complete Flow

### Architecture
```
Write → WAL → MemTable (RAM, sorted) → SSTable (Disk, sorted, immutable)
                                              ↓ (compaction)
                                        Merged SSTable
```

### MemTable
- RAM mein sorted structure (Red-Black Tree ya Skip List)
- Same key aaya → overwrite (replace) — space same rehta hai
- Flush conditions: Size limit, time limit, WAL bada hua, shutdown

### SSTable
- Sorted, immutable file on disk
- Andar: Data Blocks + Sparse Index + Bloom Filter
- Har SSTable ki apni sparse index hoti hai (RAM mein)

### Why Sequential Writes
- LSM append-only hai — in-place update nahi
- MemTable flush → ek baar sequential disk write
- B-Tree mein random writes (fixed locations update karne padte hain)

### Read Flow
```
Query aai:
1. MemTable check (latest data)
2. Bloom Filter (exist karta hai?)
3. Sparse Index (konsa block?)
4. Binary Search in block
5. Result ✓
```

### Compaction
- Multiple SSTables → merge sort → ek badi sorted SSTable
- Latest version rakho, old versions aur tombstones delete karo
- Background mein hota hai

---

## 4. Bloom Filter

### What
RAM mein bit array + k hash functions = fast membership test

### How
```
Add element:
  k hash functions → k positions → bits set to 1

Check element:
  Same k positions dekho
  Koi bhi 0 → "Definitely nahi hai" (100% accurate)
  Saare 1   → "Shayad hai" (false positive possible)
```

### Key Properties
- False Negative: KABHI NAHI
- False Positive: Possible (but controllable)
- Delete: Possible nahi (bits shared hain)
- Memory: 10 crore elements → ~125MB (vs 5GB for full set)

### Formula
```
k = (m/n) × ln(2)
k = hash functions, m = bit array size, n = expected items
```

---

## 5. B-Tree Internals

### Structure
```
Internal Nodes: Keys + References (guide only, no data)
Leaf Nodes:     
  Primary (Clustered): actual row data
  Secondary: primary key pointer
```

### InnoDB Clustered Index
- Primary Key = Data (leaf node mein actual row)
- No separate heap file needed
- Secondary index → stores primary key (not offset)
- 1 hop: Primary key query → direct data
- 2 hops: Secondary key query → PK → data

### Postgres (Non-Clustered)
```
Index File (alag): city=Lucknow → heap offset 100
Heap File (alag): Offset 100 → actual row

3 alag files per write:
  likes_heap + primary_idx + secondary_idx
= Maximum write amplification
```

---

## 6. Clustered vs Non-Clustered Index

| | Clustered | Non-Clustered |
|---|---|---|
| Data location | Leaf node mein | Heap file mein |
| Lookup | 1 hop | 2 hops |
| Example | InnoDB PK, LSM SSTable | Postgres B-Tree |
| Count per table | Sirf 1 | Multiple |
| Source of truth | Index itself | Heap file |

### LSM = Clustered by Nature
- SSTable mein key + value saath hain
- Alag heap file nahi
- Secondary index → PK store karta hai (not offset)
- Compaction mein data move hota hai → offset invalid → isliye PK store karo

---

## 7. Secondary Index in LSM

### Structure
```
Main MemTable:      (post_id, user_id) → {data}
Secondary MemTable: (city, post_id)    → ""  ← empty value

Flush:
Main SSTable + Secondary SSTable (alag files, simultaneous flush)
```

### When Secondary Index Changes
- Insert: New entry add karo
- Delete: Tombstone likho
- Indexed column change: Purani tombstone + nayi entry (2 ops)
- Non-indexed column change: Secondary index touch nahi hota ✓

### Read Flow
```
WHERE city = 'Lucknow':
  Secondary SSTable → Bloom Filter → Sparse Index → Binary Search
  → PK mila → Main SSTable → actual row
```

---

## 8. Heap File Concept

### What
- Generic term for "woh file jahan actual rows hain"
- Postgres: `base/db_oid/table_oid`
- MySQL: `.ibd` file (but InnoDB mein clustered — no separate heap)
- LSM: SSTable (sorted + immutable, not really "heap")

### Key Insight
```
Non-Clustered (Postgres):
  Index → heap file offset → data
  Update → row moves → ALL indexes update karo

Clustered (InnoDB/LSM):
  Index IS the data
  No heap file
  Secondary → PK (stable) → data
```

---

## 9. Write Amplification

### Definition
```
1 logical write ke liye N physical disk writes = Write Amplification N×
```

### B-Tree (Postgres) — Worst Case
```
1 UPDATE →
  WAL write
  Heap file write
  Primary index update
  Secondary index 1 update
  Secondary index 2 update
= 5 disk writes (write amp = 5×)
```

### LSM — Better
```
Write aate hi:
  WAL write (1 disk write)
  MemTable update (RAM)
= 1 disk write abhi

Compaction mein thoda amplification hota hai
But sequential hai (fast)
```

---

## 10. Sequential vs Random Writes

### Disk Analogy
```
Sequential: Needle ek direction mein chali → 200 MB/s
Random:     Needle baar baar jump kari    →   2 MB/s
Difference: 100×
```

### Why B-Tree = Random
- Data fixed locations pe hai
- Update → us location pe jaana padta hai
- Multiple locations = multiple seeks

### Why LSM = Sequential  
- Append-only — kabhi in-place update nahi
- MemTable flush → ek baar sequential write
- Needle ek direction mein chali

---

## 11. DynamoDB Internals

### Storage Engine
- Likely B-Tree based (exact details secret)
- Original Dynamo: Berkeley DB + MySQL (pluggable)

### Partitioning
```
post_id → MD5 hash → partition number → Node
Consistent hashing se distribute
Har partition: 10GB max, 3000 RCU, 1000 WCU
```

### GSI vs LSI

| | GSI | LSI |
|---|---|---|
| Partition Key | Different (new) | Same as table |
| Node | Alag node | Same node |
| Consistency | Eventually consistent | Strongly consistent |
| Storage | Data copy → 2× size | No copy → same size |
| Cost | 2× | Same |
| Size limit | Unlimited | 10GB per partition |
| Create when | Anytime | Table creation time only |
| Throughput | Apna separate | Table ke saath share |

### Why GSI = Eventually Consistent
```
Main Table: post_id=101 → Node 1
GSI:        city=Lucknow → Node 5 (alag node)

Write aaya → Node 1 update hua
Node 5 ko async propagate kiya
= Eventually consistent
```

### Why LSI = Strongly Consistent
```
Same partition key = Same node
Write aaya → Same node update
No network hop → Strong consistency free mein
```

### GSI Data Copy — Why
```
GSI alag node pe:
Query → Node 5 → PK mila → Node 1 → data
= Cross node network call = latency

Solution: Poora data copy karo Node 5 pe
= No cross node hop
= Fast reads
= But 2× storage :(
```

---

## 12. Elasticsearch Internals

### Architecture
```
Elasticsearch = Lucene (search engine)
              + Distributed coordination layer

Har shard = Standalone Lucene index
```

### Inverted Index
```
Normal:     Document → Words
Inverted:   Word → Documents

"prady" → [Doc1, Doc3]
"lucknow" → [Doc1]

Search "prady lucknow":
Intersection → Doc1 (both words) ← highest score
```

### Lucene Segments (Same as LSM!)
```
LSM:           Lucene:
MemTable    →  In-Memory Buffer
SSTable     →  Segment (immutable)
Compaction  →  Merge
WAL         →  Translog
```

### Delete in Lucene
- Turant delete nahi — deletion bitmap mein flag
- Merge pe actually delete hota hai
- = Tombstone jaisi cheez

### Tokenization
```
"Prady lives in Lucknow city"
→ Tokenize: ["prady", "lives", "lucknow", "city"]
→ Stop words remove: "in" gone
→ Lowercase + stem
→ Inverted index mein store
```

---

## 13. Instagram Like System Design

### Schema
```
Likes Table (LSM):
PK = post_id
SK = user_id  
Value = {timestamp, status}

Posts Table:
PK = post_id
Value = {content, owner, total_likes (sharded counter)}

GSI on Likes (B-Tree, async):
PK = user_id
SK = post_id
```

### 3 Queries
```
Q1: "Did I like?" → Primary index (post_id, user_id) direct lookup
    Strong consistent, no GSI needed ✓

Q2: "Total likes?" → Sharded counter in Posts table
    Aggregate shards, eventually consistent ✓

Q3: "All posts I liked?" → GSI (user_id, post_id)
    Eventually consistent okay ✓
```

### Hot Key Problem (Virat Kohli)
```
Problem: post_id=101 → single partition → overload

Solution: Key Sharding
post_id=101#shard0 → Node 3
post_id=101#shard1 → Node 7
...
post_id=101#shard9 → Node 5

Write: hash(user_id) % 10 = shard (deterministic!)
Read:  Sum all 10 shards (parallel)
```

### Atomic Counter — Race Condition
```
Bina atomic:
User A reads 5000, User B reads 5000
Both write 5001 → Lost update!

With atomic increment:
DynamoDB atomic operation → no race condition

Sharded for hot posts:
10 shards → 10 atomic counters → sum on read
```

### Transaction Failure
```
Like inserted → Counter increment failed?

Options:
1. DynamoDB TransactWrite (atomic, but 2× cost)
2. Eventual consistency accept karo (approximate count okay)
3. Event driven: Like insert → Kafka event → async counter update
```

### Real-time Likes (WebSocket + Pub/Sub)
```
Like aaya
    ↓
Like Service
    ↓
DB insert + Redis counter + Kafka publish
    ↓
Kafka topic "post:101"
    ↓
All WebSocket servers subscribe
    ↓
Each server pushes to connected users
    ↓
50 lakh screens update ✓

SQS = point to point (1 message → 1 consumer) ✗
Kafka/Redis Pub/Sub = broadcast (1 → N) ✓
```

---

## 14. Compaction Strategies

### Size-Tiered Compaction
```
Similar size SSTables merge karo:
[10MB][10MB][11MB] → [31MB]
[31MB][30MB][32MB] → [93MB]

✓ Write friendly (fast)
✓ Write heavy workloads
✗ Space amplification (old + new simultaneously)
✗ L0 pile up under heavy traffic
```

### Leveled Compaction
```
Fixed size levels:
L0: 4 SSTables (small, any overlap)
L1: 10MB total (non-overlapping key ranges)
L2: 100MB total
L3: 1GB total

L0 full → merge into L1
L1 full → merge into L2

✓ Read friendly
✓ Less space amplification
✓ Non-overlapping = fewer files to check
✗ More write amplification
✗ Slower writes
```

### When to Use
```
Size-Tiered: Write heavy (IoT, logs, metrics, likes)
Leveled:     Read heavy (user profiles, transactions)
```

### Production Problem
```
Sale event:
Writes 100× → MemTable jaldi flush
→ L0 SSTables pile up
→ Cassandra writes throttle karta hai
→ Write latency 200ms spike

Fix:
Short term: nodetool compact, concurrent_compactors badhao
Long term: MemTable size badhao, pre-scale on known events
```

---

## 15. Quick Reference — Which DB For What

| Workload | DB Choice | Why |
|---|---|---|
| Write heavy | Cassandra, RocksDB (LSM) | Sequential writes |
| Read heavy | MySQL, Postgres (B-Tree) | Fast reads |
| Both heavy | Hybrid: LSM primary + B-Tree GSI | Best of both |
| Full text search | Elasticsearch (Lucene) | Inverted index |
| Real-time semantic | Vector DB (Pinecone, Qdrant) | Embeddings |
| Hot key writes | Sharded counters + async aggregation | Distribute load |

---

## 16. Key Concepts — Side Notes to Remember

```
1. WAL = crash recovery, always sequential, both LSM and B-Tree use it
2. Binlog = replication (MySQL), separate from WAL
3. MemTable replace karta hai same key ke liye (space same)
4. SSTable immutable — update nahi hoti
5. Tombstone = logical delete, physical delete at compaction
6. Bloom Filter = "definitely not" guarantee, false positives possible
7. Sparse Index = navigation tool for SSTable blocks
8. Clustered Index = data IS the index (InnoDB, LSM by nature)
9. Non-clustered = index → pointer → separate data (Postgres)
10. GSI eventually consistent kyunki alag node pe async replication
11. LSI strongly consistent kyunki same node, no network hop
12. GSI storage 2× kyunki data copy hoti hai (cross-node hop avoid)
13. Hot key = shard karo, deterministic hash use karo
14. Pub/Sub = broadcast (Kafka/Redis), SQS = point-to-point
15. WebSocket = persistent connection, server push possible
16. Write Amplification = zyada writes = SSD wear = slow performance
17. Sequential writes 100× faster than random writes on disk
18. Compaction = merge sort on SSTables, removes old versions + tombstones
```

---

## 17. Interview Cheat Sheet — Common Questions

**Q: LSM vs B-Tree kab choose karein?**
Write heavy → LSM. Read heavy → B-Tree. Both → Hybrid.

**Q: Why is secondary index eventually consistent in DynamoDB GSI?**
Alag node pe hai → async replication → eventual consistency.

**Q: Hot key problem kaise solve karein?**
Detect → Key shard karo (deterministic) → Parallel writes → Async aggregate.

**Q: Read your own write guarantee kaise doge?**
Primary key se direct lookup karo (not GSI). GSI eventually consistent hai.

**Q: Compaction spike on sale event kaise handle karein?**
Pre-scale, MemTable size badhao, concurrent compaction threads badhao.

**Q: Real-time like count kaise show karein?**
WebSocket connections + Kafka Pub/Sub broadcast to all WebSocket servers.

**Q: Delete in LSM kaise hota hai?**
Tombstone likho → Compaction mein actually delete hoga.

---

*Boom chika! 🎯 — Prady's DB Internals Revision*

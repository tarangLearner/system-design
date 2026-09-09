data structures
1. list/array
2. stack
3. queue
4. Heap -> task schduling for memory managment 
5. trees -> hirechical data
6. hash table -> its effeicit to look data , insertion and deletion used -> search engine, cache system, compiler/interpreter
7. suffiex tree -> searching strings in documents, this is perfect for text editors and search algo
8. Graphs -> tracking relationship and provide it
9. R-trees -> good at finding nearest neighbors, it stores spatial data

---

# Data Structures — Interview-Ready Reference

> **Purpose:** the complexity tables you must recall instantly, **plus** the thing most DSA notes skip — *where each structure shows up inside a real system design*.
> **Companions:** [low-level-design.md](low-level-design.md) · [databases.md](databases.md) · [caching.md](caching.md) · [system-design-interview-playbook.md](system-design-interview-playbook.md) · [README.md](README.md)

---

## Index

| # | Topic | Section |
|---|---|---|
| 1 | Big-O in 60 seconds + the growth table | [§1](#1-big-o-the-only-part-you-need) |
| 2 | **The master complexity table** | [§2](#2-the-master-complexity-table) |
| 3 | Linear: array · linked list · stack · queue · deque | [§3](#3-linear-structures) |
| 4 | Hash table — collisions, load factor, why O(1) is amortised | [§4](#4-hash-tables-the-most-important-one) |
| 5 | Trees: BST · balanced · **B/B+Tree** · heap · trie · segment/Fenwick | [§5](#5-trees) |
| 6 | Graphs: representations, BFS/DFS, shortest path, topological sort | [§6](#6-graphs) |
| 7 | Spatial: R-tree · quadtree · geohash · S2 | [§7](#7-spatial-structures) |
| 8 | Probabilistic: bloom filter · HyperLogLog · count-min sketch · t-digest | [§8](#8-probabilistic-structures-system-design-favourites) |
| 9 | **Which structure does each system use?** ⭐ | [§9](#9-where-each-structure-shows-up-in-system-design) |
| 10 | Structure selection flow chart | [§10](#10-how-to-choose) |
| ★ | Rapid-fire Q&A | [§11](#11-rapid-fire-qa) |
| ★ | 🏭 **Real-world: hash rings · buddy allocation · sparse indexes · version stamps** | [§12](#12-real-world-case-study--these-structures-inside-uber-and-linkedin-infrastructure) |

---

## 1. Big-O — the only part you need

| Growth | Name | n = 1,000,000 |
|---|---|---|
| O(1) | Constant | 1 |
| O(log n) | Logarithmic | ~20 |
| O(n) | Linear | 1,000,000 |
| O(n log n) | Linearithmic | ~20,000,000 |
| O(n²) | Quadratic | 10¹² ❌ |
| O(2ⁿ) | Exponential | ☠️ |

> ⭐ **The practical takeaway:** the gap between O(n) and O(log n) at a million elements is **50,000×**. That is the entire justification for indexes, tries, heaps and hash tables.

**Amortised vs worst case:** a dynamic array's `push` is O(1) *amortised* — most pushes are O(1), but a resize copies everything in O(n). Averaged over n pushes it's O(1). ⚠️ For a **latency-sensitive** service, the amortised average hides a p99 spike — say this if asked about tail latency.

---

## 2. The Master Complexity Table

| Structure | Access | Search | Insert | Delete | Space | Ordered? |
|---|---|---|---|---|---|---|
| **Array (static)** | **O(1)** | O(n) | O(n) | O(n) | O(n) | by index |
| **Dynamic array** | **O(1)** | O(n) | O(1)* at end | O(n) | O(n) | by index |
| **Singly linked list** | O(n) | O(n) | **O(1)** at head | **O(1)** given the node | O(n) | insertion |
| **Doubly linked list** | O(n) | O(n) | **O(1)** | **O(1)** given the node | O(n) | insertion |
| **Stack** | O(n) | O(n) | **O(1)** | **O(1)** | O(n) | LIFO |
| **Queue** | O(n) | O(n) | **O(1)** | **O(1)** | O(n) | FIFO |
| **Hash table** | — | **O(1)*** | **O(1)*** | **O(1)*** | O(n) | ❌ **no order** |
| **BST (unbalanced)** | O(n)⚠️ | O(n)⚠️ | O(n)⚠️ | O(n)⚠️ | O(n) | ✅ in-order |
| **Balanced BST** (AVL / Red-Black) | O(log n) | **O(log n)** | **O(log n)** | **O(log n)** | O(n) | ✅ |
| **B / B+Tree** | O(log n) | **O(log n)** | O(log n) | O(log n) | O(n) | ✅ + **range scans** |
| **Binary heap** | O(1) for min/max | O(n) | **O(log n)** | **O(log n)** pop | O(n) | partial |
| **Trie** | — | **O(L)** L = key length | O(L) | O(L) | O(alphabet × nodes) | ✅ lexicographic |
| **Skip list** | O(log n)* | O(log n)* | O(log n)* | O(log n)* | O(n log n) | ✅ |
| **LSM tree** | — | O(log n) + read amp | **O(1)** buffered | O(1) tombstone | O(n) | ✅ per SSTable |
| **Graph (adj. list)** | — | O(V+E) traverse | O(1) add edge | O(E) | O(V+E) | ❌ |

\* amortised / expected

### Sorting, for completeness

| Algorithm | Best | Average | Worst | Space | Stable | Note |
|---|---|---|---|---|---|---|
| **Quicksort** | O(n log n) | O(n log n) | **O(n²)** | O(log n) | ❌ | Fastest in practice; randomise the pivot |
| **Mergesort** | O(n log n) | O(n log n) | **O(n log n)** | O(n) | ✅ | ⭐ **The basis of external sorting** — sorting data bigger than RAM |
| **Heapsort** | O(n log n) | O(n log n) | O(n log n) | **O(1)** | ❌ | In-place, predictable |
| **Timsort** | O(n) | O(n log n) | O(n log n) | O(n) | ✅ | Python/Java default; exploits existing runs |
| **Counting/Radix** | O(n+k) | O(n+k) | O(n+k) | O(n+k) | ✅ | Only for bounded integer keys |

---

## 3. Linear Structures

### 3.1 Array vs Linked List — the real answer

| | Array | Linked List |
|---|---|---|
| Memory | **Contiguous** | Scattered nodes + pointers |
| Random access | **O(1)** | O(n) |
| Insert/delete in middle | O(n) (shift) | O(1) *given the node* |
| Memory overhead | None | 8–16 bytes per node for pointers |
| **Cache locality** ⭐ | **Excellent** | **Terrible** |

> ⭐ **The answer that separates levels:** *"In theory a linked list wins on insertion. In practice arrays usually win anyway because of **cache locality** — an array walk prefetches sequential cache lines, while a linked-list walk is a chain of dependent cache misses at ~100 ns each. Modern CPUs make O(n) over contiguous memory beat O(1) over scattered memory for anything under a few thousand elements."*

### 3.2 Stack (LIFO)

**Used for:** call stacks · undo/redo ([design-patterns.md](design-patterns.md#command)) · expression evaluation · balanced-bracket checks · DFS (iterative) · browser back button · JVM/interpreter frames · backtracking.

### 3.3 Queue (FIFO) & variants

| Variant | Use |
|---|---|
| **Queue** | BFS · task scheduling · **message queues** ([distributed-systems.md](distributed-systems.md)) · request buffering |
| **Deque** | Sliding-window maximum · LRU recency list · work-stealing pools |
| **Circular buffer / ring buffer** ⭐ | Fixed-memory logging, audio/video buffers, metrics windows, Kafka-style batching, `Disruptor` |
| **Priority queue (heap)** | Dijkstra · task priorities · rate limiter expiry · timer wheels |
| **Blocking queue** | Producer-consumer with backpressure ([concurrency.md](concurrency.md)) |

---

## 4. Hash Tables (the most important one)

```mermaid
flowchart LR
    K["key 'user:42'"] --> H["hash(key)"] --> M["% bucket_count"] --> B["bucket 7"]
    B --> C["collision handling:<br/>chaining (linked list/tree)<br/>or open addressing (probe)"]
```

| Concept | Detail |
|---|---|
| **Collision** | Two keys hash to the same bucket. **Unavoidable** (pigeonhole principle) |
| **Chaining** | Each bucket holds a list. Java 8+ converts a bucket to a **red-black tree** past 8 entries, so worst case becomes O(log n) instead of O(n) |
| **Open addressing** | Probe for the next free slot (linear/quadratic/double hashing). Better cache locality, worse at high load |
| **Load factor** | `entries / buckets`. Java resizes at **0.75** — a trade-off between memory and collision rate |
| **Rehashing** | On resize, every key is rehashed — an **O(n) pause**. ⚠️ Pre-size your maps in hot paths |
| **Why O(1) is "amortised"** | A single operation can hit a resize or a long chain. Worst case is O(n) |

⚠️ **Hash-flooding DoS:** an attacker sends keys that all collide, turning every lookup into O(n) and pinning your CPU. **Mitigation:** randomised hash seeds per process (Python, Java, Ruby all do this now) and treeified buckets. This is a genuine OWASP-relevant issue — worth naming.

> 💡 **From the original note above:** use a **HashSet** when you only need presence/absence, not a HashMap with dummy values. Candidates get dinged for this.

**LRU Cache = HashMap + Doubly Linked List** — the single most-asked structure-design question:

```mermaid
flowchart LR
    HM["HashMap<br/>key → node"] -->|"O(1) lookup"| N["Node in the list"]
    DLL["Doubly linked list<br/>MRU ⟷ ... ⟷ LRU"] -->|"O(1) move-to-front,<br/>O(1) evict from tail"| E["Evict"]
```
Both `get` and `put` are **O(1)**: the map finds the node, the list maintains recency. In JavaScript a `Map` preserves insertion order, so it doubles as the recency list — see the implementation in [cache-aside-lld.md](cache-aside-lld.md) §6.

---

## 5. Trees

### 5.1 BST → balanced BST

An unbalanced BST built from sorted input degenerates into a linked list — **O(n)**. That's why production code uses self-balancing trees:

| Tree | Balance rule | Character |
|---|---|---|
| **AVL** | Height difference ≤ 1 | Stricter → **faster reads**, more rotations on write |
| **Red-Black** | Colour invariants | Looser → **faster writes**. Used by Java `TreeMap`, C++ `std::map`, Linux scheduler |
| **Skip list** | Probabilistic layers | Simpler to make lock-free → Redis **sorted sets**, LevelDB memtable |

### 5.2 ⭐ B-Tree vs B+Tree — why databases don't use BSTs

> **The reason is disk, not asymptotics.** A binary tree over 1 billion keys is ~30 levels deep = 30 random disk reads. A B+Tree with a fanout of ~500 is **~4 levels** = 4 reads.

| | B-Tree | B+Tree |
|---|---|---|
| Data stored in | All nodes | **Leaves only** |
| Leaves linked | ❌ | ✅ **Linked list** |
| Range scan | Requires tree traversal | ⭐ **Walk the leaf chain — sequential I/O** |
| Fanout | Lower (nodes carry data) | **Higher** (internal nodes are pure keys → more per page) |
| Used by | Some filesystems | **Postgres, MySQL/InnoDB, SQL Server, Oracle** |

> ⭐ **Say this:** *"A B+Tree node is sized to one disk page (~4–16 KB), which is why the fanout is in the hundreds and the tree is only 3–4 levels deep for a billion rows. And because the leaves are a linked list, `WHERE created_at BETWEEN ...` is a sequential leaf walk instead of a tree traversal. That's the whole reason range queries are cheap on a relational index and impossible on a hash index."* → [databases.md](databases.md) §4

### 5.3 Heap

An **almost-complete binary tree** stored in a flat array — `parent = (i-1)/2`, `children = 2i+1, 2i+2`. No pointers, perfect cache locality.

| Operation | Cost |
|---|---|
| Peek min/max | **O(1)** |
| Insert / extract | O(log n) |
| **Build heap from n items** | ⭐ **O(n)**, not O(n log n) |
| Find arbitrary element | O(n) — ⚠️ heaps aren't for searching |

**System uses:** Dijkstra & A* · **task/job scheduling** (the original note above) · **top-k** streaming (keep a min-heap of size k → O(n log k)) · timer wheels & TTL expiry · median maintenance (two heaps) · **external merge sort** (k-way merge of sorted runs).

### 5.4 Trie (prefix tree)

```mermaid
flowchart TD
    R((root)) --> C[c] --> A[a] --> T["t ✓ 'cat'"]
    A --> R2["r ✓ 'car'"] --> D["d ✓ 'card'"]
    R --> D2[d] --> O[o] --> G["g ✓ 'dog'"]
```

Lookup is **O(L)** where L is the key length — **independent of how many keys are stored**. That's the superpower.

**Used for:** autocomplete / typeahead ⭐ · spell check · **IP routing tables** (longest-prefix match) · dictionary/word games · T9 · **URL routing** in web frameworks · **radix tree** (compressed trie) in Redis and Linux page caches.

⚠️ **Memory is the cost.** A naive trie stores an array of 26+ child pointers per node. Fixes: a compressed **radix tree**, a map instead of an array per node, or a **DAWG/succinct trie**.

> ⭐ **The autocomplete answer:** *"A trie gives prefix matching in O(L), but walking the subtree to rank suggestions is expensive at query time. So I'd **precompute and cache the top-k completions at each node** offline, and rebuild that periodically — turning a query into one O(L) walk plus a constant-size read."*

### 5.5 Segment tree & Fenwick (BIT)

| | Segment tree | Fenwick / BIT |
|---|---|---|
| Range query | O(log n) — any associative op | O(log n) — prefix sums |
| Point update | O(log n) | O(log n) |
| Space | O(4n) | **O(n)** |
| Flexibility | Any op (min/max/sum/gcd), lazy propagation | Sums only, much simpler code |

**Used for:** range analytics · time-series roll-ups · leaderboards ("rank of score X") · competitive programming.

---

## 6. Graphs

### 6.1 Representations

| | Adjacency matrix | Adjacency list |
|---|---|---|
| Space | O(V²) | **O(V+E)** |
| "Is there an edge u→v?" | **O(1)** | O(degree) |
| Iterate a node's neighbours | O(V) | **O(degree)** |
| Best for | Dense graphs | ⭐ **Sparse graphs — i.e. almost every real graph** |

### 6.2 The algorithms and what they answer

| Algorithm | Answers | Complexity |
|---|---|---|
| **BFS** | Shortest path in an **unweighted** graph; level order | O(V+E) |
| **DFS** | Connectivity, cycle detection, path existence | O(V+E) |
| **Topological sort** | A valid ordering of dependencies | O(V+E) |
| **Dijkstra** | Shortest path, **non-negative** weights | O((V+E) log V) |
| **Bellman-Ford** | Shortest path with **negative** weights; detects negative cycles | O(V·E) |
| **A\*** | Shortest path with a heuristic — ⭐ **how map routing actually works** | depends on heuristic |
| **Union-Find (DSU)** | Connected components, cycle detection in undirected graphs | ~O(1) amortised |
| **Kruskal / Prim** | Minimum spanning tree | O(E log V) |
| **Tarjan / Kosaraju** | Strongly connected components | O(V+E) |
| **PageRank** | Node importance | iterative |

> ⭐ **System-design links:** *"Topological sort is how a **build system**, a **task scheduler with dependencies**, and a **service startup order** all work — and detecting a cycle is how you detect a circular dependency. Union-Find is how you detect a cycle when adding an edge, which is exactly the check a **friend-recommendation** or **fraud-ring** system needs."*

**Used in production for:** social graphs (friends-of-friends) · recommendation engines · **map routing** (contracted A\*) · dependency resolution (npm/Maven) · fraud detection · network topology · **Kubernetes owner references** · Git commit DAG.

---

## 7. Spatial Structures

> Expanding the original note's point 9 — this is the heart of any "design Uber/Yelp/Google Maps" question.

| Structure | Idea | Used by |
|---|---|---|
| **R-tree** | Nested bounding rectangles | PostGIS, SQLite R\*Tree, spatial indexes |
| **Quadtree** | Recursively split a plane into 4 quadrants until each holds ≤ k points | Image compression, collision detection, older map tiling |
| **KD-tree** | Split alternately on each dimension | k-NN search, ML |
| **Geohash** ⭐ | Interleave lat/long bits into a **base-32 string**; a shared prefix means spatial proximity | Redis `GEO`, Elasticsearch, DynamoDB geo |
| **S2 / H3** | Map the sphere onto space-filling-curve cells (S2) or hexagons (H3) | Google Maps, **Uber (H3)** |

### Why geohash is the interview answer

```
Geohash of a point: "tdr1y8v"
  "t"       → a huge region
  "tdr"     → ~150 km cell
  "tdr1y8v" → ~150 m cell

⭐ Two nearby points share a prefix ⇒ a proximity query becomes a STRING PREFIX query,
   which any B-tree index or key-value store can serve.
```

> ⭐ **The full "find drivers near me" answer:** *"I'd index driver locations by geohash prefix at a precision matching the search radius, and shard by that prefix. A query reads the user's cell **plus its 8 neighbours** — because the target may sit just across a cell boundary — then filters by exact haversine distance and sorts. Location updates are extremely write-heavy and highly perishable, so they go to Redis with a TTL rather than a relational table."*

⚠️ **The two geohash gotchas to name:** (1) **edge effects** — adjacent points can have completely different prefixes near a boundary, hence querying neighbours; (2) **cell size varies with latitude**, which is exactly why Google built S2 and Uber built H3.

---

## 8. Probabilistic Structures (system-design favourites)

> ⭐ **These trade a small, *bounded* error for enormous memory savings. Naming them is a strong senior signal** — most candidates never mention them.

| Structure | Answers | Error | Memory |
|---|---|---|---|
| **Bloom filter** | "Is X *definitely not* in the set?" | False positives, **never** false negatives | ~10 bits/element for 1% FP |
| **Counting bloom filter** | Same, but supports **delete** | Same | ~4× a plain bloom filter |
| **Cuckoo filter** | Same + delete + better locality | Same | Better than counting bloom |
| **HyperLogLog** | "How many **distinct** items?" | ~**0.81%** standard error | ⭐ **12 KB for billions of items** |
| **Count-Min Sketch** | "How often did X appear?" (frequency) | Over-estimates only | Sub-linear |
| **t-digest / DDSketch** | "What's the p99?" over a stream | Bounded relative error | Kilobytes |
| **MinHash / SimHash** | "How similar are these two sets/docs?" | Approximate Jaccard | Small signatures |

**Where they're used:**

| Structure | Real use |
|---|---|
| Bloom filter | **Cassandra/RocksDB SSTable skipping** · cache-penetration defence ([caching.md](caching.md)) · Chrome malicious-URL check · CDN one-hit-wonder filtering · crawler URL dedupe |
| HyperLogLog | **Redis `PFCOUNT`** · unique visitors/DAU · "unique viewers" counts · Presto/BigQuery `APPROX_COUNT_DISTINCT` |
| Count-Min Sketch | **Heavy-hitter / hot-key detection** ⭐ · trending topics · network flow monitoring |
| t-digest | p50/p95/p99 in metrics systems (Prometheus histograms, Elasticsearch percentiles) |
| MinHash | Near-duplicate detection in crawlers, plagiarism, recommendation |

> ⭐ **The line that lands:** *"For 'count unique daily visitors' I would not store 200 million user IDs in a set — that's gigabytes per day. HyperLogLog gives me the same answer within about 1% using 12 KB, and HLLs are **mergeable**, so I can union per-hour sketches to get per-day, per-week and per-month counts for free. If the product needs an exact number, I'll say so and pay for it — but for a dashboard, 1% error is invisible."*

---

## 9. Where each structure shows up in system design

| Structure | The system that runs on it |
|---|---|
| **Hash table** | Every cache · session store · in-memory index · database hash join · load-balancer flow table |
| **Doubly linked list + hash map** | **LRU cache** eviction |
| **B+Tree** | **Every relational database index** — Postgres, MySQL, Oracle |
| **LSM tree + SSTables + bloom filters** | **Cassandra, RocksDB, LevelDB, HBase, ScyllaDB** — the write-optimised stack |
| **Skip list** | **Redis sorted sets** (leaderboards, priority queues, rate limiters) · LevelDB memtable |
| **Heap / priority queue** | Job schedulers · Dijkstra · TTL expiry · top-k streaming |
| **Trie / radix tree** | Autocomplete · **IP routing tables** · URL routers · Redis internal encodings |
| **Inverted index** (map term → doc list) | **Elasticsearch, Lucene, Solr** — all full-text search |
| **Merkle tree** ⭐ | Cassandra/Dynamo anti-entropy repair · **Git** · **blockchains** · S3 integrity |
| **Consistent hash ring + virtual nodes** | Cache/shard placement — Memcached, Cassandra, DynamoDB ([load-balancer.md](load-balancer.md)) |
| **Geohash / S2 / H3** | Uber, Lyft, Yelp, Tinder, Google Maps proximity search |
| **Graph (adjacency list)** | Social graphs · recommendations · fraud rings · dependency resolution |
| **Circular buffer** | Metrics windows · audio/video buffers · Kafka batching · `LMAX Disruptor` |
| **Bitmap / bitset** | Feature flags per user · presence · Roaring bitmaps in analytics engines |
| **Count-min sketch** | Hot-key detection in a cache or shard |
| **HyperLogLog** | Unique-visitor counting at web scale |
| **Bloom filter** | Skipping expensive lookups everywhere |
| **DAG** | Airflow/Spark job graphs · Git history · build systems |

---

## 10. How to choose

```mermaid
flowchart TD
    Q{What is the dominant operation?} --> A["Lookup by exact key<br/>➡️ <b>Hash table</b>"]
    Q --> B["Lookup by key AND range/sorted scan<br/>➡️ <b>B+Tree / sorted map / skip list</b>"]
    Q --> C["Always need the min or max<br/>➡️ <b>Heap</b>"]
    Q --> D["Prefix matching<br/>➡️ <b>Trie / radix tree</b>"]
    Q --> E["Full-text search<br/>➡️ <b>Inverted index</b>"]
    Q --> F["Nearest neighbours in 2D<br/>➡️ <b>Geohash / S2 / R-tree / quadtree</b>"]
    Q --> G["Relationships and paths<br/>➡️ <b>Graph</b>"]
    Q --> H["FIFO / LIFO ordering<br/>➡️ <b>Queue / Stack</b>"]
    Q --> I["Recency-based eviction<br/>➡️ <b>Hash map + doubly linked list</b>"]
    Q --> J["'Probably absent' at huge scale<br/>➡️ <b>Bloom filter</b>"]
    Q --> K["Approximate distinct count<br/>➡️ <b>HyperLogLog</b>"]
    Q --> L["Write throughput above all<br/>➡️ <b>LSM tree / append-only log</b>"]
```

**Then ask three follow-up questions:**
1. **Does it fit in memory?** If not, you need a disk-friendly structure (B+Tree, LSM) — asymptotics stop being the whole story once random I/O is involved.
2. **Is it concurrent?** Some structures are far easier to make lock-free (skip lists, append-only logs) than others (balanced trees). → [concurrency.md](concurrency.md)
3. **Read-heavy or write-heavy?** B+Tree favours reads; LSM favours writes. This single question chooses your database.

---

## 11. Rapid-fire Q&A

| Question | Answer |
|---|---|
| **Array vs linked list?** | Array: O(1) access, contiguous, cache-friendly. List: O(1) insert given the node, but pointer-chasing kills cache performance. Arrays usually win in practice. |
| **Why is a hash table O(1) "amortised"?** | Individual operations can hit a resize (O(n)) or a long collision chain. Expected cost is O(1) with a good hash and a bounded load factor. |
| **How are hash collisions handled?** | Chaining (list, treeified past a threshold in Java 8+) or open addressing (probing). |
| **What is a load factor?** | entries ÷ buckets. Java resizes at 0.75 to bound collisions at the cost of memory. |
| **HashMap vs HashSet?** | Same structure; a set only stores keys. Use a set for pure membership checks. |
| **Why don't databases use binary search trees?** | Depth. A BST over 1B keys is ~30 levels = 30 random disk reads. A B+Tree with fanout ~500 is ~4 levels. |
| **B-Tree vs B+Tree?** | B+Tree keeps data only in leaves and links the leaves, giving higher fanout and sequential range scans. That's why every relational engine uses it. |
| **B+Tree vs LSM tree?** | B+Tree updates in place (read-optimised); LSM appends and compacts (write-optimised, with read amplification offset by bloom filters). |
| **Implement an LRU cache.** | HashMap for O(1) lookup + doubly linked list for O(1) recency updates and O(1) tail eviction. |
| **LRU vs LFU?** | LRU when recency predicts reuse (the common case). LFU when popularity is stable and some items are perpetually hot. |
| **Heap vs BST?** | Heap: O(1) min/max, O(log n) insert, no search, no ordering. BST: O(log n) search **and** full in-order traversal. |
| **How do you find the top k of a stream?** | A **min-heap of size k** → O(n log k) time, O(k) space. |
| **Why a trie for autocomplete?** | Lookup is O(key length), independent of dictionary size. Precompute top-k at each node so ranking isn't done at query time. |
| **How do you find nearby drivers?** | Geohash/S2/H3 cell as the index and shard key; query the cell plus its 8 neighbours; refine by exact distance. Store in Redis with a TTL. |
| **BFS vs DFS?** | BFS for shortest path in unweighted graphs and level-order. DFS for connectivity, cycles and topological sort. |
| **Which algorithm does map routing use?** | A\* (Dijkstra with a heuristic), plus heavy precomputation like contraction hierarchies. |
| **What is a bloom filter for?** | Cheaply proving something is **absent** so you can skip an expensive lookup. No false negatives. |
| **How do you count unique visitors at scale?** | HyperLogLog — ~12 KB, ~1% error, and sketches are mergeable across time windows. |
| **How do you detect a hot key?** | Count-Min Sketch over the request stream, or sampled request logs. |
| **What is a Merkle tree used for?** | Comparing large datasets cheaply — replica repair in Cassandra/Dynamo, Git, blockchains, S3 integrity. |
| **Adjacency matrix vs list?** | Matrix: O(V²) space, O(1) edge lookup — dense graphs. List: O(V+E) space — every real-world graph. |
| **What is a circular buffer for?** | Fixed-memory streaming: logs, metrics windows, audio buffers, batching. It never allocates after startup. |

---

## 12. Real-World Case Study — these structures inside Uber and LinkedIn infrastructure

> **Sources:** LinkedIn — *[Northguard and Xinfra](https://www.linkedin.com/blog/engineering/infrastructure/introducing-northguard-and-xinfra)* · Uber — *[CacheFront](https://www.uber.com/en-US/blog/how-uber-serves-over-40-million-reads-per-second-using-an-integrated-cache/)*, *[Intelligent load management](https://www.uber.com/in/en/blog/from-static-rate-limiting-to-intelligent-load-management/)*.
>
> Every structure below is one you already know from [§10 How to choose](#10-how-to-choose). This is where each one actually shows up in a system serving trillions of records a day.

### 12.1 Consistent hash ring — sharding *metadata*, not just data

Northguard's control plane is a **DS-RSM** (Dynamically-Sharded Replicated State Machine): a set of **vnodes** spread over a **consistent hash ring**, each vnode being a Raft group that owns one shard of the cluster's metadata.

| What's hashed | Hashed by | Why |
|---|---|---|
| Topic metadata | **Topic name** | All operations on one topic land on one coordinator |
| Range & segment metadata | **Range ID** | *"This minimises metadata hotspots"* — a busy topic's segments spread across many vnodes |

The interesting part is the **choice of hash key per entity type**. Hashing everything by topic name would have concentrated all of a hot topic's segment churn on one Raft group. Splitting the key space by entity gives you locality where you want it (topic operations) and spread where you need it (segment churn).

**Uber does the same trick in reverse, for blast radius.** CacheFront shards Redis by **partition key** — deliberately a *different* scheme from Docstore's own sharding — so that when one Redis cluster dies, *"all requests from a failed Redis shard will be distributed among all database shards"* instead of stampeding one.

> ⭐ **Say this:** *"Consistent hashing isn't only for placing data on nodes. Two shard keys worth choosing deliberately: hash **metadata** by an entity-appropriate key so the control plane doesn't develop hotspots, and hash your **cache** by a different key than your database so a cache-shard outage fans out across all database shards instead of concentrating on one."*

### 12.2 Buddy allocation — from the OS textbook to a log store

Northguard's **ranges** (its log abstraction, covering a contiguous slice of the keyspace) split and merge *"exactly the same way that the **buddy memory allocator** algorithm works"* — a range can only be merged with its unique **buddy** range.

```mermaid
flowchart TD
    R1["Range R1<br/>keyspace [0, 1)"] -->|split| R2["R2 [0, 0.5)"]
    R1 -->|split| R3["R3 [0.5, 1)"]
    R2 -->|merge with buddy| R4["R4 [0, 1)"]
    R3 -->|merge with buddy| R4

    style R1 fill:#dae8fc
    style R4 fill:#d5e8d4
```

Two properties fall out of the buddy discipline for free:

| Property | Consequence |
|---|---|
| **Total ordering survives split/merge** | Split `R1 → R2, R3`: every record in `R1` **happens-before** every record in `R2` and `R3`. Merge `R2, R3 → R4`: both happen-before `R4` |
| **Ranges of different topics inherently align** | Two streams partitioned the same way can be **joined without a shuffle**. Compare Kafka: joining a 10-partition stream with a 16-partition stream forces an expensive repartition |

> ⭐ A classic OS-course structure (buddy allocation) chosen for a distributed log, because the constraint it enforces — you may only merge with your buddy — is exactly what keeps the key-space partitioning **aligned and reversible**. Good structure choices come from the invariant you need, not from the domain the structure came from.

### 12.3 Sparse index + LSM + write-ahead log — a storage engine, disassembled

Northguard's segment store ("fps store") is a compact tour of [§5 Trees](#5-trees):

| Component | Structure | Role |
|---|---|---|
| **Write-ahead log** | Append-only sequential file | Durability. Written and `fsync`'d before the data is considered committed |
| **File-per-segment** | Immutable ~1 GB files | Segments are sealed at 1 GB, 1 hour, or replica failure — then never modified |
| **Sparse index in RocksDB** | **LSM tree** | Maps offsets to file positions. Sparse = one entry per block, not per record: the index stays in memory and you do one short scan after the lookup |
| **Direct I/O + app-level cache** | — | The broker knows which consume streams are active, so it caches what will genuinely be read next |

**Why sparse and not dense?** A dense index over trillions of records would not fit in memory. A sparse index trades a tiny bounded scan for an index small enough to keep resident — the same trade B+Tree page-level indexing and Kafka's own `.index` files make.

**Batching before flush** is the other detail: appends accumulate until *~10 ms* have passed, or a size limit, or an append-count limit. That's the classic latency-vs-throughput knob — one `fsync` amortised over many records ([latency.md §7](latency.md#7-throughput-littles-law--why-queues-explode)).

### 12.4 Sliding window — for error counting, not just for rate limiting

CacheFront's circuit breaker is a **sliding window over time buckets**:

> *"We count the number of errors on each node **per time bucket** and compute the number of errors in the sliding window width."*

| Detail | Why it matters |
|---|---|
| **Bucketed** counters, not a per-event list | O(number of buckets) memory regardless of traffic volume — the same reason you bucket a histogram instead of storing samples |
| **Proportional** short-circuiting | *"The circuit breaker is configured to short circuit **a fraction** of the requests to that node, proportional to the error count"* — then trips fully at the threshold |
| **Per node** | The window is keyed by Redis node, so one sick node doesn't trip the breaker for healthy ones |

That gradual response is the structural difference from a textbook breaker: a boolean open/closed flag oscillates; a proportional response derived from a windowed count degrades smoothly.

### 12.5 Priority queue — request tiering under overload

Uber's shedder ranks every request into tiers **t0 … t5** (t0 = critical infrastructure, t1 = the most important user-facing traffic, t5 = background pipelines) and sheds from the bottom up. Requests without an explicit priority get a default derived from the calling service.

The structural insight is a nice one for an interview: **once the queue is a priority queue, you no longer need separate queues per workload class.** Their v1 ran three physical queues (read / write / slow); after adding priority, *"we simplified the queue structure to just read and write queues. Long-running and background operations were marked with lower priority instead of having a separate queue."*

Paired with **adaptive LIFO** — FIFO at normal load, **LIFO under pressure**, because the requests at the head of an overloaded FIFO queue have already been abandoned by their callers ([concurrency.md §10](concurrency.md#10-real-world-case-study--concurrency-control-inside-ubers-databases)).

### 12.6 Version stamps — CAS across a network

The lost-update race between CacheFront's read path and its CDC consumer is solved with a structure you already know from **optimistic locking** and the **ABA problem** ([§11](#11-rapid-fire-qa)):

| Element | Implementation |
|---|---|
| **Version** | The MySQL row **timestamp**, encoded into the value stored in Redis |
| **Compare-and-set** | A **Lua script run via `EVAL`** that behaves like `MSET` but first parses the timestamps already present and only writes if the incoming value is newer |
| **Atomicity** | Redis runs the script atomically — *"in a single request instead of requiring multiple round trips"* |

Also in the same system: **negative caching**, where absent rows are stored *"with a special flag"* so repeat lookups for non-existent keys never reach the database. That's the same job a **bloom filter** does inside an LSM engine — cheaply proving absence to skip an expensive lookup — implemented here as an explicit tombstone because the key set is unbounded and the answer must be exact.

### 12.7 The pattern behind all of them

| Structure | Where it appeared | The invariant it was chosen for |
|---|---|---|
| Consistent hash ring | Northguard metadata; CacheFront Redis sharding | Even spread + minimal reshuffling; deliberately mismatched keys for blast radius |
| Buddy-allocated ranges | Northguard log abstraction | Reversible splits that preserve ordering **and** cross-topic alignment |
| LSM (RocksDB) + sparse index | Northguard segment index | Write-heavy index that must stay memory-resident |
| Write-ahead log | Northguard durability | Sequential writes; commit before mutate |
| Sliding window of buckets | CacheFront circuit breaker | Bounded memory error rate over time |
| Priority queue + adaptive LIFO | Uber Cinnamon | Shed by importance; don't serve abandoned work |
| Version-stamped CAS | CacheFront invalidation | Lost-update prevention without a distributed lock |
| Flagged tombstones (negative cache) | CacheFront | Exact absence proof for an unbounded key set |

> ⭐ **Say this:** *"Data-structure choices in infrastructure are almost never about Big-O — everything here is O(1) or O(log n) either way. They're about which **invariant** the structure enforces for free: buddy ranges give you reversible splits that keep ordering, a sparse index gives you a memory-resident index, a bucketed sliding window gives you bounded memory, and a version stamp gives you compare-and-set without a lock. That's the thing I'd reason about out loud."*

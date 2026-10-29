# TASK 

```

# Task v4: Global Collaborative Editor Architecture 

**Complete in one response. Do not ask questions. State assumptions and proceed. Prefer density: covering all seven deliverables beats exhaustive depth on three.** 

Design a real-time collaborative document platform (Google Docs plus Figma) for 50M DAU across 5 regions. 

**Definitions (fixed)**
- **Content:** plaintext and ciphertext of documents, operations, snapshots, comments, search indexes, and backups. You must classify metadata (user IDs, document IDs, timestamps, sizes, ACLs) and justify it.
- **Committed edit:** an edit acknowledged to the client as durable.
- **Failure domain:** you define the geographic and administrative failure domains (zone, region, jurisdiction) and state which are independent.
- **Region failure:** simultaneous loss of all compute and durable storage in one defined region, with surviving regions and inter-region links otherwise healthy. For EU tenants, surviving compute and durable replicas must remain inside the EU boundary. You define what a "region" means for the EU and show how enough independent EU-local replicas survive its loss.
- **RPO ≈ 0:** no committed edit is lost under a region failure.
- **RTO ≤ 2 min:** a hard maximum per event, measured from failure to the first successful edit submission by affected clients on surviving capacity.
- **Availability SLI:** successful join and submit requests ÷ total requests, over a rolling 30 days, including regional outages. Offline local editing is excluded.
- **Propagation latency:** sender submit to receiver apply, for clients within 50 ms RTT of their region.

**Requirements**
- p99 propagation under 200 ms in-region and under 800 ms cross-region
- Offline editing up to 30 days with deterministic convergence
- EU content never leaves the EU, yet EU and US users collaborate in one document
- Optional per-tenant E2EE with search and admin recovery
- 99.99% availability, RPO ≈ 0, RTO ≤ 2 min
- Clients up to 12 months old work through protocol and schema changes with zero downtime

**Reference scenario (for cost comparability)**
- Region mix: NA 35%, EU 25%, APAC 30%, LATAM 6%, MEA 4%.
- Peak concurrent sessions are 10% of DAU. 20% of those are actively editing, each sending 1 batched message per second of about 300 B on the wire.
- Average 3 collaborators per document. 10% of documents are cross-region. The share of message traffic that crosses regions is not given. State your own routing assumption.
- 2B documents, 500 KB average **logical** storage including history, before replication.
- 5% of DAU have at least one offline session per day. 0.1% of DAU stay offline more than 7 days.
- Reference prices (approximate, for comparability): vCPU-hour $0.04, SSD $0.10/GB-month, object storage $0.023/GB-month, inter-region transfer $0.02/GB, internet egress $0.08/GB.
- The scenario does not fully determine the bill. You must model replication factor, hot/cold placement, index overhead, and backups separately, and state each assumption.

**Deliverables**
1. Architecture and data model (document, operation log, snapshot, sharding, session routing, presence)
2. Consistency model and conflict resolution, with justification
3. Guarantee matrix for the three most severe requirement conflicts. Columns: property, preconditions, single-region failure behavior, partition behavior, degradation policy, **decision, cost, and sacrificed property**. State what cannot be met.
4. Failure analysis: full regional outage (including fencing, EU-internal survivors, and request-weighted availability impact), and a poison-pill edit that crashes every receiving client
5. Capacity and cost: reference-scenario arithmetic, latency budget, normalized $/DAU/month, chosen budget ceiling with defense. Identify missing workload distributions (document popularity, maximum size, log growth, reconnect bursts) and model at least one heavy-tailed scenario, for example a 5,000-editor document or a reconnect storm after failover.
6. Cuts and roadmap, with a measurable exit criterion per phase
7. Verification plan, with pass criteria 

``` 


--- 


# Scoring Rubric 

``` 

# Scoring Rubric (100 points, plus up to 10 bonus) 

**1. Architecture and data model (15)**
- 13–15: Components, data flow, and storage are explicit. Sharding, routing, and presence are covered. Each component is tied to a requirement.
- 8–12: Sound but generic, or one major area is missing.
- 0–7: Buzzword stack or disconnected parts.

**2. Consistency and conflict resolution (15)**
- 13–15: Justified CRDT, OT, or hybrid choice. Explains why replicas converge, what metadata accumulates, when garbage collection is safe (causal stability), and the cost of 30-day merges.
- 8–12: Correct but shallow, with no growth or GC analysis.
- 0–7: Factual errors.

**3. Guarantee matrix and trade-offs (20)**
- 17–20: Genuinely severe conflicts chosen. Every row has a decision, cost, and sacrificed property. The matrix is complete and coherent. Impossibilities are recognized with a defensible alternative. Metadata classification is justified.
- 11–16: Real conflicts, but vague resolutions or an incomplete matrix.
- 0–10: Trivial conflicts or "all requirements met."

**4. Failure analysis (15)**
- 13–15: Regional outage covers fencing and split-brain prevention, quorum placement across independent EU-local failure domains, which acknowledged edits survive, RTO mechanics, request-weighted availability impact, and recovery invariants. Poison-pill covers pre-commit validation, quarantine, client safe mode, and compensating operations versus erasing committed history, including snapshots and offline clients.
- 8–12: Both addressed, with mechanical gaps.
- 0–7: Generic redundancy and retries.

**5. Capacity and cost (15)**
- Reference-scenario arithmetic is correct and traceable, including replication, tiering, and the heavy-tailed scenario (6).
- Latency budget is decomposed into propagation, processing, and queueing (4).
- Normalized $/DAU is derived and the budget ceiling is defended (5).

**6. Verification (10)**
- 9–10: Property-based convergence tests, deterministic simulation, chaos experiments tied to named failure modes, and a formal model of a named component with stated invariants. Pass criteria are defined.
- 5–8: Reasonable list, weakly tied to risks.
- 0–4: "Unit and integration tests."

**7. Roadmap (5)**
- 5: Concrete cuts with rationale, phases, and measurable exit criteria.
- 2–4: No prioritization logic or no exit criteria.
- 0–1: Missing.

**8. Assumptions and communication (5)**
- 5: Labeled, consistent assumptions and a scannable structure.
- 2–4: Some implicit assumptions.
- 0–1: Disorganized.

## Grading rules
1. Score each dimension against its anchors first.
2. **Correctness gate:** a dimension containing one or more listed disqualifying errors is capped at 60% of its maximum. Multiple errors in one dimension do not lower the cap further. Each listed error belongs to the one dimension where it is listed. Apply the cap after the initial score.
3. **Deductions** are applied separately: −2 to −5 each for unjustified buzzwords, internal contradictions, or unverifiable claims stated as fact. −5 to −10 for claiming full compliance with no compromise. Do not deduct twice for the same underlying error unless it creates a distinct contradiction elsewhere.
4. Top scores require reasoning, not just naming a technology. Length earns nothing.
5. Total is capped at 40 if the response asks questions instead of completing the task.

**Disqualifying errors by dimension**
- **Architecture:** a data path that contradicts the candidate's own residency classification.
- **Consistency:** claiming no CRDT metadata growth. OT without an ordering or transform authority. Tombstone GC without causal stability.
- **Matrix:** claiming E2EE with unrestricted server-side plaintext indexing, or admin recovery with no stated trust change. Claiming all requirements are met.
- **Failure:** claiming RPO ≈ 0 with asynchronous cross-region replication and no stated loss window. Leader failover with no fencing.
- **Capacity:** double-counting traffic. Unit errors of 10× or more. Ignoring replication in storage cost.
- **Verification:** a formal model with no stated invariants.

## Bonus (+2 each, max +10)
Items: convergence proof sketch; GC protocol with safety condition; E2EE threat model; migration compatibility matrix (old and new clients versus servers); 40% budget-cut sensitivity analysis; named TLA+ invariants with a model-checking plan. If a candidate completes all six, the grader counts the best five. Bonus rewards depth, not length or jargon.

## Bands
- **90–110:** Principal.
- **75–89:** Senior.
- **55–74:** Mid-level.
- **Below 55:** Insufficient.

## Grader process
1. Two graders score independently.
2. Before real submissions, both score one deliberately flawed answer and one excellent answer to calibrate.
3. Reconcile any dimension that differs by more than 3 points.
4. Use the answer key below to check arithmetic. Accept ±2× when assumptions are stated and consistent.

## Grader answer key (sanity values, decimal units)
- **Load at peak:** 5M concurrent sessions. 1M active editors sending 1M msg/s ≈ 300 MB/s ≈ 2.4 Gbps ingress. Fan-out to 2 other collaborators, upper bound ≈ 2M deliveries/s ≈ 600 MB/s.
- **Offline users:** 5% of DAU is 2.5M reconnects/day. 0.1% of DAU is 50,000 users offline over 7 days.
- **Storage (1 PB logical = 1,000,000 GB):**
  - One copy: about $23,000/month on object storage, or $100,000/month on SSD.
  - Three full copies: about $69,000/month on object storage, or $300,000/month on SSD.
  - All before indexes, backups, and request charges.
  - One object-storage copy is about $23,000 ÷ 50M ≈ $0.00046/DAU/month.
- **Cross-region traffic (illustrative):** if 10% of message traffic crosses regions, that is about 30 MB/s at peak. Sustained for 24 hours, that is about 2.6 TB ≈ $52/day at $0.02/GB. This is an upper bound, since peak-to-average ratio reduces it. Accept other routing assumptions if stated.
- **Latency physics:** about 5 µs/km in fiber, so 10,000 km ≈ 50 ms one way. Cross-region p99 of 800 ms leaves large headroom, but synchronous cross-region commit consumes part of the 200 ms in-region budget.
- **Downtime budget:** 30 × 24 × 60 × 0.0001 = 4.32 min. A 2-minute EU-only outage at 25% of requests ≈ 0.5 equivalent minutes ≈ 11.6% of the budget.
- **EU quorum:** if the EU is one defined region, RPO ≈ 0 under its loss needs independent EU-local durable replicas elsewhere. A majority quorum across three independent EU failure domains tolerates the loss of one. A two-domain design cannot have both zero loss and write availability after a failure. Credit candidates who state this and choose which property to sacrifice. 


``` 


--- 


# Response Developer A 

``` 

# Global Collaborative Editor: Design v4

## 0. Fixed assumptions and definitions

**Failure domains**

| Domain | Definition | Independent of | Not independent of |
|---|---|---|---|
| Zone | One AZ (≥3 per cloud region) | Power, cooling, rack network | Regional control plane |
| Region | One cloud region (metro) | Compute, storage, power, network of other regions | Our binaries and config (so cell-based staged rollout), provider IAM and DNS, global control plane |
| Jurisdiction | Legal/admin domain with its own cloud accounts, KMS/HSM root, deploy pipeline, break-glass | Other jurisdictions | None (separately operated) |
| Backup domain | Separate account, object-lock, separate KMS key | All of the above | None |

**Topology.** There are 5 product regions on 11 cloud regions, with traffic shares in parentheses.
- NA: us-east (15), us-central (10), us-west (10)
- EU: eu-fra (9), eu-dub (8), eu-sto (8), plus eu-par as a cold rebuild spare
- APAC: sin (12), tyo (10), bom (8)
- LATAM: sao (6)
- MEA: uae (4)

**Voter sets** (3 Raft voters per shard group, each in a distinct region):

| Set | Regions |
|---|---|
| NA | {use, usc, usw} |
| EU | {fra, dub, sto} |
| APAC | {sin, tyo, bom} |
| LATAM | {sao, use, usc} |
| MEA | {uae, bom, fra} |

Non-EU content may sit in EU regions. The reverse is forbidden.

**EU "region"** means one EU cloud region. The EU boundary is EU/EEA territory plus an EU-established operating entity with an EU-resident HSM root. Every acked EU op is on ≥2 of 3 EU voters, so any single EU region loss leaves 2 voters (a quorum) and at least one copy of every acked op, all inside the EU.

**Out of scope (stated as unmet):** loss of two regions in one voter set, loss of the whole EU jurisdiction, and provider-wide failure. The last has only a second-provider backup restore, with RTO in hours and outside the SLO.

**Residency interpretation.** Provider infrastructure outside the EU never stores, processes, or caches EU content. That includes ciphertext, indexes, backups, logs, crash dumps, and CDN. Delivery to an authorized end-user device is the collaboration itself. Tenants can opt into stricter policy (EU-device-only, no offline cache), which has a cost shown in §3.

**Metadata classification**

| Item | Class | Location | Justification |
|---|---|---|---|
| Doc ID (random 128-bit, no embedded time or tenant) | Global routing metadata | Global directory: docID→(jurisdiction, shard group, epoch) | Opaque; required so a US user following an EU link can be routed |
| Tenant ID→jurisdiction | Global routing metadata | Global | Random ID; needed for login routing |
| User ID | Personal data | User's home jurisdiction; EU docs hold pairwise pseudonymous subject IDs | GDPR personal data |
| ACL / sharing graph | Restricted (content-adjacent) | Doc's home jurisdiction only | Reveals who collaborates with whom |
| Timestamps, sizes | Restricted | In-jurisdiction; exported globally only as buckets with k≥1,000 docs | Leak activity and content length |
| Op envelope (actor, counter, lamport, size, type) | Restricted | In-jurisdiction, cleartext even under E2EE | Server needs it to sequence. Leaks typing cadence, which is an accepted, documented leak |
| Titles, comments, thumbnails, search terms | Content | In-jurisdiction | |
| Telemetry | Hashed IDs, jurisdiction-local salt | No raw IDs leave | |

**Added workload assumptions**
- Diurnal average = 50% of peak.
- 10% of docs are E2EE.
- 8% of docs are hot (touched in the last 30 days).
- Current-state snapshot = 25% of logical size.
- Compression 2.5:1.
- Durable op bytes = 70% of wire bytes (210 B).
- Presence: 0.5 Hz × 60 B from 50% of sessions.
- Join: 70% delta-sync (5 KB), 30% full (50 KB).
- Mean session 20 min.

## 1. Architecture and data model

**Tiers.** Edge (anycast, L4 pass-through only, TLS never terminates outside the doc's jurisdiction) → Gateway (stateless; QUIC/WebSocket, auth, rate limits) → Relay (fan-out, viewport filtering) → **Document Session Owner (DSO)**, one per active doc, holding CRDT state under a lease from the shard-group leader → **Raft log** → segment and snapshot store (object storage) → search indexers. A control plane holds the directory, config, and key management.

**Sharding.** docID → 2¹⁶ vshards → ~16k Raft groups (EU ≈ 4k, ≈122k docs per group), with online split and merge. Leaders are placed in the region nearest the median editor. A group is the unit of failover. A *cell* (~2% of tenants) is the unit of rollout.

**Write path**
1. Client sends an idempotent batch keyed by opID = (actorID, counter).
2. Gateway checks auth, size, and rate.
3. DSO validates (schema, limits) and shadow-applies on the oldest and newest supported client cores.
4. DSO assigns `seq`. In parallel, it (a) fans out *tentatively* via relays and (b) appends to Raft (local NVMe fsync plus two remote followers).
5. At 2-of-3 durable, the commit watermark advances and `ACK(opIDs)` goes to the sender. The watermark piggybacks to everyone.

Receivers keep tentative ops until the watermark passes them. On a leader change they re-upload unacked ops, which is safe because ops are idempotent. Clients key their outbox on opID, not `seq`.

**Data model**

| Object | Key → fields | Storage |
|---|---|---|
| Document | docID → tenantID, homeJur, shardGroup, schemaVer, keyMode (std/e2ee), aclVer, headSeq, snapshotSeq/gen, size, created/updated | Metadata DB, Raft RF3 |
| Op log entry | (docID, seq) → opID, lamport, opType, schemaVer, keyEpoch, deps, payload (plain or ciphertext) | Raft WAL on NVMe, flushed as 64–256 KB zstd segments `{docID}/{startSeq}-{term}.seg` to object storage in primary and buddy regions |
| Snapshot | (docID, gen, snapSeq) → state blob, skip-set hash, tombstone skeleton, stateHash, keyEpoch | Hot: SSD on voters. Cold: object. Cut every 1,000 ops, 5 min active, or 256 KB tail |
| ACL, membership, restore | Log entries of the same doc | Same log, so revocation has an exact position in the total order |
| Presence | (docID, session) → pseudonymous user, cursor/selection, viewport, TTL 10 s | DSO memory only, never persisted, encrypted under E2EE |
| Directory | docID→(jur, group, epoch); group→leader (cache, validated by Raft term) | Global, replicated, minimal fields |

**History policy.** Full-fidelity ops are kept for 30 days. Older history folds into named and daily checkpoints. Tombstone *content* is dropped at 35 days, but the ID-run *skeleton* is kept forever, so late anchors still resolve deterministically.

**Session routing.** The client asks the directory for the doc's jurisdiction (cached ≤60 s) and connects to that jurisdiction's gateway. For EU-pinned docs, US clients cross the Atlantic directly, with no non-EU relays. Gateways resolve group→leader from cache, and a stale cache is corrected by an epoch-mismatch redirect. Relays outside the doc's jurisdiction are used only for unpinned tenants.

**Protocol and schema evolution (12-month clients, zero downtime)**
- The wire format is protobuf with append-only fields. Unknown fields and unknown op types are preserved and round-tripped as opaque.
- The log is immutable. Read-time upcasters (N-13…N) convert ops, so we never rewrite history.
- New semantics ship expand → dual-write → contract. Contract waits ≥13 months after release.
- A feature that old clients cannot interpret is gated per tenant (min client version) or ships as an additive embed with a fallback.
- Raft log-format bumps use a cluster-wide feature barrier after all replicas upgrade.
- Rolling deploys run N and N-1 servers side by side.

**E2EE**
- **Keys:** a per-doc DEK (XChaCha20-Poly1305) with epochs, HPKE-wrapped per member device. MLS is used above ~20 members, since TreeKEM is O(log n), which matters for 5,000-member docs.
- **Server role:** it sequences and replicates envelopes only. Clients produce signed snapshots and compact.
- **Removal:** triggers a new key epoch. Removed members keep what they already had.
- **Admin recovery:** the tenant recovery key (RKEK) is Shamir-split 3-of-5 across admin hardware keys, or held in a tenant HYOK KMS in the tenant's jurisdiction. The provider holds zero shares. Every recovery is appended to a key-transparency log visible to members.
- **Search:** see conflict C in §3.

## 2. Consistency model and conflict resolution

**Model.** Per-document strong eventual consistency over a sequenced log. There are no cross-document transactions.
- Raft gives a per-doc total commit order (linearizable per doc).
- A typed CRDT tree gives convergence.

| Data | Mechanism |
|---|---|
| Text | Fugue (non-interleaving sequence CRDT) with Peritext formatting marks, run-length-encoded IDs |
| Object properties (canvas) | LWW register per (object, prop), ordered by (lamport, actorID). Wall clock is never used |
| Hierarchy / z-order | Kleppmann move-tree (undo-redo by (lamport, actor)) plus fractional indexing. A cycle-creating move is a deterministic no-op |
| Sets (reactions, tags) | Add-wins OR-set |
| Delete vs concurrent edit | Delete wins for text. For objects, the tombstone wins over property edits and a restore op exists |

**Convergence.** Merges are commutative, associative, and idempotent (dedupe by opID), so any delivery order gives the same state. The DSO publishes `stateHash(seq)` every 1,000 ops. A client mismatch triggers resync and a divergence report.

**Client guarantees**
- Read-your-writes by local apply.
- Monotonic reads via the commit watermark.
- Causal delivery: ops with missing deps are buffered.
- Intention preservation per CRDT.

**30-day offline**
- The client keeps a local replica and a durable outbox.
- Reconnect sends `SYNC(versionVector, outbox)`. It returns a delta, or snapshot + tail past the window. Dedupe is by opID.
- Offline edits lose LWW ties to newer online edits. This is deterministic and shown in property history.
- Ops submitted after ACL revocation are rejected and parked in an owner-review inbox. We cannot verify pre-revocation authorship, so this loses some legitimate edits.
- Convergence is guaranteed for ≤35 days. Beyond that it is best-effort via the skeleton.

**Why not the alternatives**
- **OT** needs server-side transformation. That breaks E2EE, and transform cost grows with a 30-day concurrency window.
- **Figma-style server-authoritative LWW for text** loses intent.
- **CRDT without a sequencer** has no commit point for RPO, no precise revocation semantics, and no safe snapshotting.

**Costs:** per-char metadata (≈1.3–2× plain text after RLE), tombstones, and LWW surprises.

## 3. Guarantee matrix (three most severe conflicts)

**A. Durability × latency × availability under region failure**

| Property | Preconditions | Single-region failure | Partition | Degradation policy | Decision | Cost | Sacrificed |
|---|---|---|---|---|---|---|---|
| RPO≈0 | Ack only after fsync on ≥2 of 3 voters in distinct regions; ≤1 region lost | ≥1 copy of every acked op survives; 2 voters re-form quorum; 0 acked loss | Minority cannot commit (CP). Clients edit locally, shown "unsaved" | If a 2nd voter is lost, acks halt (read-only) rather than ack on 1 copy | Raft 2-of-3 cross-region; never ack on 1 copy | $4.4k/mo log replication + $4.4k segment copies. Durable-ack p99: NA 45, EU 50, MEA 90, APAC 110, LATAM 190 ms | Ack latency; minority-side write availability |
| p99 <200 ms in-region | Client ≤50 ms RTT, leader in-region, lease valid | Docs led from the failed region stall ≤5 s. Failed-region users move to survivors. LATAM→us-east adds ~115 ms RTT, p99 ≈ 270 ms | Leader serves tentative fanout only under a 2 s lease, then read-only | Tick 20→50 ms, shed presence first, viewer tier 250 ms | Tentative fanout decoupled from durable ack | Protocol complexity; ~$0 | "Visible = durable": a peer can see an op lost only if sender and all holders die (never acked, so RPO is unaffected). LATAM/MEA latency SLO during failover |
| RTO ≤2 min + 99.99% | Warm followers, ≤40% steady CPU, jittered reconnect, 3-vantage failure detector | All groups serving ≤60 s, clients ≤105 s. us-east loss = 18 weighted-seconds = 6.9% of the 259 s monthly budget | Detector quorum prevents false evacuation | Admission priority: submit > join > presence > search | Size for N+1 (60% post-failure) | +$37k/mo vs a 70%-utilization design | Efficiency. The remaining ~240 s must cover everything else. ≥14 region-grade events/month cannot be absorbed |

**B. EU residency × EU/US collaboration × EU-local survival**

| Property | Preconditions | Single-region failure | Partition | Degradation policy | Decision | Cost | Sacrificed |
|---|---|---|---|---|---|---|---|
| Residency | EU-tagged tenant; all data plane in EU voter set; policy-as-code; EU HSM root | Re-replicate only inside the EU (eu-par spare) | EU↔world cut: EU fully served, non-EU collaborators lose EU docs | Fail closed, never cross-border | No relays, CDN, index, backup, or logs outside EU. US users connect straight to EU gateways | 3-region EU footprint + spare; extra directory hop | Non-EU latency; global cross-jurisdiction search/AI |
| EU↔US collaboration | One home jurisdiction per doc (no cross-jurisdiction multi-master) | US editors reconnect to surviving EU regions (+~95 ms RTT) | US editors cut off; local edits queue, subject to device policy | Viewer mode, editor cap | Remote sessions into the home jurisdiction | $0.8k/mo cross-region traffic | US-side p99 ≈ 240–530 ms (meets 800, not 200). Offline cache of EU docs on non-EU devices if the tenant sets strict policy |
| EU survival of a region loss | 3 EU voters + spare; EU ≤40% CPU | 2 voters keep quorum. Cold: primary, buddy, and WORM copies are in 3 distinct EU regions, so ≥2 survive | If the 2 survivors partition, EU writes halt (CP) | Durability is RF2 until rebuild: hot ≈20 min, cold ≈6 h at 40 Gbps | 3 voters + 4th spare region | EU compute at 40% | A 2nd EU failure while degraded halts EU writes |

**C. E2EE × search × admin recovery (× server-side validation)**

| Property | Preconditions | Single-region failure | Partition | Degradation policy | Decision | Cost | Sacrificed |
|---|---|---|---|---|---|---|---|
| Search | Server sees ciphertext only | Client index unaffected. S2 index rebuilt in survivors | Local search works offline | S2 off → local only | S1 default: client-side encrypted index. S2 opt-in: SSE with keyed trigram tokens, padded buckets | Client storage ≈35% of synced text; saves the server index | Server-side ranking, snippets, discovery of unsynced docs. S2 leaks token equality, frequency, and search/access patterns |
| Admin recovery | RKEK 3-of-5 or tenant HYOK; shares never at provider | Unaffected (keys outside the region data plane) | Needs admin quorum online | <3 shares → data unrecoverable by design | Tenant-held recovery, transparency log | Admin burden; ~$0 infra | "Only members can read" (admins can). Provider cannot recover lost keys |
| Server merge / validation / compaction | Payload unreadable | Same as plaintext (log replication is payload-agnostic) | Same | Clients produce signed snapshots; DSO sequences only | Client-side compaction and poison defense | +40% storage on E2EE docs ≈ +$1.1k/mo; client CPU | Shadow-apply, malware scan, previews, AI. Longer poison-pill time-to-mitigate |

**What cannot be met**
1. Loss of the entire EU jurisdiction: EU tenants go unavailable, with no US failover.
2. Two simultaneous regions lost in one voter set: writes halt.
3. 200 ms in-region p99 for LATAM/MEA during failover, and for non-EU users on EU docs.
4. 99.99% if non-region causes consume more than ~240 s per 30 days.
5. E2EE together with server-side search, AI, and validation.
6. Absolute "never leaves EU": delivery to non-EU devices and their offline caches is unavoidable.
7. The 12-month client window during an emergency security fix. We force-expire the vulnerable version instead.
8. "Visible = durable" at the same time as <200 ms.

## 4. Failure analysis

### 4A. Full regional outage (eu-dub lost; compute and storage gone)

**Fencing.** The epoch is the Raft term, enforced at four layers.
1. Raft quorum.
2. Conditional object writes: segment names include the term, and manifests change by CAS on (term, version).
3. Gateways reject ops carrying a stale `leaderEpoch`.
4. A 2 s leader lease with an election timeout ≥3 s, so there is no dual-leader window. A zombie (partitioned but alive) region self-fences to read-only within 2 s, and the directory is only a cache.

**Timeline (EU)**

| T+ | Event |
|---|---|
| 0 | Failure |
| 2–3 s | Clients miss 2–3 1 Hz heartbeats |
| 3–5 s | ~1,365 EU groups that lost their leader elect new ones with fra/sto (heartbeat 0.5 s, timeout 3–4 s). 2 of 3 is a normal quorum, so no unsafe recovery is needed |
| ~10 s | Detector (3 vantage points × 3 misses) marks the region dead, withdraws BGP/DNS, bumps the directory epoch |
| 5–60 s | Clients reconnect with jitter U(2,58 s), trying all 3 EU endpoints in parallel. Tentative ops are re-uploaded |
| ≤60 s | All groups serving (server RTO) |
| ≤105 s | Last client submits, including one admission-control retry. Hard budget 120 s |

Followers hold applied state, so DSO activation is lazy per doc (50 ms–1 s).

**EU-internal survivors**
- Hot data: fra and sto each hold a full copy.
- Cold data: primary, buddy, and WORM copies sit in 3 distinct EU regions.
- Capacity: EU runs at 40%, so survivors sit at 60%.
- Rebuild: durability returns to RF3 via eu-par (hot ≈20 min, cold ≈6 h).

**Reconnect surge.**
- eu-dub carries 8% × 5M = 400k sessions, so about 6.9k joins/s over 58 s.
- At 20 ms CPU each that is 138 vCPU on top of 300 in use, against 500 provisioned in the survivors, or 88%.
- Shedding presence and search (~40 vCPU) brings it to 80%.

**Request-weighted availability.** Weighted unavailable seconds = traffic share × recovery time. The monthly budget is 0.01% × 2,592,000 s = 259.2 s.

| Lost region | Share | Worst case (120 s) | Expected (55 s) |
|---|---|---|---|
| us-east (largest) | 15% | 18 s (6.9% of budget) | 8 s |
| eu-dub | 8% | 9.6 s | 4.4 s |
| sao | 6% | 7.2 s | 3.3 s |

Surviving-region editors on docs led from the failed region add ~0.3 s weighted (≤5 s stall). EU docs with US editors add about 1 s. One such event leaves ≥93% of the budget for deploys and other faults.

### 4B. Poison-pill edit (crashes every receiving client)

**Prevention**
- The DSO shadow-applies each op on the oldest and newest supported client cores (WASM). An op that throws on any of them is rejected or capability-gated. This is inside the 150 µs DSO budget.
- Structural limits: depth, 64 KB op size, rate.
- Per-doc feature gating by capability, so new op types never reach old clients.
- CI runs coverage-guided fuzzing of decode and apply on every supported client version.

**Detection and containment (client)**
- Before applying a remote batch, the client durably writes `applying:[opIDs]`.
- If it restarts with the marker set, it counts a crash. After 2 crashes it enters **safe mode**: read-only, bisecting the batch in a sandbox to find the culprit.
- It reports `suspect(opID, clientVer, stackHash)`. This is envelope-only data, so it also works under E2EE.
- Restart backoff bounds the crash loop to 2 iterations.

**Mitigation (server)**
- When reports arrive from ≥3 distinct devices (≥2 users) or ≥10% of sessions in 60 s, the DSO appends `Quarantine{opIDs}` to the log.
- Docs with ≤3 sessions need the owner or a doc admin.
- This is a grow-only skip-set (a CRDT), so all replicas converge. Causal descendants re-anchor deterministically.
- The DSO writes a repaired snapshot (gen+1) within 30 s.
- Sync sends quarantine records *before* ops, so late and offline clients never apply the pill.
- A **class-wide kill switch** (signed denylist of op signatures, pushed on heartbeat in ≤30 s) enforces at gateways and the DSO. A client-version fence blocks the offending op class from old builds.

| T+ | Event |
|---|---|
| 0 | Pill committed |
| 0.5 s | First crashes |
| ~3 s | Relaunch, second crash |
| ~6 s | Safe mode, reports sent |
| ~10 s | Quorum, Quarantine committed |
| ~12 s | Clients resume |

The target is time-to-mitigate ≤60 s p99 and ≤120 s worst case. The blast radius is one doc (up to 5,000 sessions). Under E2EE, quarantine relies on client reports only, so mitigation is slower. Collusion to censor an op is bounded by the ≥3-device, ≥2-user rule and admin override.

## 5. Capacity and cost

### Reference arithmetic

**Load.** Peak sessions are 5M (NA 1.75M, EU 1.25M, APAC 1.5M, LATAM 0.3M, MEA 0.2M). Active editors are 1.0M, so 1.0M msg/s × 300 B = 300 MB/s ingress. With 2 receivers each, fanout is 2M msg/s = 600 MB/s. Presence: 2.5M sessions × 0.5 Hz × 60 B = 75 MB/s in, 150 MB/s out. Joins: 5M / 1,200 s = 4.2k/s, averaging 18.5 KB, so 77 MB/s.

**Routing assumption.** A cross-region doc has 1 of 3 collaborators remote, so every op crosses a region boundary exactly once. Cross-region share = 10% of ops, or 100k msg/s, 30 MB/s peak, 15 MB/s average, ≈39 TB/month.

**Log growth (scenario inconsistency).** Durable ops average 105 MB/s ≈ 9.1 TB/day raw. The 30-day full-fidelity tail alone is ≈272 TB raw (27% of the 1 PB logical total). Folding older history ~10× gives net growth ≈2.7% per month, offset by retention expiry. The model uses steady state.

**Raw peak vCPU**

| Component | Derivation | vCPU |
|---|---|---|
| Gateways and relays | Connections 5M × 20 µs = 100; edit in 1M × 30 µs = 30; edit out 2M × 30 µs = 60; presence 3.75M × 25 µs = 94 | 284 |
| DSO | 1M × 150 µs | 150 |
| Raft follower apply | 2 × 1M × 40 µs | 80 |
| Join / ACL / API | 4.2k/s × 20 ms | 84 |
| Snapshot / compaction | | 20 |
| Search index + query | | 180 |
| **Total** | | **≈800** |

**Provisioning:** 800 × 1.5 (platform overhead) ÷ 0.40 utilization = **3,000 vCPU**.

### Monthly cost (720 h; replication factors, tiers, indexes, and backups modeled separately)

| Line | Derivation | $/month |
|---|---|---|
| Compute | 3,000 × $0.04 × 720 | 86,400 |
| Internet egress | (300 + 75 + 40) = 415 MB/s avg × 2.592M s = 1,076 TB × $80/TB | 86,100 |
| Inter-region | Raft replication 218 TB + segment/backup copies 218 TB + cross-region clients 39 TB = 475 TB × $20/TB | 9,500 |
| Hot SSD | 160M docs × 145 KB = 23.2 TB; RF3 = 69.6 TB; ÷0.7 fill = 99.4 TB × $100/TB | 9,900 |
| Search index | 35% of 225 TB non-E2EE state = 78.75 TB. 25% on SSD at RF2 (56 TB provisioned) + 75% in object at 2 copies | 8,300 |
| Metadata / ACL / directory | 2 TB RF3 ÷0.7 | 900 |
| Object (primary + buddy) | 1 PB ÷ 2.5 = 400 TB × 2 × $23/TB | 18,400 |
| Backup (WORM, separate domain) | 400 TB × 1.15 retention × $23/TB | 10,600 |
| **Total** | | **≈230,100** |

**Normalized: $230.1k / 50M DAU = $0.0046 per DAU per month.** Cost drivers: compute 38%, egress 37%, storage 21%, inter-region 4%.

### Latency budget (p99, ms)

| Hop | In-region | Cross-region (EU doc, US-east receiver) |
|---|---|---|
| Client → gateway | 25 | 25 |
| Gateway auth/rate (cached) | 2 | 2 |
| Gateway → DSO | 2 | 2 |
| DSO validate + sequence | 8 | 8 |
| Relay coalescing tick | 20 | 20 |
| Relay → client | 27 | ~70 |
| Receiver decode + apply | 15 | 15 |
| Sum | 99 | 142 |
| + one loss-recovery round | +50 → ~150 (50 ms margin) | +~100 → ~240 |

Worst pair (Sydney receiver on an EU doc): one-way ≈175 ms, so about 247 ms before tails and ≈530 ms with one 280 ms recovery. That is under 800 ms with ~270 ms margin.

### Missing workload distributions (assumed and modeled)

| Missing distribution | Treatment |
|---|---|
| Doc popularity / collaborators per doc | Assume Zipf. Mean 3, p99.9 ≈ 200, tail to 5,000. Cap 100 editor slots plus viewer mode |
| Maximum size | 25 MB state, 64 KB op, paste chunked |
| Log growth | Reconciled above (fold + 35-day tombstone) |
| Reconnect bursts | Failover, release, mobile-carrier outage, Monday 9 am local. Jitter U(2,58 s), admission control |
| Offline size and duration | 0.1% of DAU = 50k users. Assume 20 MB upload and 5 MB download each: ~1 TB and 250 GB, negligible |
| E2EE share | 10% |

### Heavy-tail scenario 1: 5,000-session doc

Naively, 20% active gives 1,000 typists at 210 B/op = 210 KB/s per receiver × 5,000 = **1.05 GB/s**. Presence at O(n²) adds 0.75 GB/s. Total is 1.8 GB/s, about **$518/hour** for one doc, and the edit fanout alone is 1.75× the platform's entire 600 MB/s peak.

With policy (100 editor slots, 4,900 viewers):
- Viewers get viewport-scoped ops (≈5%) coalesced at 250 ms: ≈1.5 KB/s each = 7.4 MB/s.
- Editors receive 2.1 MB/s.
- Presence is capped at 50 cursors per viewer: 7.5 MB/s.
- Total ≈ 17 MB/s ≈ **$4.9/hour**, about 106× cheaper.
- A relay tree of 50 relays × 100 clients keeps DSO egress ≈1 MB/s.
- Editor p99 stays <200 ms. Viewer p99 ≈350 ms, a documented exception (<500 ms).
- A 5,000-client meeting-start join burst at 500 joins/s = 9 MB/s, served from the in-region snapshot cache.

### Heavy-tail scenario 2: reconnect storm

The eu-dub storm is in §4A: 400k sessions, 6.9k joins/s, 138 vCPU, 88% survivor CPU, shed to 80%.

### Budget ceiling

**$0.0075 per DAU per month ($375k/month), 1.63× the model.** Defense:
- +20% for unmodeled platform services (observability, KMS/HSM, WAF/DDoS, CI/staging).
- +10% for drills and heavy-tail surges.
- +25% for price and estimate error (compute and egress dominate).

At an assumed ≥3% paid conversion at $10 per month (≈$0.30 per DAU-month), the ceiling is ≤2.5% of revenue. If we breach it, the levers are:
1. Raise utilization from 40% to 50% (about −$17k).
2. Trim egress with tighter coalescing and viewer sampling.
3. Add private interconnect for egress.

## 6. Cuts and roadmap

**Cuts for v1**
- No SSE (S2) search for E2EE.
- No relays for unpinned docs.
- Cap of 100 editors and 1,000 viewers.
- No sao/uae local compute (LATAM/MEA served from NA/APAC).
- No BYOK/HYOK.
- Offline 7 days, 6-month client compatibility until Phase 2.
- Canvas limited to frames, shapes, and LWW properties.

| Phase | Scope | Measurable exit criterion |
|---|---|---|
| P0 (0–3 mo) | NA only: text CRDT, DSO, 3-region Raft, snapshots | p99 in-region ≤200 ms at 100k concurrent in load test; 0 lost acked ops across 50 region-kill runs; RTO p99 ≤120 s over 20 drills |
| P1 (3–6) | EU jurisdiction, residency enforcement, EU/US collaboration | EU region-failure RTO ≤120 s; 0 EU honeytoken hits in non-EU storage/logs for 30 days; EU rebuild hot ≤20 min |
| P2 (6–9) | 30-day offline, 12-month compatibility CI, poison-pill quarantine | 10⁶ randomized 30-day-offline merges converge bit-identically; 12-version client matrix green; poison game-day time-to-mitigate ≤120 s |
| P3 (9–12) | APAC, LATAM, MEA; editor slots and viewer mode; relays for unpinned docs | Cross-region p99 ≤800 ms for 30 days of probes; 400k-session reconnect drill 100% submitting in ≤120 s; 106×-class heavy-doc cost reduction reproduced |
| P4 (12–15) | E2EE with client-side search and admin recovery | Third-party crypto audit with 0 critical findings; recovery drill: 3-of-5 recovers 100% of test DEKs; no E2EE plaintext markers server-side |
| P5 (15–18) | Availability and cost hardening, optional S2 search | 99.99% SLI over 90 days including ≥1 region drill; measured $/DAU ≤$0.0075 extrapolated to 50M |

## 7. Verification plan

| Area | Method | Pass criterion |
|---|---|---|
| Commit/fencing protocol | TLA+ model plus FoundationDB-style deterministic simulation (10⁶ runs with random region kills, partitions, clock skew) | 0 lost acked ops; 0 dual-leader commits |
| CRDT convergence | Property-based tests and nightly fuzz (10⁹ permuted interleavings, including virtual-clock 30-day offline) | 0 divergent state hashes |
| RPO under region kill | Jepsen-style: kill a region mid-load at 100k ops/s; audit acked opIDs against recovered log | 0 missing acked ops in ≥1,000 kills (rule of three: <0.3% per-kill loss at 95% confidence) |
| RTO | Monthly production region-evacuation drill; ≥5,000 distributed prober clients | 100% of probers and p99.9 of real clients submit ≤120 s |
| Latency | Probers ≥1,000 per region pair, PTP/GPS-synced stamps; load test at 1.5× peak | p99 <200 ms in-region and <800 ms cross-region on rolling 24 h |
| Residency | OPA policy-as-code on IaC; daily honeytoken docs searched in non-EU storage, logs, backups, and heap dumps; egress allow-list audit; quarterly external audit | 0 hits; 0 policy violations |
| E2EE | Marker strings in E2EE docs; crypto audit; recovery drill | 0 plaintext hits server-side; 0 critical findings; 100% recovery |
| Compatibility | 12 monthly client builds × server N and N+1 on golden op logs; rolling-deploy test | Round-trips byte-exact including unknown fields; deploy error rate ≤0.001% |
| Poison pill | Injected synthetic pills; decoder/apply fuzz ≥10⁹ execs/day | Time-to-mitigate ≤120 s; ≤2 crash-loop iterations; 0 pills reach snapshots |
| Heavy tail | Replay of 5,000-session doc and 400k-session reconnect storm | Editor p99 <200 ms, viewer <500 ms, egress ≤80 Mbps; CPU <85% with presence shed |
| Availability | Prober plus server-side request SLI; monthly chaos (zone, region, partition), drills counted | ≥99.99% over rolling 30 days, 3 consecutive months |
| Cost | Quarterly bill vs model | Unit cost within ±15% of model and ≤ceiling | 

``` 


---


# Response Developer A REVISED 


``` 

# Global Collaborative Editor: Design v4.1 (revised after self-audit) 

## Audit: what was wrong and what changed 

**Numerical and logical errors**
1. **N+1 sizing was false for the largest regions.** Uniform 40% utilization gives 60% post-failure only when regions are equal (EU). us-east is 43% of NA capacity, so its loss left survivors at ~70% before the reconnect surge and ~130% with it. There is now an explicit per-voter-set sizing rule and table, which raises provisioning from 3,000 to 3,489 vCPU (§5).
2. **Reconnect-storm CPU mixed raw and overhead-adjusted vCPU** (138 raw added to 300 adjusted). The correct EU figure is 94% before shedding and 74% after (was 88% / 80%).
3. **Egress derivation "(300 + 75 + 40)" was meaningless.** It is now 0.5 × (600 + 150 + 77) = 413.5 MB/s.
4. **Raft replication traffic assumed 2.5:1 compression of 210 B entries** (218 TB). It is now uncompressed, 544 TB. I also added cold-snapshot copy traffic and object request charges, both previously missing.
5. **Totals omitted items.** The E2EE storage overhead was stated in §3 but not summed, and DSO presence CPU and the EU spare standing cost were missing. Unexplained inputs (145 KB/hot doc, snapshot vCPU, search vCPU) are now derived.
6. **Budget ceiling arithmetic was inconsistent.** $375k was 1.63× the model, but the defended contingencies sum to +55%. It is recomputed.
7. **5,000-session scenario:** editors used 210 B while the rest used 300 B, and presence was double-counted (0.75 → 0.375 GB/s). The pass criterion "≤80 Mbps" contradicted the model's own 18 MB/s (144 Mbps).
8. **"Expected 55 s" recovery had no derivation.** It is now ≈40 s: 3 s detect + 30 s mean jitter + join and retry.
9. **Residency contradiction:** a generic anycast L4 edge could carry EU-doc traffic through non-EU PoPs. EU prefixes are now announced only from EU PoPs.
10. **Tentative fan-out hole:** re-upload by receivers needed an authorization mechanism, and the argument that no dependents are orphaned was missing. I added both, plus the monotonic-reads caveat.
11. **"Second-provider backup restore" was claimed but never designed or priced.** It is now stated as not covered, with the cost of adding it quantified.
12. **"Any delivery order"** is corrected to any causally consistent order.
13. **Region count:** 11 active + 1 spare (was "11" while listing 12).
14. **Matrix mislabel:** "US-side p99 240–530 ms" (530 is Sydney).
15. **Per-doc 64–256 KB segment objects** would generate ~2B PUTs/month. They are replaced by group-packed segments.
16. **RTO is a hard maximum**, but exit criteria used "p99 over 20 drills". They now use max, and rule-of-three bounds are stated for small drill counts.
17. **Roadmap:** P3 launched LATAM/MEA while the cuts excluded sao/uae. sao/uae now arrive in P5, and LATAM/MEA are served remotely until then.
18. **Poison pill:** a 3 s "relaunch" is unrealistic for app crashes, so apply now runs in an isolated worker. I also added shadow-apply memory and version-coverage caveats and old-client handling.
19. **Parallel connect to all 3 endpoints** would triple handshakes. It is now staggered.
20. **Relay-tree DSO egress** is corrected from 1 to 1.5 MB/s.
21. **Added missing items:** SLI success definition, auth/directory dependency, zone-loss behavior, offline reconnect volume, and LWW wording.

Numbers not mentioned above were re-derived and are unchanged.

---

## 0. Fixed assumptions and definitions

**Failure domains**

| Domain | Definition | Independent of | Not independent of |
|---|---|---|---|
| Zone | One AZ (≥3 per cloud region) | Power, cooling, rack network | Regional control plane |
| Region | One cloud region (metro) | Compute, storage, power, network of other regions | Our binaries and config (so cell-based staged rollout), provider IAM and DNS, global control plane |
| Jurisdiction | Legal/admin domain with its own cloud accounts, KMS/HSM root, deploy pipeline, break-glass | Other jurisdictions | None (separately operated) |
| Backup domain | Separate account, object-lock, separate KMS key | All of the above | None |

Zone loss removes ~⅓ of groups' voters in that region. Quorum holds on the other two regions, and re-placement on surviving zones takes ≤5 min.

**Topology.** There are 5 product regions on **11 active cloud regions + 1 cold spare**, with traffic shares in parentheses.
- NA: us-east (15), us-central (10), us-west (10)
- EU: eu-fra (9), eu-dub (8), eu-sto (8), plus eu-par as the cold spare
- APAC: sin (12), tyo (10), bom (8)
- LATAM: sao (6)
- MEA: uae (4)

**Voter sets** (3 Raft voters per shard group, each in a distinct region):

| Set | Regions |
|---|---|
| NA | {use, usc, usw} |
| EU | {fra, dub, sto} |
| APAC | {sin, tyo, bom} |
| LATAM | {sao, use, usc} |
| MEA | {uae, bom, fra} |

Non-EU content may sit in EU regions. The reverse is forbidden.

**EU "region"** means one EU cloud region. The EU boundary is EU/EEA territory plus an EU-established operating entity with an EU-resident HSM root. Every acked EU op is on ≥2 of 3 EU voters, so any single EU region loss leaves 2 voters (a quorum) and at least one copy of every acked op, all inside the EU. Raft's up-to-date vote rule guarantees the new leader holds every committed entry.

**Out of scope (stated as unmet):**
- Loss of two regions in one voter set.
- Loss of the whole EU jurisdiction.
- Provider-wide failure. No second-provider copy is designed here. Adding one costs ≈+$21k/mo (460 TB storage ≈ $10.6k plus 131 TB/mo transfer ≈ $10.5k).

**Residency interpretation.** Provider infrastructure outside the EU never stores, processes, or caches EU content. That includes ciphertext, indexes, backups, logs, crash dumps, CDN, edge, and DDoS scrubbing for EU prefixes. Delivery to an authorized end-user device is the collaboration itself. Tenants can opt into stricter policy (EU-device-only, no offline cache), which has a cost shown in §3.

**SLI success.** A submit succeeds on durable ACK ≤10 s, and a join succeeds when state is delivered ≤10 s. Retries of the same opID count once. The RTO applies to affected clients with a working network path to any surviving gateway.

**Metadata classification**

| Item | Class | Location | Justification |
|---|---|---|---|
| Doc ID (random 128-bit, no embedded time or tenant) | Global routing metadata | Global directory: docID→(jurisdiction, shard group, epoch) | Opaque; required so a US user following an EU link can be routed |
| Tenant ID→jurisdiction | Global routing metadata | Global | Random ID; needed for login routing |
| User ID | Personal data | User's home jurisdiction; EU docs hold pairwise pseudonymous subject IDs | GDPR personal data |
| ACL / sharing graph | Restricted (content-adjacent) | Doc's home jurisdiction only | Reveals who collaborates with whom |
| Timestamps, sizes | Restricted | In-jurisdiction; exported globally only as buckets with k≥1,000 docs | Leak activity and content length |
| Op envelope (actor, counter, lamport, size, type) | Restricted | In-jurisdiction, cleartext even under E2EE | Server needs it to sequence. Leaks typing cadence, which is an accepted, documented leak |
| Titles, comments, thumbnails, search terms | Content | In-jurisdiction | |
| Telemetry | Hashed IDs, jurisdiction-local salt | No raw IDs leave | |

**Added workload assumptions**
- Diurnal average = 50% of peak. Capacity is held flat at peak, because state-bearing tiers do not autoscale within the RTO.
- 10% of docs are E2EE.
- 8% of docs are hot (touched in the last 30 days).
- Current-state snapshot = 25% of logical size.
- Compression 2.5:1. E2EE clients compress before encrypting, which leaks only compressed length, already visible in the envelope.
- Durable op bytes = 70% of wire bytes (210 B).
- Presence: 0.5 Hz × 60 B from 50% of sessions.
- Join: 70% delta-sync (5 KB), 30% full (50 KB), 20 ms CPU. Mean session 20 min.
- All 3 collaborators of a doc are online together (conservative fan-out).
- Open docs at peak = 5M ÷ 3 = 1.67M. Docs with an active editor ≈ 1.67M × (1 − 0.8³) ≈ 800k.
- Offline reconnects: 5% of DAU = 2.5M syncs/day ≈ 29/s at ~50 KB, about 4% of join traffic and already inside the join model.

## 1. Architecture and data model

**Tiers.** Edge → Gateway (stateless; QUIC/WebSocket, auth, rate limits) → Relay (fan-out, viewport filtering) → **Document Session Owner (DSO)**, one per active doc, holding CRDT state under a lease from the shard-group leader → **Raft log** → segment and snapshot store (object storage) → search indexers. A control plane holds the directory, config, and key management.

- **Edge:** per-jurisdiction anycast prefixes, with the EU prefix announced only from EU PoPs. This is L4 pass-through, and TLS terminates only at in-jurisdiction gateways. A US client opening an EU doc is handed an EU hostname by the directory and reaches EU gateways directly, so no non-EU edge touches the stream.
- **Auth:** 15-minute signed tokens validated locally at gateways. A directory or identity outage therefore does not fail joins or reconnects for already-authenticated users. The directory and identity service are replicated across ≥3 regions per jurisdiction.

**Sharding.** docID → 2¹⁶ vshards → ~16k Raft groups (EU ≈ 4k, ≈122k docs per group), with online split, merge, and voter-set migration. Leaders are placed in the region nearest the median editor. A group is the unit of failover. A *cell* (~2% of tenants) is the unit of rollout.

**Write path**
1. Client sends an idempotent batch keyed by opID = (actorID, counter).
2. Gateway checks auth, size, and rate.
3. DSO validates (schema, limits) and shadow-applies on a pinned set of client cores (oldest, newest, and any version where fuzzing found divergence).
4. DSO assigns a *provisional* `seq` and issues an **authorization receipt** = HMAC(group key, opID ‖ actor ‖ digest ‖ aclVer). The group key lives in Raft state. In parallel, it (a) fans out *tentatively* via relays and (b) appends to Raft (local NVMe fsync plus two remote followers).
5. At 2-of-3 durable, the commit watermark advances and `ACK(opIDs)` goes to the sender. The watermark piggybacks to everyone.

**Tentative-op rules**
- Receivers keep tentative ops and their receipts until the watermark passes them.
- On a term change, every holder re-submits its held ops to the new leader (dependencies first, deduped by opID, receipts proving the old leader authorized them). It also fetches the committed tail since its watermark and reconciles by opID, discarding provisional seqs.
- Delivery is causal, so any holder of a dependent op also holds its dependencies. A dependency can therefore be lost only if every holder of every dependent is also lost, so dependents are never orphaned.
- An op vanishes for peers only if the sender and all holders die before commit. It was never acked, so RPO is unaffected. Peers that applied it rebuild from committed state.
- Under E2EE, device signatures replace receipts.

**Data model**

| Object | Key → fields | Storage |
|---|---|---|
| Document | docID → tenantID, homeJur, shardGroup, schemaVer, keyMode (std/e2ee), aclVer, headSeq, snapshotSeq/gen, size, created/updated | Metadata DB, Raft RF3 (~1 KB/doc including ACL and segment index) |
| Op log entry | (docID, seq) → opID, lamport, opType, schemaVer, keyEpoch, deps, payload (plain or ciphertext) | Raft WAL on NVMe, flushed as **group-packed** zstd segments (≥4 MB or 15 min, per-doc offset index) `{group}/{startIdx}-{term}.seg` to object storage in primary and buddy regions |
| Snapshot | (docID, gen, snapSeq) → state blob, skip-set hash, tombstone skeleton, stateHash, keyEpoch | Hot: SSD on voters, derived locally by each replica. Cut every 1,000 ops, 5 min active, or 256 KB tail. Cold copies are written (packed) only at ≥256 KB tail or fold time |
| ACL, membership, restore | Log entries of the same doc | Same log, so revocation has an exact position in the total order |
| Presence | (docID, session) → pseudonymous user, cursor/selection, viewport, TTL 10 s | DSO memory only, never persisted, encrypted under E2EE |
| Directory | docID→(jur, group, epoch); group→leader (cache, validated by Raft term) | Global, replicated, minimal fields |

**History policy.** Full-fidelity ops are kept for 30 days. Older history folds into named and daily checkpoints. Tombstone *content* is dropped at 35 days, but the ID-run *skeleton* is kept forever, so late anchors still resolve deterministically.

**Session routing.** The client asks the directory for the doc's jurisdiction (cached ≤60 s) and connects to that jurisdiction's gateway. Gateways resolve group→leader from cache, and a stale cache is corrected by an epoch-mismatch redirect. Relays outside the doc's jurisdiction are used only for unpinned tenants.

**Protocol and schema evolution (12-month clients, zero downtime)**
- The wire format is protobuf with append-only fields. Unknown fields and unknown op types are preserved and round-tripped as opaque.
- The log is immutable. Read-time upcasters (N-13…N) convert ops, so we never rewrite history.
- New semantics ship expand → dual-write → contract. Contract waits ≥13 months after release.
- A feature that old clients cannot interpret is gated per tenant (min client version) or ships as an additive embed with a fallback.
- Raft log-format bumps use a cluster-wide feature barrier after all replicas upgrade.
- Rolling deploys run N and N-1 servers side by side.

**E2EE**
- **Keys:** a per-doc DEK (XChaCha20-Poly1305) with epochs, HPKE-wrapped per member device. MLS is used above ~20 members, since TreeKEM is O(log n), which matters for 5,000-member docs.
- **Server role:** it sequences and replicates envelopes only. Clients produce signed snapshots and compact.
- **Removal:** triggers a new key epoch. Removed members keep what they already had.
- **Admin recovery:** the tenant recovery key (RKEK) is Shamir-split 3-of-5 across admin hardware keys, or held in a tenant HYOK KMS in the tenant's jurisdiction. The provider holds zero shares. Every recovery is appended to a key-transparency log visible to members.
- **Search:** see conflict C in §3.

## 2. Consistency model and conflict resolution

**Model.** Per-document strong eventual consistency over a sequenced log. There are no cross-document transactions.
- Raft gives a per-doc total commit order (linearizable per doc).
- A typed CRDT tree gives convergence.

| Data | Mechanism |
|---|---|
| Text | Fugue (non-interleaving sequence CRDT) with Peritext formatting marks, run-length-encoded IDs |
| Object properties (canvas) | LWW register per (object, prop), ordered by (lamport, actorID). Wall clock is never used |
| Hierarchy / z-order | Kleppmann move-tree (undo-redo by (lamport, actor)) plus fractional indexing. A cycle-creating move is a deterministic no-op |
| Sets (reactions, tags) | Add-wins OR-set |
| Delete vs concurrent edit | Delete wins for text. For objects, the tombstone wins over property edits and a restore op exists |

**Convergence.** The data types are op-based CRDTs. Concurrent ops commute, delivery is causal, and ops are idempotent (dedupe by opID), so **any causally consistent delivery order** gives the same state. The DSO publishes `stateHash(seq)` every 1,000 ops. A client mismatch triggers resync and a divergence report.

**Client guarantees**
- Read-your-writes by local apply.
- Monotonic reads via the commit watermark. The one exception is the rare rollback of a tentative op that lost all holders.
- Causal delivery: ops with missing deps are buffered.
- Intention preservation per CRDT.

**30-day offline**
- The client keeps a local replica and a durable outbox.
- Reconnect sends `SYNC(versionVector, outbox)`. It returns a delta, or snapshot + tail past the window. Dedupe is by opID.
- An offline replica's lamport clock lags, so its LWW writes lose to concurrent online writes even when later in wall-clock time. This is deterministic and shown in property history.
- Ops submitted after ACL revocation are rejected and parked in an owner-review inbox. We cannot verify pre-revocation authorship, so this loses some legitimate edits.
- Convergence is guaranteed for ≤35 days. Beyond that it is best-effort via the skeleton.

**Why not the alternatives**
- **OT** needs server-side transformation. That breaks E2EE, and transform cost grows with a 30-day concurrency window.
- **Figma-style server-authoritative LWW for text** loses intent.
- **CRDT without a sequencer** has no commit point for RPO, no precise revocation semantics, and no safe snapshotting.

**Costs:** per-char metadata (≈1.3–2× plain text after RLE), tombstones, and LWW surprises.

## 3. Guarantee matrix (three most severe conflicts)

**A. Durability × latency × availability under region failure**

| Property | Preconditions | Single-region failure | Partition | Degradation policy | Decision | Cost | Sacrificed |
|---|---|---|---|---|---|---|---|
| RPO≈0 | Ack only after fsync on ≥2 of 3 voters in distinct regions; ≤1 region lost | ≥1 copy of every acked op survives; 2 voters re-form quorum; 0 acked loss | Minority cannot commit (CP). Clients edit locally, shown "unsaved" | If a 2nd voter is lost, acks halt (read-only) rather than ack on 1 copy | Raft 2-of-3 cross-region; never ack on 1 copy | $10.9k/mo Raft replication (544 TB) + $5.2k/mo segment and snapshot copies (262 TB). Durable-ack p99: NA 45, EU 50, MEA 90, APAC 110, LATAM 190 ms | Ack latency; minority-side write availability |
| p99 <200 ms in-region | Client ≤50 ms RTT, leader in-region, lease valid | Docs led from the failed region stall ≤5 s. Failed-region users move to survivors. LATAM→us-east adds ~115 ms RTT, p99 ≈ 270 ms | Leader serves tentative fanout only under a 2 s lease, then read-only | Tick 20→50 ms (in-region p99 ≈130 ms), shed presence first, viewer tier 250 ms | Tentative fanout decoupled from durable ack; receipts let holders re-submit | Protocol complexity; ~$0 | "Visible = durable": peers can briefly see an op that is later rolled back, only if sender and all holders die (never acked, so RPO is unaffected). LATAM/MEA latency SLO during failover |
| RTO ≤2 min + 99.99% | Warm followers, per-set sizing (below), jittered reconnect, 3-vantage failure detector | All groups serving ≤60 s, clients ≤105 s. us-east loss = 18 weighted-seconds worst case = 6.9% of the 259 s monthly budget | Detector quorum prevents false evacuation | Admission priority: submit > join > presence > search | Per-voter-set sizing: largest-member loss leaves survivors ≤80% after shedding; steady ≤40% | +$49k/mo vs a uniform 70%-steady design (3,489 vs 1,774 vCPU) | Efficiency (36% overall steady utilization). The remaining ~240 s must cover everything else. ≥14 region-grade events/month cannot be absorbed |

**B. EU residency × EU/US collaboration × EU-local survival**

| Property | Preconditions | Single-region failure | Partition | Degradation policy | Decision | Cost | Sacrificed |
|---|---|---|---|---|---|---|---|
| Residency | EU-tagged tenant; all data plane in EU voter set; policy-as-code; EU HSM root; EU-only edge prefixes | Re-replicate only inside the EU (eu-par spare) | EU↔world cut: EU fully served, non-EU collaborators lose EU docs | Fail closed, never cross-border | No relays, CDN, index, backup, logs, or edge outside EU. US users connect straight to EU gateways | 3-region EU footprint + spare ($1.2k/mo standing); extra directory hop | Non-EU latency; global cross-jurisdiction search/AI |
| EU↔US collaboration | One home jurisdiction per doc (no cross-jurisdiction multi-master) | US editors reconnect to surviving EU regions (+~95 ms RTT) | US editors cut off; local edits queue, subject to device policy | Viewer mode, editor cap | Remote sessions into the home jurisdiction | ≈$0.8k/mo cross-region allowance (39 TB) | Non-EU p99 ≈ 240 ms (US-east) to 530 ms (Sydney): meets 800, not 200. Offline cache of EU docs on non-EU devices if the tenant sets strict policy |
| EU survival of a region loss | 3 EU voters + spare; EU sized per §5 | 2 voters keep quorum. Cold: primary, buddy, and WORM copies are in 3 distinct EU regions, so ≥2 survive | If the 2 survivors partition, EU writes halt (CP) | Durability is RF2 until rebuild: hot ≈20 min, cold ≈6 h at 40 Gbps | 3 voters + 4th spare region | EU compute at 819 vCPU | A 2nd EU failure while degraded halts EU writes |

**C. E2EE × search × admin recovery (× server-side validation)**

| Property | Preconditions | Single-region failure | Partition | Degradation policy | Decision | Cost | Sacrificed |
|---|---|---|---|---|---|---|---|
| Search | Server sees ciphertext only | Client index unaffected. S2 index rebuilt in survivors | Local search works offline | S2 off → local only | S1 default: client-side encrypted index. S2 opt-in: SSE with keyed trigram tokens, padded buckets | Client storage ≈35% of synced text; saves the server index | Server-side ranking, snippets, discovery of unsynced docs. S2 leaks token equality, frequency, and search/access patterns |
| Admin recovery | RKEK 3-of-5 or tenant HYOK; shares never at provider | Unaffected (keys outside the region data plane) | Needs admin quorum online | <3 shares → data unrecoverable by design | Tenant-held recovery, transparency log | Admin burden; ~$0 infra | "Only members can read" (admins can). Provider cannot recover lost keys |
| Server merge / validation / compaction | Payload unreadable | Same as plaintext (log replication is payload-agnostic) | Same | Clients produce signed snapshots; DSO sequences only | Client-side compaction and poison defense | +40% retained storage on E2EE docs (no server folding) ≈ +$1.2k/mo; client CPU | Shadow-apply, malware scan, previews, AI. Longer poison-pill time-to-mitigate |

**What cannot be met**
1. Loss of the entire EU jurisdiction: EU tenants go unavailable, with no US failover.
2. Two simultaneous regions lost in one voter set: writes halt.
3. 200 ms in-region p99 for LATAM/MEA during failover, and for non-EU users on EU docs.
4. 99.99% if non-region causes consume more than ~240 s per 30 days.
5. E2EE together with server-side search, AI, and validation.
6. Absolute "never leaves EU": delivery to non-EU devices and their offline caches is unavoidable.
7. The 12-month client window during an emergency security fix. We force-expire the vulnerable version instead.
8. "Visible = durable" at the same time as <200 ms.
9. Provider-wide durable-storage loss (no second-provider copy; see §0 for its price).

## 4. Failure analysis

### 4A. Full regional outage (eu-dub lost; compute and storage gone)

**Fencing.** The epoch is the Raft term, enforced at four layers.
1. Raft quorum.
2. Conditional object writes: segment names include the term, and manifests change by CAS on (term, version).
3. Gateways reject ops carrying a stale `leaderEpoch`.
4. A 2 s leader lease with an election timeout ≥3 s, so there is no dual-leader window under bounded clock drift. A zombie (partitioned but alive) region self-fences to read-only within 2 s, and the directory is only a cache.

**Timeline (EU)**

| T+ | Event |
|---|---|
| 0 | Failure |
| 2–3 s | Clients miss 2–3 1 Hz heartbeats |
| 3–5 s | ~1,365 EU groups that lost their leader elect new ones with fra/sto (heartbeat 0.5 s, timeout 3–4 s). 2 of 3 is a normal quorum, so no unsafe recovery is needed. Holders re-submit tentative ops with receipts |
| ~10 s | Detector (3 vantage points × 3 misses) marks the region dead, withdraws BGP/DNS, bumps the directory epoch |
| 5–60 s | Clients reconnect with jitter U(2,58 s) using staggered happy-eyeballs across the 3 EU endpoints (second attempt after 250 ms, losers cancelled, so handshakes are not tripled) |
| ≤60 s | All groups serving (server RTO) |
| ≤105 s | Last client submits, including one admission-control retry. Hard budget 120 s |

Followers hold applied state, so DSO activation is lazy per doc (50 ms–1 s). Cold docs load from the surviving buddy region's object store.

**EU-internal survivors**
- Hot data: fra and sto each hold a full copy.
- Cold data: primary, buddy, and WORM copies sit in 3 distinct EU regions.
- Rebuild: durability returns to RF3 via eu-par, with a standing minimum of 40 vCPU plus reserved burst. Hot data takes ≈20 min (5.8 TB at 5 GB/s). Cold data takes ≈6 h (100 TB).

**Reconnect surge (consistent units: overhead-adjusted vCPU)**
- eu-dub carries 8% × 5M = 400k sessions, so 6.9k joins/s over 58 s.
- That costs 6.9k × 20 ms × 1.5 = 207 vCPU, on top of 311 in use (all EU load now on survivors), against 549 provisioned in fra + sto: **94%** unshed.
- Shedding presence and search frees 111 vCPU, giving **74%**.
- The worst EU case, losing fra, gives exactly 80% after shedding by construction (§5).

**Request-weighted availability.** Weighted unavailable seconds = traffic share × recovery time. This is a conservative proxy, since backing-off clients send fewer failing requests. The monthly budget is 0.01% × 2,592,000 s = 259.2 s. Expected recovery ≈40 s = 3 s detect + 30 s mean jitter + ~4 s join + retries.

| Lost region | Share | Worst case (120 s) | Expected (≈40 s) |
|---|---|---|---|
| us-east (largest) | 15% | 18 s (6.9% of budget) | 6.0 s |
| eu-dub | 8% | 9.6 s | 3.2 s |
| sao | 6% | 7.2 s | 2.4 s |

Surviving-region editors on docs led from the failed region add ~0.3 s weighted (≤5 s stall). EU docs with US editors add about 1 s. One such event leaves ≥93% of the budget for deploys and other faults.

### 4B. Poison-pill edit (crashes every receiving client)

**Prevention**
- The DSO shadow-applies each op on a pinned set of client cores (WASM): oldest, newest, and any intermediate version where fuzzing found divergence. An op that throws on any of them is rejected or capability-gated. This costs ~30 µs per core, inside the 150 µs DSO budget.
- Shadow state adds ~3 in-memory replicas of active docs (≈300 GB fleet-wide, against ≈14 TB of RAM bundled at 4 GB/vCPU).
- Structural limits: depth, 64 KB op size, rate.
- Per-doc feature gating by capability, so new op types never reach old clients.
- CI runs coverage-guided fuzzing of decode and apply on every supported client version.
- Residual risk: bugs in unpinned versions and E2EE docs (no shadow-apply) depend on detection below.

**Detection and containment (client)**
- Remote ops are applied in an **isolated worker or process**. A poison crash kills the worker, not the app, so restart takes ≈1 s without user action.
- Before applying a remote batch, the client durably writes `applying:[opIDs]`.
- If the worker restarts with the marker set, it counts a crash. After 2 crashes it enters **safe mode**: read-only, bisecting the batch in a sandbox to find the culprit.
- It reports `suspect(opID, clientVer, stackHash)`. This is envelope-only data, so it also works under E2EE.
- Restart backoff bounds the crash loop to 2 iterations.

**Mitigation (server)**
- When reports arrive from ≥3 distinct devices (≥2 users) or ≥10% of sessions in 60 s, the DSO appends `Quarantine{opIDs}` to the log.
- Docs with ≤3 sessions need the owner or a doc admin.
- This is a grow-only skip-set (a CRDT), so all replicas converge. Causal descendants re-anchor deterministically.
- The DSO writes a repaired snapshot (gen+1) within 30 s.
- Sync sends quarantine records *before* ops, so late and offline clients never apply the pill. For clients that predate the Quarantine op, non-E2EE docs filter quarantined ops server-side, and a client-version fence covers the rest.
- A **class-wide kill switch** (signed denylist of op signatures, pushed on heartbeat in ≤30 s) enforces at gateways and the DSO. It also covers a pill pasted into other docs.

| T+ | Event |
|---|---|
| 0 | Pill committed |
| 0.5 s | Apply workers crash in every receiving client; the UI survives |
| ~2 s | Worker restarted, second crash |
| ~3 s | Safe mode, bisecting |
| ~6 s | `suspect` reports sent |
| ~10 s | Quorum, Quarantine committed |
| ~12 s | Clients resume |

The target is time-to-mitigate ≤60 s p99 and ≤120 s worst case. The blast radius is one doc (up to 5,000 sessions) unless the pill is copied elsewhere, which the class-wide denylist covers. Under E2EE, quarantine relies on client reports only, so mitigation is slower. Collusion to censor an op is bounded by the ≥3-device, ≥2-user rule and admin override.

## 5. Capacity and cost

### Reference arithmetic

**Load.** Peak sessions are 5M (NA 1.75M, EU 1.25M, APAC 1.5M, LATAM 0.3M, MEA 0.2M). Active editors are 1.0M, so 1.0M msg/s × 300 B = 300 MB/s ingress. With 2 receivers each, fanout is 2M msg/s = 600 MB/s. Presence: 2.5M sessions × 0.5 Hz × 60 B = 75 MB/s in, 150 MB/s out. Joins: 5M / 1,200 s = 4.2k/s, averaging 18.5 KB, so 77 MB/s. **Peak egress = 600 + 150 + 77 = 827 MB/s; average = 413.5 MB/s.**

**Routing assumption.** A cross-region doc has 1 of 3 collaborators remote, so every op crosses a region boundary exactly once. Cross-region share = 10% of ops, or 100k msg/s, 30 MB/s peak, 15 MB/s average, ≈39 TB/month. Client traffic to home-jurisdiction gateways is internet egress, already counted below. The 39 TB is kept at the inter-region rate as a conservative allowance for gateway→DSO hops and unpinned-doc relay hops.

**Log growth (scenario inconsistency).** Durable ops average 105 MB/s ≈ 9.1 TB/day raw. The 30-day full-fidelity tail alone is ≈272 TB raw (27% of the 1 PB logical total). Folding older history ~10× gives net growth ≈27 TB/month (2.7%). If it is not offset by expiry, that adds ≈$0.8k/mo to the bill each month. The model uses 1 PB steady state.

**Raw peak vCPU**

| Component | Derivation | vCPU |
|---|---|---|
| Gateways and relays | Connections 5M × 20 µs = 100; edit in 1M × 30 µs = 30; edit out 2M × 30 µs = 60; presence 3.75M × 25 µs = 94 | 284 |
| DSO | Edits 1M × 150 µs = 150; presence 1.25M × 10 µs = 13 | 163 |
| Raft follower apply | 2 × 1M × 40 µs | 80 |
| Join / ACL / API | 4.2k/s × 20 ms | 84 |
| Snapshot / compaction | 800k docs ÷ 300 s = 2.7k snaps/s × 8 ms = 21; fold/compaction 6 | 27 |
| Search index + query | Re-index 800k ÷ 30 s = 26.7k/s × 5 ms = 133; queries 5.6k/s × 10 ms = 56 | 190 |
| **Total** | | **828** |

With 1.5× platform overhead, load at 100% utilization is **1,242 vCPU**. Of that, sheddable presence and search is 297 raw, or 445 adjusted.

**Sizing rule.** Steady utilization ≤40% in every region. The loss of the largest member of any voter set must leave survivors ≤80% *including* the reconnect surge, after shedding presence and search. The surge for a lost region with share *s* is 5M ÷ 58 s × 20 ms × 1.5 = 2,586 × *s* vCPU.

| Set | In-use load | Worst loss | Surge | Shed | Survivor need at ≤80% | Provisioned | Post-failure util |
|---|---|---|---|---|---|---|---|
| NA | 435 | use (15%) | 388 | 156 | 833 | use 467, usc 417, usw 417 = **1,301** | 80% |
| EU | 311 | fra (9%) | 233 | 111 | 540 | fra 279, dub 270, sto 270 = **819** | 80% (dub loss: 74%) |
| APAC | 373 | sin (12%) | 310 | 134 | 686 | sin 373, tyo 311, bom 375 = **1,059** | 80% |
| LATAM | 75 | sao (6%) | 155 | 27 | Absorbed: use+usc carry 311 + 203 = 514 of 884 (58%) | sao **186** | n/a |
| MEA | 50 | uae (4%) | 103 | 18 | Absorbed by bom/fra headroom (443 spare vs 135 needed) | uae **124** | n/a |
| **Total** | | | | | | **3,489 vCPU** | |

Unshed, us-east loss would run survivors at 99% (823/834), so shedding is mandatory, not optional. The previous uniform 40% sizing (3,000 vCPU) would have left NA at ~130%.

### Monthly cost (720 h; replication factors, tiers, indexes, and backups modeled separately)

| Line | Derivation | $/month |
|---|---|---|
| Compute | 3,489 × $0.04 × 720 | 100,500 |
| Internet egress | 413.5 MB/s × 2.592M s = 1,072 TB × $80/TB | 85,700 |
| Inter-region | Raft replication, uncompressed, 105 MB/s × 2 followers = 544 TB + segment and cold-snapshot copies to buddy and backup domain (109 TB log + 22 TB snapshots) × 2 = 262 TB + cross-region allowance 39 TB = 845 TB × $20/TB | 16,900 |
| Hot SSD | 160M docs × 145 KB (125 KB snapshot + 20 KB WAL tail/metadata) = 23.2 TB; RF3 = 69.6 TB; ÷0.7 fill = 99.4 TB × $100/TB | 9,900 |
| Search index | 35% of 225 TB non-E2EE state = 78.75 TB. 25% on SSD at RF2 (56 TB provisioned) + 75% in object at 2 copies | 8,300 |
| Metadata / ACL / directory | 2 TB RF3 ÷0.7 | 900 |
| Object (primary + buddy) | 1 PB ÷ 2.5 = 400 TB × 2 × $23/TB | 18,400 |
| Backup (WORM, separate domain) | 400 TB × 1.15 retention × $23/TB | 10,600 |
| E2EE overhead | 40 TB compressed E2EE data × 40% = 16 TB × (2 × $23 + 1.15 × $23) | 1,200 |
| EU spare standing (eu-par) | 40 vCPU × $0.04 × 720 | 1,200 |
| Object requests (not in the price list; $5/M PUT assumed) | 16.4k groups × 2,880 flushes/month × 3 copies ≈ 142M PUTs | 700 |
| **Total** | | **≈254,300** |

**Normalized: $254.3k / 50M DAU = $0.0051 per DAU per month.** Cost drivers: compute 39%, egress 34%, storage 19%, inter-region 7%, other 1%.

### Latency budget (p99, ms; sum of per-hop p99s, which is conservative)

| Hop | In-region | Cross-region (EU doc, US-east receiver) |
|---|---|---|
| Client → gateway | 25 | 25 |
| Gateway auth/rate (cached) | 2 | 2 |
| Gateway → DSO | 2 | 2 |
| DSO validate + sequence | 8 | 8 |
| Relay coalescing tick | 20 | 20 |
| Relay → client (25 ms one-way access = 50 ms RTT, +2 jitter) | 27 | ~70 |
| Receiver decode + apply | 15 | 15 |
| Sum | 99 | 142 |
| + one loss-recovery round | +50 → ~150 (50 ms margin) | +~100 → ~240 |

Worst pair (Sydney receiver on an EU doc): one-way ≈175 ms, so about 247 ms before tails and ≈530 ms with one 280 ms recovery. That is under 800 ms with ~270 ms margin.

### Missing workload distributions (assumed and modeled)

| Missing distribution | Treatment |
|---|---|
| Doc popularity / collaborators per doc | Assume Zipf. Mean 3, p99.9 ≈ 200, tail to 5,000. Cap 100 editor slots plus viewer mode |
| Maximum size | 25 MB state, 64 KB op, paste chunked |
| Log growth | Reconciled above (fold + 35-day tombstone) |
| Reconnect bursts | Failover, release, mobile-carrier outage, Monday 9 am local. Jitter U(2,58 s), staggered connects, admission control, shedding (sizing rule above) |
| Offline size and duration | 0.1% of DAU = 50k users. Assume 20 MB upload and 5 MB download each: ~1 TB and 250 GB, negligible. Daily reconnects: see §0 |
| E2EE share | 10% |

### Heavy-tail scenario 1: 5,000-session doc

Naively, 20% active gives 1,000 typists × 300 B = 300 KB/s per receiver × 5,000 = **1.5 GB/s**. Presence is 2,500 senders × 0.5 Hz × 60 B = 75 KB/s, fanned out to 5,000 receivers = 375 MB/s. Total is 1.875 GB/s, about **$540/hour** for one doc. The edit fanout alone is 2.5× the platform's entire 600 MB/s peak.

With policy (100 editor slots, 4,900 viewers):
- Editors: 100 ops/s × 300 B × 100 receivers = 3 MB/s.
- Viewers get viewport-scoped ops (≈5%) coalesced at 250 ms: ≈1.5 KB/s each = 7.35 MB/s.
- Presence is capped at 50 cursors per viewer: 7.35 MB/s, plus ≈0.3 MB/s among editors.
- Total ≈ 18 MB/s (144 Mbps) ≈ **$5.2/hour**, about 104× cheaper.
- A relay tree of 50 relays × 100 clients keeps DSO egress ≈1.5 MB/s.
- Editor p99 stays <200 ms. Viewer p99 ≈350 ms, a documented exception (<500 ms).
- A 5,000-client meeting-start join burst at 500 joins/s = 9 MB/s, served from the in-region snapshot cache.

### Heavy-tail scenario 2: reconnect storm

The eu-dub storm is in §4A (94% unshed, 74% shed). The us-east storm is 750k sessions, 12.9k joins/s, and 388 vCPU on usc + usw (834 vCPU): 99% unshed and 80% shed, which is why the sizing table drives provisioning.

### Budget ceiling

**$0.0080 per DAU per month ($400k/month), 1.57× the model.** The defended contingencies are:
- +20% for unmodeled platform services (observability, KMS/HSM, WAF/DDoS, CI/staging).
- +10% for drills and heavy-tail surges.
- +25% for price and estimate error (compute and egress dominate).

These sum to +55% ($394k), rounded up to $400k. At an assumed ≥3% paid conversion at $10 per month (≈$0.30 per DAU-month), the ceiling is ≈2.7% of revenue. If we breach it, the levers are:
1. Accept a 90% post-failure peak instead of 80%, saving ≈$6k and only if the full-scale reconnect drills pass.
2. Trim egress 10% with tighter coalescing and viewer sampling, saving ≈$8.6k.
3. Dictionary-compress the Raft stream at 1.5:1, saving ≈$3.6k.
4. Add private interconnect for egress.

## 6. Cuts and roadmap

**Cuts for v1**
- No SSE (S2) search for E2EE.
- No relays for unpinned docs.
- Cap of 100 editors and 1,000 viewers.
- No sao/uae regions until P5. LATAM and MEA are served remotely from use and sin in the meantime.
- No BYOK/HYOK.
- Offline 7 days, 6-month client compatibility until Phase 2.
- Canvas limited to frames, shapes, and LWW properties.

| Phase | Scope | Measurable exit criterion |
|---|---|---|
| P0 (0–3 mo) | NA only: text CRDT, DSO, 3-region Raft, snapshots, receipts | p99 in-region ≤200 ms at 100k concurrent in load test; 0 lost acked ops across 50 region-kill runs (95% bound <6% per kill) plus 10⁶ simulated runs; RTO max ≤120 s over 20 drills; us-east-scale reconnect drill: 100% of probers ≤120 s, survivor CPU ≤85% |
| P1 (3–6) | EU jurisdiction, residency enforcement, EU-only edge, EU/US collaboration | EU region-failure RTO max ≤120 s; 0 EU honeytoken hits in non-EU storage/logs for 30 days; EU rebuild hot ≤20 min |
| P2 (6–9) | 30-day offline, 12-month compatibility CI, poison-pill quarantine | 10⁶ randomized 30-day-offline merges converge bit-identically; 12-version client matrix green; poison game-day time-to-mitigate ≤120 s |
| P3 (9–12) | APAC; LATAM/MEA served remotely; editor slots and viewer mode; relays for unpinned docs | Cross-region p99 ≤800 ms for 30 days of probes; largest-member reconnect drills per voter set (NA 750k, APAC 600k, EU 450k sessions): 100% of probers submit ≤120 s, CPU ≤85% shed; ~100×-class heavy-doc cost reduction reproduced |
| P4 (12–15) | E2EE with client-side search and admin recovery | Third-party crypto audit with 0 critical findings; recovery drill: 3-of-5 recovers 100% of test DEKs; no E2EE plaintext markers server-side |
| P5 (15–18) | sao/uae regions and voter-set migration, availability and cost hardening, optional S2 search | 99.99% SLI over 90 days including ≥1 region drill; sao/uae failover drills pass RTO; measured $/DAU ≤$0.0080 extrapolated to 50M |

## 7. Verification plan

| Area | Method | Pass criterion |
|---|---|---|
| Commit/fencing protocol | TLA+ model (including receipt re-submission and term-change reconcile) plus FoundationDB-style deterministic simulation (10⁶ runs with random region kills, partitions, clock skew) | 0 lost acked ops; 0 dual-leader commits; 0 orphaned dependents |
| CRDT convergence | Property-based tests and nightly fuzz (10⁹ causally-consistent interleavings, including virtual-clock 30-day offline) | 0 divergent state hashes |
| RPO under region kill | Jepsen-style in test cells with production topology: kill a region mid-load at 100k ops/s; audit acked opIDs against recovered log | 0 missing acked ops in ≥1,000 kills (rule of three: <0.3% per-kill loss at 95% confidence) |
| RTO | Monthly production region-evacuation drill; ≥5,000 distributed prober clients | 100% of probers submit ≤120 s (hard max). Real-client p99.9 is reported as a monitor |
| Capacity | Largest-member reconnect drill per voter set at full session counts, with shedding | Survivor CPU ≤85% after shedding; 100% of probers ≤120 s |
| Latency | Probers ≥1,000 per region pair with netem-emulated 50 ms RTT access, PTP/GPS-synced stamps; load test at 1.5× peak | p99 <200 ms in-region and <800 ms cross-region on rolling 24 h |
| Residency | OPA policy-as-code on IaC; daily honeytoken docs searched in non-EU storage, logs, backups, heap dumps, and non-EU edge captures; egress allow-list audit; quarterly external audit | 0 hits; 0 policy violations |
| E2EE | Marker strings in E2EE docs; crypto audit; recovery drill | 0 plaintext hits server-side; 0 critical findings; 100% recovery |
| Compatibility | 12 monthly client builds × server N and N+1 on golden op logs; rolling-deploy test | Round-trips byte-exact including unknown fields; deploy error rate ≤0.001% |
| Poison pill | Injected synthetic pills (including in an unpinned client version and an E2EE doc); decoder/apply fuzz ≥10⁹ execs/day | Time-to-mitigate ≤120 s; ≤2 crash-loop iterations; 0 pills reach snapshots |
| Heavy tail | Replay of 5,000-session doc and largest-region reconnect storm | Editor p99 <200 ms, viewer <500 ms, egress ≤160 Mbps; CPU ≤85% with presence and search shed |
| Availability | Prober plus server-side request SLI (success per §0); monthly chaos (zone, region, partition), drills counted | ≥99.99% over rolling 30 days, 3 consecutive months |
| Cost | Quarterly bill vs model | Unit cost within ±15% of model and ≤ceiling | 


``` 


--- 


# Response Developer Validation Layer 


``` 

# Global Collaborative Editor: Design v4

## 0. Fixed assumptions and definitions

**Failure domains** (the last column lists what each domain does *not* isolate)

| Domain | Definition | Independent of | Shares fate with |
|---|---|---|---|
| Zone | One AZ (≥3 per cloud region) | Power, cooling, rack network of other zones | Regional control plane |
| Region | One cloud region (metro) | Compute, storage, power, network of other regions | Provider IAM and DNS, our binaries and config (hence cell-based staged rollout), global directory, anycast control |
| Jurisdiction | Legal/admin domain with its own cloud accounts, KMS/HSM root, deploy pipeline, break-glass | Other jurisdictions' accounts, keys, operators | The cloud provider, public DNS/BGP, the signed-build trust root, the global directory (routing metadata only) |
| Backup domain | Separate account, object-lock, separate KMS key, in a region other than primary and buddy | Region or zone loss, credential compromise, operator error | For EU content, the EU jurisdiction's fate (backups must stay in the EU). In v1, also the provider's |

**Topology.** There are 5 product regions on 11 active cloud regions, plus one pilot-light spare (12 deployed locations). Traffic shares are in parentheses.
- NA: us-east (15), us-central (10), us-west (10)
- EU: eu-fra (9), eu-dub (8), eu-sto (8). eu-par (France) is a pilot-light spare and not a voter.
- APAC: sin (12), tyo (10), bom (8)
- LATAM: sao (6)
- MEA: uae (4)

**Voter sets** (end state, P3; 3 Raft voters per shard group, each in a distinct region):

| Set | Regions |
|---|---|
| NA | {use, usc, usw} |
| EU | {fra, dub, sto} |
| APAC | {sin, tyo, bom} |
| LATAM | {sao, use, usc} |
| MEA | {uae, bom, fra} |

- In v1, LATAM docs home in the NA set and MEA docs in the APAC set (see §6).
- A physical region can sit in two sets (use and usc in NA and LATAM, bom in APAC and MEA, fra in EU and MEA). Losing it removes one voter from each set, but only groups *led* from it re-elect. Availability is therefore counted by serving region (§4A), not by voter-set membership.
- Non-EU content may sit in EU regions. The reverse is forbidden.
- Leader placement is optimized only among the eligible regions of the doc's voter set. An EU-pinned doc whose median editor is in the US still leads in fra, dub, or sto.

**EU "region" and boundary.** An EU region is one cloud region in an EU member state (fra DE, dub IE, sto SE, spare par FR). The EU boundary is the territory of the 27 member states plus an EU-established operating entity with an EU-resident HSM root. EEA-only states are not treated as EU. Every acked EU op is on ≥2 of 3 EU voters, so any single EU region loss leaves 2 voters (a quorum) and at least one copy of every acked op, all inside the EU.

**Out of scope (stated as unmet):**
- Loss of two regions in one voter set.
- Loss of the whole EU jurisdiction.
- Provider-wide failure. This is unmitigated in the base design. An optional second-provider, EU-resident WORM copy is priced in §5 (≈$21k/month, restore RTO in hours, outside the SLO).

**Residency interpretation, and the literal conflict.** Read literally ("no EU content byte exists on any device outside the EU"), the requirement is incompatible with EU and US users collaborating on one document, because a US collaborator who can read the doc holds its content on a non-EU device. Even E2EE does not help, since the plaintext is rendered there. We therefore **mark the literal reading unmet at the end-user-device boundary** and satisfy this operational reading: provider infrastructure outside the EU never stores, processes, or caches EU content. That covers ciphertext, indexes, backups, logs, crash dumps, and CDN. EU-pinned traffic enters only through EU-announced anycast prefixes and IP-literal EU endpoints, so none of our non-EU PoPs carry it. Public-internet transit of TLS-protected bytes to a non-EU user is unavoidable and acknowledged. Tenants can opt into stricter policy at the cost shown in §3B:
- EU-device-only: no US collaboration.
- No-offline-cache: non-EU devices hold content in memory only.

**Content and metadata classification**

| Item | Class | Form and location | Justification and residual leak |
|---|---|---|---|
| Document body, ops, snapshots, comments, titles, thumbnails | Content | Plaintext (std) or ciphertext (E2EE); home jurisdiction only | |
| Search indexes | Content | Std: plaintext inverted index. S1: client-side only. S2: keyed-token blocks, treated as content | S2 leaks token equality, frequency, and access patterns |
| Segments, backups, manifests | Content | Same form as source; EU copies EU-only; manifests hold docID and hash only | |
| Doc ID (random 128-bit, no embedded time or tenant) | Global routing metadata | Global directory: docID→(jurisdiction, shard group, epoch), ≈32 B/entry, 64 GB total | Random IDs are still linkable, so the directory holds no tenant, user, or time fields, is access-logged, and has no join key to user IDs outside the jurisdiction. Residual leak: a docID exists and has a home jurisdiction |
| Tenant ID→jurisdiction | Global routing metadata | Global | Random; needed for login routing |
| User ID | Personal data | Home jurisdiction. Docs hold pairwise pseudonymous subject IDs, and the mapping stays in the user's home jurisdiction | GDPR personal data |
| User→docs reverse index | Restricted | User's home jurisdiction; docID and home jurisdiction only, no titles | Needed for "shared with me"; titles are fetched from the doc's jurisdiction on open |
| ACL, sharing graph, membership | Restricted (content-adjacent) | Doc's home jurisdiction only | Reveals who collaborates with whom |
| Timestamps, sizes | Restricted | In-jurisdiction. Exported globally only as aggregate buckets with k≥1,000 docs and no tenant breakdown | Residual: bucketed volume per jurisdiction |
| Op envelope | Restricted | In-jurisdiction. Std: actor, counter, lamport, size, type. E2EE: only docID, device pseudonym, counter (dedupe), size padded to 256 B, keyEpoch, schemaVer, op class (content/ACL/key/snapshot) | E2EE lamport, deps, and semantic type are inside the ciphertext. Residual leak: typing cadence and size buckets (accepted, documented) |
| Telemetry | Hashed IDs, jurisdiction-local salt | No raw IDs leave | |

**Added workload assumptions**
- Diurnal average = 50% of peak.
- 10% of docs are E2EE: incompressible ciphertext, +5% envelope overhead.
- 8% of docs are hot (touched in the last 30 days) = 160M docs.
- Current-state snapshot = 25% of logical size (125 KB). Hot SSD holds it uncompressed for fast DSO activation.
- Compression is 2.5:1 on non-E2EE stored data. Stored fraction = 0.9/2.5 + 0.1×1.05 = **0.465**, so 1 PB logical stores as 465 TB.
- Durable op bytes = 70% of wire bytes (210 B).
- Presence: 0.5 Hz × 60 B from 50% of sessions.
- Join: 70% delta-sync (5 KB, 8 ms CPU), 30% full (50 KB, 40 ms CPU).
- Mean session is 20 min.
- Offline: 5% of DAU per day = 2.5M reconnect syncs/day, averaging 100 KB up and 1 MB down. The 0.1% long-offline cohort is 50k users, mean episode 14 days, 20 MB up and 5 MB down per reconnect.
- History folds ≈10× after 30 days.
- Memory is priced inside the vCPU-hour at 4 GB/vCPU.

## 1. Architecture and data model

**Tiers.** Edge (anycast, L4 pass-through, TLS never terminates outside the doc's jurisdiction) → Gateway (stateless; QUIC/WebSocket, auth, rate limits) → Relay (fan-out, viewport filtering) → **Document Session Owner (DSO)**, one per active doc, co-located with the shard-group Raft leader under its lease → **Raft log** → segment and snapshot store (object storage) → search indexers. A control plane holds the directory, config, and key management.
- For plaintext docs the DSO holds CRDT state.
- For E2EE docs it holds only sequencing state: head index, dedupe window, key epochs, and ACL position.

**Sharding.** docID → 2¹⁶ vshards → ~16k Raft groups (EU ≈ 4,096, ≈122k docs per group), with online split and merge. A group is the unit of failover. A *cell* (~2% of tenants) is the unit of rollout.

**Write path**
1. The client sends an idempotent batch keyed by opID = (actorID, counter).
2. The gateway checks auth, size, and rate.
3. The DSO validates:
   - *Plaintext docs:* decode, schema, and limits, then apply on a native reference core over a copy-on-write overlay. Ops flagged high-risk (new op types, unusual depth or shape, ≈1% of ops) are also shadow-applied in sandboxed WASM cores for **all 13 supported client builds** (N-12…N), using a bounded context window around the op's anchors.
   - *E2EE docs:* envelope validation only (device signature checked against the ACL, size, rate, dedupe, key epoch).
4. The DSO appends to Raft. The entry's `(term, index)` is its `seq`, not a separate counter. In parallel it fans out *tentatively* via relays, tagged with `(term, index)`. The append is a local NVMe fsync plus two remote followers.
5. At 2-of-3 durable, the commit watermark advances and `ACK(opIDs)` goes to the sender. The watermark piggybacks to everyone.

**Tentative-state protocol.** Receivers hold committed state Sc (ops ≤ watermark) and a journal J of tentative ops keyed by opID. The view is Sc ⊕ J.
- On a term change the new leader publishes `TermStart(term, firstIndex)`. Receivers drop journal entries from older terms at or beyond `firstIndex` unless the same opID reappears, then recompute the view from Sc ⊕ J′.
- Ops that depend on a dropped op wait up to 10 s for their live sender to re-upload it (idempotent by opID), then are dropped. The author's client re-anchors its own dependents to the retracted op's recorded origin.
- A retraction occurs only if the sender and every holder of an uncommitted op die inside the commit window (46–160 ms). It appears to receivers as an ordinary remote deletion.
- Clients key their outbox on opID, not `seq`.

**Data model**

| Object | Key → fields | Storage |
|---|---|---|
| Document | docID → tenantID, homeJur, shardGroup, schemaVer, textAlgoGen, keyMode (std/e2ee), aclVer, headIndex, snapshotIdx/gen, size, created/updated | Metadata DB, Raft RF3 |
| Op log entry | (docID, term, index) → opID, lamport, opType, schemaVer, keyEpoch, deps, payload (plain or ciphertext, stored verbatim and never re-serialized by the server) | Raft WAL on NVMe, mirrored across 2 zones, flushed as 64–256 KB zstd segments `{docID}/{startIdx}-{term}.seg` to object storage in primary and buddy regions |
| Snapshot | (docID, gen, snapIdx) → state blob, skip-set hash, tombstone skeleton, stateHash, keyEpoch, trust flag | Hot: SSD on voters (uncompressed). Cold: object. Cut every 1,000 ops, 5 min active, or 256 KB tail |
| ACL, membership, restore, comments | Log entries of the same doc (comments are log entries with an anchor and a pseudonymous author; ciphertext under E2EE) | Same log, so revocation has an exact position in the total order |
| Search index | Per-doc index segments; std: plaintext; S2: encrypted token blocks | Same replication and backup path as segments |
| Presence | (docID, session) → pseudonymous user, cursor/selection, viewport, TTL 10 s | DSO memory only, never persisted, encrypted under E2EE |
| Directory | docID→(jur, group, epoch); group→leader (cache, validated by Raft term) | Global, replicated to every region, minimal fields |

**History policy.**
- Full-fidelity ops are kept for 30 days. Older history folds into named and daily checkpoints.
- Tombstone *content* is dropped at 35 days. The ID-run *skeleton* (ranges only, no content, ≈2 B per run, ≤1% of doc size) is kept until the document is purged, so late anchors still resolve deterministically.
- The skeleton carries only pseudonymous actor IDs. Erasure of a user's pseudonym mapping unlinks them.

**Zone analysis.**
- A voter's WAL is a synchronous 2-zone pair (fsync on both before the voter counts as durable, +1 ms), so a single node or zone loss degrades a voter without losing it.
- Snapshot SSD is single-zone and rebuildable from object storage.
- Gateways, relays, and DSOs are stateless or lazily reactivated. Zone loss leaves 2/3 of a region's capacity, so utilization goes from 40% to 60%, the same N+1 rule as for regions.

**Session routing.**
- The client asks the directory for the doc's jurisdiction (cached ≤60 s) and connects to that jurisdiction's gateway. For EU-pinned docs, US clients cross the Atlantic directly, with no non-EU relays. Gateways resolve group→leader from cache, and a stale cache is corrected by an epoch-mismatch redirect (one RTT).
- Clients carry a signed config with IP-literal endpoints for all three voter-set regions, so failover has no DNS-TTL dependence.
- In v1, all remote clients connect straight to the home gateway. For unpinned (non-EU) tenants from P3, a client attaches to its nearest local relay, which forwards over the backbone to the home DSO's gateway.

**Protocol and schema evolution (12-month clients, zero downtime)**
- The supported window is N-12…N (13 monthly builds). The wire format is protobuf with append-only fields. Unknown fields and op types are preserved as opaque bytes.
- The log is immutable. Read-time upcasters (N-12…N) convert ops, so history is never rewritten.
- New semantics ship expand → dual-write → contract. Contract waits ≥13 months after release.
- Old clients keep working without downtime or crashes on every existing feature. New features reach them only as an additive embed with a fallback representation, or are gated per tenant by min client version. Old clients preserve what they cannot interpret but cannot use it.
- A text-CRDT algorithm change (e.g. away from Fugue) is a doc *generation* change, `GenerationChange(index, algo, checkpointHash)`, applied at the same log index by all replicas. It happens only after the tenant's min client version passes the new algorithm's release by ≥13 months. Until then docs stay on the old algorithm.
- Raft log-format bumps use a cluster-wide feature barrier after all replicas upgrade. Rolling deploys run N and N-1 servers side by side.
- Byte-exactness holds at the log and segment level, because the server never re-serializes. At the client it is semantic preservation of unknown fields.

**E2EE**
- **Keys:** a per-doc DEK (XChaCha20-Poly1305) with epochs, HPKE-wrapped per member device. Each new epoch is also wrapped to the tenant recovery public key RK_pub. MLS is used above ~20 members, since TreeKEM is O(log n), which matters for 5,000-member docs. Key-epoch records form a hash chain in the log, so backups restore the wraps.
- **Server role:** it sequences and replicates envelopes only. Clients produce snapshots and compact. Clients compute and sign `stateHash` at checkpoints, and a snapshot becomes trusted when ≥2 distinct devices sign the same hash.
- **Removal:** triggers a new key epoch. Removed members keep what they already had.
- **Admin recovery (tenant-admin, not provider-assisted):**
  - The RK private key is Shamir-split 3-of-5 across admin hardware keys, or the DEK wraps use a tenant HYOK KMS in the tenant's jurisdiction. The provider holds zero shares.
  - Recovery: ≥3 admins sign a request, which is logged to a key-transparency log visible to members, with a tenant-configurable cooling period (default 24 h). The admins reconstruct RK, unwrap the DEK epochs, and re-wrap them for new members.
  - RK rotation re-wraps DEK epochs in the background from an admin client.
- **Offline key-epoch rule:** an op authored offline carries the epoch it was encrypted under. It is accepted iff the author device was a member at that epoch and has not been removed by upload time. Otherwise it goes to the owner-review inbox. Accepted old-epoch ops stay decryptable, and members receive wraps for past epochs.
- **Presence and routing:** relays cannot read E2EE cursor or viewport data. E2EE docs therefore get no server-side viewport scoping, and fan-out is full, so they are capped at 100 editors and 1,000 viewers. An opt-in 6-bit coarse region bucket in the envelope restores ≈16× reduction at the cost of leaking coarse scroll position (default off).
- **Search:** S1 and S2 are specified in §3C. Search over offline-edited docs works locally via S1. S2 does not cover offline edits until sync.

## 2. Consistency model and conflict resolution

**Model, in named layers**
1. *Committed log:* linearizable per doc (Raft; ack = commit). There are no cross-document transactions.
2. *Replicas:* CRDT strong eventual consistency. Any two replicas that have applied the same set of committed ops and the same skip-set have identical state.
3. *Tentative view:* read-your-writes by local apply. Monotonic reads are defined on the committed prefix (the watermark never regresses). Tentative remote ops may be retracted (§1 protocol).

| Data | Mechanism |
|---|---|
| Text | Fugue (non-interleaving sequence CRDT) with Peritext formatting marks, run-length-encoded IDs |
| Object properties (canvas) | LWW register per (object, prop), ordered by (lamport, actorID). Wall clock is never used |
| Hierarchy / z-order | Kleppmann move-tree (undo-redo by (lamport, actor)) plus fractional indexing. A cycle-creating move is a deterministic no-op |
| Sets (reactions, tags) | Add-wins OR-set |
| Delete vs concurrent edit | Delete wins for text. For objects, the tombstone wins over property edits and a restore op exists |

**Convergence.** Merges are commutative, associative, and idempotent (dedupe by opID). Any delivery order gives the same state *given* causal delivery (ops with missing deps are buffered), identical schema-version interpretation, and the same skip-set. The DSO publishes `stateHash(index)` every 1,000 ops for plaintext docs. For E2EE docs, clients publish signed hashes instead (§1). A mismatch triggers resync and a divergence report.

**30-day offline**
- The client keeps a local replica and a durable outbox. Reconnect sends `SYNC(versionVector, outbox)`. It returns a delta, or snapshot + tail past the window. Dedupe is by opID. The server keeps no per-client sync state beyond a device registry row, because the client pulls from the immutable log and snapshots. The only server cost is the retention already modeled (30-day log, 35-day tombstones).
- **LWW:** an offline edit carries a stale lamport, so it loses to *any* intervening online edit of the same property. It wins only if nobody touched that property since. This is deterministic and shown in property history. Lost-edit exposure is measured (LWW overwrites and delete-vs-edit conflicts are logged), not assumed.
- **Move-tree:** integrating an offline batch undoes and redoes all later moves, O(moves in window). The budget is 20k moves × 10 µs = 0.2 s for a p99 doc. Docs with >200k moves in the window rebuild from a checkpoint on a background thread.
- **Revocation:** ops submitted after ACL revocation are rejected and parked in an owner-review inbox (cap 10k ops per doc, 90-day retention). We cannot verify pre-revocation authorship, so some legitimate offline edits are lost. *Convergence* is guaranteed. *Acceptance* of every legitimate offline edit is not.
- Convergence is guaranteed for offline durations ≤30 days, with a 5-day margin to the 35-day tombstone horizon. Beyond that it is best-effort via the skeleton.

**Why not the alternatives**
- **OT** needs server-side transformation. That breaks E2EE, and transform cost grows with a 30-day concurrency window.
- **Figma-style server-authoritative LWW for text** loses intent.
- **CRDT without a sequencer** has no commit point for RPO, no exact revocation position, and no revocation-aware snapshot or compaction. Snapshots themselves are possible.

**Costs:** per-char metadata (≈1.3–2× plain text after RLE), tombstones, and LWW surprises.

## 3. Guarantee matrix (three most severe conflicts)

**Why these three.** They are the conflicts where meeting one requirement forces violating another at the design level, with no amount of money removing the trade:
- A is physics and CAP (durability × latency × availability).
- B is law versus collaboration.
- C is cryptography versus server-side function.

Considered and covered elsewhere: 12-month compatibility × emergency security fixes (§1, list below) and offline × revocation × GC (§2).

**A. Durability × latency × availability under region failure**

| Property | Preconditions | Single-region failure | Partition | Degradation | Decision | Cost ($/mo) | Sacrificed |
|---|---|---|---|---|---|---|---|
| RPO≈0 | Ack after fsync on ≥2 of 3 voters in distinct regions [1] | ≥1 copy of every acked op survives; 2 voters re-form quorum | Minority cannot commit (CP); clients edit locally, shown "unsaved" | 2nd voter lost → read-only; never ack on 1 copy | Raft 2-of-3 cross-region | $24.6k: $15.9k inter-region + $8.6k follower apply (80 raw vCPU ×3.75 ×$28.8) | Ack latency [2]; minority write availability |
| p99 <200 ms in-region | Client ≤50 ms RTT; leader in-region; lease valid | Docs led from the failed region stall ≤12 s (typically 5 s). LATAM→use adds ~115 ms RTT, p99 ≈270 ms | Leader serves tentative fan-out ≤2 s under lease, then read-only | Tick 20→50 ms; shed presence first | Tentative fan-out decoupled from durable ack | $0 infra (same bytes) | "Visible = durable"; LATAM/MEA latency SLO during failover |
| RTO ≤2 min + 99.99% | Warm followers; ≤40% steady, ≤60% post-loss CPU; ticketed admission; 3-vantage detector [3] | Groups led ≤12 s; editors submit ≤30 s; forced-admit stop 61 s; contractual cap 120 s | Detector without quorum does not evacuate (a stall beats split-brain); term fencing still guards safety | Admission: submit > join (editing) > join (idle) > presence > search | Size for N+1 | +$38.5k vs a 70%-steady-utilization design | Efficiency; ≈6 worst-case region events per 30 days exhaust the budget |

[1] The ack path is DSO seq → local fsync + 2 remote followers.
[2] Commit p99 (server side, DSO seq → watermark) ≈ 1.3 × median RTT to nearest follower + 10 ms (2 fsyncs + batching):
- NA 46 ms (RTT 28).
- EU 49 (RTT 30).
- MEA 49 (uae-led, bom nearest, RTT 30).
- APAC 85 (sin-led, bom RTT 58) to 101 (tyo-led, sin RTT 70).
- LATAM 160 (sao-led, use RTT 115), or 190 with use down (usc RTT 137).

[3] Failure model for the RTO bound: one region lost; survivors and links healthy; directory healthy; offered load ≤ sizing.

**B. EU residency × EU/US collaboration × EU-local survival**

| Property | Preconditions | Single-region failure | Partition | Degradation | Decision | Cost ($/mo) | Sacrificed |
|---|---|---|---|---|---|---|---|
| Residency | EU tenant; data plane in EU voter set; policy-as-code; EU HSM root | Re-replicate only inside the EU (eu-par) | EU↔world cut: EU fully served; non-EU collaborators lose EU docs | Fail closed, never cross-border | No relays, CDN, index, backup, or logs outside the EU; EU-only anycast prefixes | $3.0k (eu-par pilot light: 75 vCPU $2.2k + 8.3 TB NVMe $0.8k) | Non-EU latency; global cross-jurisdiction search/AI; **the literal "never leaves the EU" at the end-user device** |
| EU↔US collaboration | One home jurisdiction per doc (no cross-jurisdiction multi-master) | US editors reconnect to surviving EU regions (RTT change ≤30 ms) | US editors cut off; local edits queue, subject to device policy | Viewer mode, editor cap | Remote sessions into the home jurisdiction | $3.1k (39 TB/month of cross-region traffic as internet egress, already in the egress line) | US-side p99 ≈160–410 ms (meets 800, not 200). Under strict tenant policy, no US offline cache |
| EU survival of a region loss | 3 EU voters + eu-par; EU ≤40% CPU | 2 voters keep quorum. Cold: primary, buddy, WORM copies in 3 distinct EU regions, so ≥2 survive | If the 2 survivors partition, EU writes halt (CP) | RF2 until reseed (hot ≈30 min = 10 min provisioning + 20 min copy; cold ≈6.5 h at 40 Gbps), plus an in-flight tail of ≤1 s at RF1 | 3 voters + spare region | $9.6k (EU N+1: 334 extra vCPU vs a 70% design) | A 2nd EU failure while degraded halts EU writes |

Reseed arithmetic: the hot copy is 5.8 TB (25% × 23.2 TB), which is 1,160 s at 5 GB/s. A cold copy is 116 TB (25% × 465 TB), which is 23,250 s ≈ 6.5 h.

**C. E2EE × search × admin recovery (× server-side validation)**

| Property | Preconditions | Single-region failure | Partition | Degradation | Decision | Cost ($/mo) | Sacrificed |
|---|---|---|---|---|---|---|---|
| Search | S1: server sees ciphertext only. S2: server also sees keyed tokens | S1 unaffected. S2 index blocks replicate like segments, and if all copies are lost a member client re-uploads them (the server cannot rebuild) | Local search works offline | S2 off → local only | S1 default: client-side encrypted index. S2 opt-in: SSE with keyed trigram tokens and padded buckets, a distinct security mode | S1 $0 (client storage ≈35% of synced text). S2 +$0.9k (10/90 × $8.3k index) | Server-side ranking, snippets, discovery of unsynced docs. S2 leaks token equality, frequency, and access patterns |
| Admin recovery | RK 3-of-5 or tenant HYOK; shares never at provider | Unaffected (keys outside the regional data plane) | Needs admin quorum online | <3 shares → data unrecoverable by design | Tenant-admin recovery with transparency log | $0 infra (one extra ~100 B wrap per epoch) | "Only members can read" (admins can). Provider-assisted recovery is not offered |
| Server merge, validation, compaction | Payload unreadable | Same as plaintext (log replication is payload-agnostic) | Same | Clients produce signed snapshots; DSO sequences only | Client-side compaction and poison defense | +$5.4k (E2EE docs are incompressible: +65 TB stored ×$72.45 per stored TB + $0.7k segment copies) | Shadow-apply, malware scan, previews, AI. Slower poison-pill mitigation |

**What cannot be met**
1. The literal "EU content never leaves the EU" at the end-user-device boundary (see §0). It holds for provider infrastructure only.
2. Loss of the entire EU jurisdiction: EU tenants go unavailable, with no US failover.
3. Two simultaneous regions lost in one voter set: writes halt.
4. Provider-wide failure: unmitigated in the base design.
5. In-region p99 <200 ms for LATAM/MEA during failover, and for same-region pairs editing a doc homed in another jurisdiction (p99 ≈270–590 ms with one loss round, §5).
6. In viewer-tier sessions of docs with >100 editors, off-viewport ops are applied on scroll (≤300 ms after scroll), not at submit + 200 ms.
7. 99.99% if region events plus other causes exceed the budget (≈6 worst-case region events per 30 days exhaust it alone).
8. E2EE together with server-side search, AI, and validation. S2 is the leakier compromise.
9. Provider-assisted admin recovery for E2EE tenants.
10. The 12-month client window during an emergency security fix (we force-expire the vulnerable version), and new features for old clients (they get fallbacks).
11. Guaranteed acceptance of legitimate offline edits across ACL revocation, and guaranteed survival of offline LWW edits.
12. "Visible = durable" at the same time as <200 ms.
13. In v1: 30-day offline and 12-month compatibility (until P2, §6).

**Requirement status**

| Requirement | Status | Where |
|---|---|---|
| p99 <200 ms in-region | Met for region-homed docs and viewer tier; unmet per items 5 and 6 | §3A, §5 |
| p99 <800 ms cross-region | Met (worst pair 591 ms, 631 ms with queueing allowance) | §5 |
| 30-day offline, deterministic convergence | Met for convergence; acceptance not guaranteed (item 11) | §2 |
| EU content never leaves the EU | Met for infrastructure; **unmet literally at the device** | §0, §3B |
| Optional E2EE with search and admin recovery | Met via S1 and tenant-held recovery; S2 is a distinct mode | §3C |
| 99.99%, RPO≈0, RTO ≤2 min | Met for single-region failure under the stated failure model | §3A, §4A |
| 12-month clients | Met except item 10 | §1 |

## 4. Failure analysis

### 4A. Full regional outage (eu-dub lost; compute and storage gone)

**Fencing.** The epoch is the Raft term, enforced at four layers.
1. Raft quorum.
2. Conditional object writes: segment names include the term, and manifests change by CAS on (term, version).
3. Gateways reject ops carrying a stale `leaderEpoch`.
4. A 2 s leader lease with an election timeout ≥3 s, so there is no dual-leader window. A zombie (partitioned but alive) region self-fences to read-only within 2 s, and the directory is only a cache.

**Safety invariants (the proof obligations, model-checked in §7)**
- I1: an acked entry is fsynced on ≥2 voters in distinct regions.
- I2: Raft leader completeness, so any elected leader's log contains every committed entry.
- I3: the watermark advances only on a current-term entry at quorum.
- I4: tentative entries are never reported as committed.
- I5: GC ordering. WAL truncation happens only after the segment object is verified in 2 regions, the manifest CAS has succeeded, and a *trusted* snapshot covers the range. Object-segment deletion happens only after 30 days, a trusted fold checkpoint, and a confirmed WORM copy.

**Hard RTO bound.** The failure model is one region lost with survivors and links healthy.

| Stage | Hard bound | Mechanism |
|---|---|---|
| Client detects | ≤3 s | 3 missed 1 Hz heartbeats; IP-literal endpoints for all 3 voter-set regions, no DNS wait |
| Re-election | ≤12 s | Pre-vote, randomized 3–4 s timeout, ≤3 rounds |
| Routing | ≤0.3 s | Stale directory costs one epoch-mismatch redirect |
| Admission, editing clients (P0) | ≤25 s after first attempt | Server-assigned ticket slot, so repeated rejection cannot happen. Any P0 request older than 60 s is force-admitted ahead of the queue |
| First submit | ≤1 s after join | 1 Hz batch; lazy DSO activation ≤1 s |

- Editors therefore submit by **≤30 s** (design), ≤61 s (forced-admit ceiling), and ≤120 s (contractual cap). The remaining ≥59 s of margin is deliberately unallocated.
- Idle sessions (P1) drain lazily under their ticket slots after the P0 window closes, at about 2k/s for a 750k-session region, so a full drain takes ≤5 min. Idle sessions are outside the RTO measure until they edit. A first keystroke promotes a session to P0, with the next ticket ≤25 s away.
- **Critical path:** t ≤ max(t_detect + t_admit, t_elect) + t_route + t_submit = max(3 + 25, 12) + 0.3 + 1 = 29.3 s, which is the ≤30 s design figure. Clients whose directory cache points at the failed region are inside this bound: their first attempt fails within the 3 s detection, and the epoch redirect costs 0.3 s.
- **Client-side fallback:** a client with no success at 61 s (for example, a stuck mobile NAT binding) re-handshakes on a fresh QUIC connection and tries each other endpoint in turn (≤2 × 20 s timeouts). It reaches ≤101 s, which stays inside 120 s.
- **Detector verdict:** the verdict (3 vantage points × 3 misses ≈ 10 s) only steers new sessions (BGP/DNS withdrawal, directory epoch bump). Safety never depends on it, because fencing is by term.
- **Preconditions:** the bound holds under the failure model in §3A footnote [3]. If the directory is down, load exceeds sizing, or a second fault occurs, it does not hold. Those causes are budgeted separately below.

**Timeline (EU, eu-dub lost)**

| T+ | Event |
|---|---|
| 0 | Failure |
| ≤3 s | Clients miss 3 heartbeats; follower election timers (3–4 s, randomized) start expiring |
| 4–12 s | ≈1,311 EU groups (4,096 × 8/25) led from dub elect leaders on fra/sto. Pre-vote, ≤3 rounds, and 2 of 3 is a normal quorum, so no unsafe recovery |
| ~10 s | Detector verdict: BGP/DNS withdrawal, directory epoch bump |
| 3–28 s | P0 editors admitted by ticket at ≈3.2k joins/s |
| ≤29.3 s | Last P0 editor submits. DSO activation is lazy per doc (50 ms–1 s) |
| 28–328 s | P1 idle sessions drain |

**EU-internal survivors**
- **Hot data:** fra and sto each hold a full copy, so RF3 becomes RF2.
- **Cold data:** primary, buddy, and WORM copies sit in 3 distinct EU regions, so ≥2 survive.
- **Reseed:** eu-par is a pilot light (75 vCPU, 8.3 TB NVMe). Hot reseed takes ≈30 min (10 min provisioning plus 20 min copy). Cold reseed takes ≈6.5 h.
- **RF1 window:** ops acked on the lost region plus one survivor are RF1 until the new leader re-replicates them to the other survivor. That is ≤1 s for the in-flight tail, typically ≤0.2 s. A second region loss inside that window is the declared out-of-scope double fault.

**Capacity per region (unequal regions, shared voters).** The 40% steady and 60% post-loss targets are enforced per voter set, not as a fleet average. Effective demand is 832 raw vCPU × 1.5 = 1,248, split by traffic share. Each set's capacity is sized so that losing its largest region leaves survivors ≤60%, with the failed region's load shifted in proportion to survivor capacity.

| Set | Demand D | Capacity per region | Total | Steady | Worst loss → survivors |
|---|---|---|---|---|---|
| NA | 436.8 | use 468, usc 364, usw 364 | 1,196 | 36.5% | use → 728 cap, **60.0%** |
| EU | 312.0 | fra 280.8, dub 260, sto 260 | 800.8 | 39.0% | fra → 520 cap, **60.0%** |
| APAC | 374.4 | sin 374.4, tyo 343.2, bom 280.8 | 998.4 | 37.5% | sin → 624 cap, **60.0%** |
| LATAM | 74.9 | sao 187.2 | 187.2 | 40% | sao → NA absorbs (see below) |
| MEA | 49.9 | uae 124.8 | 124.8 | 40% | uae → bom/fra, +25 each |
| **Fleet** | **1,248** | | **3,307.2** | **37.7%** | |

The skew uplift over a uniform 40% (3,120 vCPU) is +187.2 vCPU. A physical region in two sets (use, usc, bom, fra) is counted once, by the load it serves. Groups it leads for other product regions re-elect within ≤12 s. That adds <0.15 weighted seconds (LATAM groups led from use ≈10%, MEA groups led from fra ≈25%).

**Reconnect surge (every large region, peak load).** Join cost is 17.6 ms raw (0.7 × 8 + 0.3 × 40), or 26.4 ms effective. P0 joins arrive at N/125 per second (20% of sessions over 25 s), and P1 starts after the P0 window. Shedding presence and index maintenance frees ≈35% of the 141 + 152 effective vCPU of those two tiers, weighted by the set's share.

| Lost region | Sessions | P0 joins/s | Join CPU | Survivor capacity | Before surge | Peak with surge | After shedding |
|---|---|---|---|---|---|---|---|
| us-east | 750k | 6.0k | 158 | 728 | 60.0% | 81.8% | 67.7% |
| sin | 600k | 4.8k | 127 | 624 | 60.0% | 80.3% | 66.2% |
| eu-fra | 450k | 3.6k | 95 | 520 | 60.0% | 78.3% | 64.2% |
| eu-dub | 400k | 3.2k | 84 | 541 | 57.7% | 73.3% | 59.8% |
| sao | 300k | 2.4k | 63 | 1,196 (NA incl. LATAM) | 42.8% | 48.1% | n/a |

At ρ ≈ 0.82, the M/M/1 p99 queueing wait for a 150 µs DSO service is ≈4.6 × ρ/(1−ρ) × 0.15 ms ≈ 3 ms. The 40 ms allowance in §5 covers GC and interference.

**SLI and request-weighted availability.** The fixed SLI is (successful join + submit requests) ÷ (total join + submit requests) over a rolling 30 days, including regional outages and excluding offline local editing.
- **Request mix:** submits dominate. Average 0.5M/s × 2.592M s = 1.296×10¹² submits, against 5.4×10⁹ joins, so submits are 99.6% of requests.
- **Counting rule:** each failed attempt is a failed request. Tickets suppress retries, so 1 attempt/s per affected editor for T seconds is a ceiling.
- **Budget:** a regional event at traffic share s, duration T, and load factor ℓ (1 = average, 2 = peak) consumes s × T × ℓ "full-outage seconds". The budget is 0.0001 × 2,592,000 = 259.2 s.
- **Cross-check:** a us-east event at the cap, average load, fails 0.15 × 0.5M × 120 = 9.0M submits. That is 9.0M ÷ 1.30×10¹² = 6.9×10⁻⁶, or 6.9% of the budget, matching 18 s.

| Lost region | Share | Design (T = 30 s, ℓ = 1) | Cap (T = 120 s, ℓ = 1) | Worst (T = 120 s, peak ℓ = 2) |
|---|---|---|---|---|
| us-east | 15% | 4.5 s (1.7%) | 18.0 s (6.9%) | 36.0 s (13.9%) |
| sin | 12% | 3.6 s (1.4%) | 14.4 s (5.6%) | 28.8 s (11.1%) |
| eu-fra | 9% | 2.7 s (1.0%) | 10.8 s (4.2%) | 21.6 s (8.3%) |
| eu-dub | 8% | 2.4 s (0.9%) | 9.6 s (3.7%) | 19.2 s (7.4%) |
| sao | 6% | 1.8 s (0.7%) | 7.2 s (2.8%) | 14.4 s (5.6%) |

- **Extra impacts:** surviving-region editors on docs led from the failed region add ≈0.12 weighted s (≈1% of editors × ≤12 s stall). EU docs with US editors add ≈0.5 weighted s at the design bound (≈1.7% of editors × 30 s) and ≈2 s at the cap.
- **Idle joins:** a join deferred by a ticket and served in its slot is a success. Even if all 600k idle us-east joins were counted failed, that is 600k ÷ 1.3×10¹² = 4.6×10⁻⁷, or 0.5% of the budget.
- **Worst event:** a us-east event at peak and the cap costs ≈38 s including stalls and joins (36 + 0.24 + 1.3). So 6.8 such events exhaust the budget, and §3A rounds down to ≈6.

**Control plane and shared dependencies** (static stability: no cloud control-plane call on the request path)

| Dependency | Failure | Behavior | SLI effect |
|---|---|---|---|
| Global directory (64 GB, replicated to all 12 locations) | Service outage | Reads served from local replicas; stale routes fixed by epoch-mismatch redirect. Writes queue. Docs created in jurisdiction X stay routable via X's local authority | Only first-open of a cross-jurisdiction doc created in the last ≤10 min fails. Degraded-mode target: ≥99.9% of joins; submits unaffected |
| KMS/HSM | Regional loss | Each jurisdiction runs KMS in all 3 regions with replicated key material. DSOs cache unwrapped DEKs of active docs. E2EE is client-keyed | None for active docs; new-doc key wrap +≤100 ms via another region |
| DNS | Outage | Signed client config carries IP-literal unicast endpoints for every voter-set region | None for failover |
| Anycast PoP | PoP loss | BGP reroutes. Clients fall back to unicast VIPs | ≤3 s per affected client |
| Deploy/config | Bad rollout | 2% cells, automatic rollback on SLI burn within ≤5 min | Budgeted below |
| Provider IAM/control API | Outage | Data plane keeps running; scale-out and reseed block, so RF2 lasts longer | None for serving |

**Brownouts and gray failures.** A region or node is drained when any of these holds:
- Submit success is <99% or ack p99 is >3× the 7-day baseline for 20 s, seen from ≥3 vantage points.
- Node fsync p99 is >50 ms for 10 s, or packet loss is >2%.
- A gateway or relay is a 2σ outlier on error rate.

Draining means leadership transfer off the node, weight 0 for new sessions, and ≤30 s end to end. It reuses the failover path, and a drained region still serves as a follower.

**Error budget (259.2 s per 30 days)**

| Cause | Allocation | Basis |
|---|---|---|
| Region events | 60 s | One cap-bound peak us-east event (38 s) plus one typical event |
| Deploys/config | 40 s | 5 deploys × 2% cell × ≤5 min at full failure ≈ 30 s, plus margin |
| Control plane (directory, KMS, DNS, edge) | 40 s | Degraded-mode joins |
| Gray failures/brownouts | 50 s | Drain within 30 s of onset |
| Provider/dependency incidents | 40 s | |
| Unallocated | 29.2 s | |

### 4B. Poison-pill edit (crashes every receiving client)

Plaintext and E2EE documents get separate guarantees.

**Prevention: plaintext docs**
- Every op gets structural limits (depth, 64 KB, rate), per-doc capability gating so new op types never reach old clients, and a native apply on a copy-on-write overlay.
- High-risk ops (new types, unusual shape, ≈1%) plus a 0.1% random sample are shadow-applied in sandboxed WASM cores for all 13 builds, with a bounded anchor window and a 20 ms timeout (a hang counts as a crash). This costs ≈20 raw vCPU (10k ops/s × 13 × 150 µs). Memory is 13 pooled instances × 64 MB per DSO host, inside the 4 GB/vCPU.
- CI fuzzes decode and apply on all 13 builds.
- Limit: bugs that need context outside the window or interactions with other ops are not prevented. Detection below covers them.

**Prevention: E2EE docs.** There is no semantic prevention, because the server sees ciphertext. The server validates the envelope only: device signature against the ACL, size, rate, dedupe, key epoch.

**Detection and containment (client)**
- Before applying a remote batch, the client durably writes `applying:[opIDs]`.
- A restart with the marker set counts as a crash. After 2 crashes the client enters **safe mode**: read-only, with the batch bisected in a sandbox.
- It reports `suspect(opID, clientVer, stackHash)` (envelope-only data).
- Restart backoff bounds the crash loop to 2 iterations.
- **E2EE local skip:** after bisection, the device skips the culprit locally and uploads a signed `LocalSkip`, within ≤10 s. This works with no server help.

**Mitigation (server)**
- **Plaintext docs:** on a report, the DSO *replays* the op in the sandbox on the reporter's build. If it crashes there, `Quarantine{opIDs}` is appended immediately. One report suffices, and collusion cannot fake a crash that replay verifies. If replay does not reproduce, the thresholds apply: ≥3 devices from ≥2 users, or ≥10% of sessions in 60 s, or the owner or a doc admin. This gives small docs (≤3 sessions) an unattended path for reproducible pills.
- **E2EE docs:** the server cannot verify. Doc-wide Quarantine needs ≥3 devices (≥2 users) or an owner/admin action. Docs with fewer devices rely on each device's local skip. Every affected device is already running, but doc-wide quarantine is unbounded without an admin.
- **Quarantine semantics:** a grow-only skip-set with `appliesFrom`, treated as "never applied". Text ops anchored to characters inserted by the pill re-anchor to the pill's recorded left origin (nearest surviving ID in the skeleton). A move or property op depending on the pill becomes a no-op or applies against the prior value. These rules are in the convergence tests.
- **Snapshot ordering:** a snapshot is `trusted` only after a 15 s hold with no suspect reports (30 s for E2EE, plus ≥2 device signatures). Recovery and I5 GC use trusted snapshots only. A Quarantine invalidates any snapshot whose range contains a quarantined op, and sync rejects a snapshot whose skip-set hash lacks the quarantine set. A stale snapshot therefore cannot reintroduce the pill.
- **Sync order:** quarantine records are sent *before* ops, so late and offline clients never apply the pill.
- **Class-wide kill switch:** for plaintext docs, a signed denylist of op signatures (type, schemaVer, shape hash) is enforced at gateways and the DSO in ≤30 s. Under E2EE only the op class, size bucket, and client build are visible, so the switch fences by *authoring build* (a min-version rule), not by op shape.

| T+ (plaintext, reproducible) | Event |
|---|---|
| 0 | Pill committed |
| 0.5 s | First crashes |
| ~3 s | Relaunch, second crash |
| ~6 s | Safe mode, reports sent |
| ~7 s | DSO replay reproduces; Quarantine committed |
| ~10 s | Clients resume (skip-set delivered ahead of ops) |
| ≤30 s | Repaired snapshot (gen+1) |

- **Bounds:** the plaintext design p99 is ≤15 s, with a hard cap of 120 s (2 crash cycles ≤8 s, report delivery with retries ≤30 s, replay ≤1 s, commit ≤0.2 s). For E2EE, each device recovers in ≤10 s and doc-wide quarantine takes ≤120 s when ≥3 devices report.
- **Blast radius:** one doc, up to 5,000 sessions.

## 5. Capacity and cost

### Reference arithmetic

**Load.** Peak sessions are 5M (NA 1.75M, EU 1.25M, APAC 1.5M, LATAM 0.3M, MEA 0.2M). Active editors are 1.0M, giving 1.0M msg/s × 300 B = 300 MB/s in. With 2 receivers each, fan-out is 2M msg/s = 600 MB/s. Presence from 2.5M sessions at 0.5 Hz × 60 B is 75 MB/s in and 150 MB/s out. Joins are 5M ÷ 1,200 s = 4.2k/s × 18.5 KB = 77.7 MB/s. Active docs are ≈670k (1.0M editors ÷ 1.5 per doc). Peak egress is 600 + 150 + 77.7 = 827.7 MB/s, and the 50% diurnal average is 413.9 MB/s.

**Offline load (5% of DAU/day).**
- That is 2.5M syncs/day averaging 100 KB up and 1 MB down.
- Egress is 28.9 MB/s average.
- CPU is 58 syncs/s at the 2× diurnal peak × 100 ms = 6 raw vCPU.
- Ingest is 7.5 TB/month (+2.8% of durable bytes, handled in the sensitivity table).
- The server keeps only a device-registry row (≈0.5 KB × 150M devices = 75 GB, inside the metadata line). The client pulls from the immutable log, so there is no per-client sync state.
- The 0.1% long-offline cohort (50k users, ≤2 reconnects/month at 20 MB up and 5 MB down) is ≈2.5 TB up and 0.5 TB down per month, which is negligible.

**Routing assumption.** A cross-region doc has 1 of 3 collaborators remote, so every op crosses a region boundary exactly once. That is 10% of ops, or 100k msg/s, 30 MB/s peak, 15 MB/s average, ≈39 TB/month. v1 serves it by direct connection to the home gateway (internet egress). The P3 relay path would be backbone transfer at $0.02/GB. We price all 39 TB at $0.08/GB as a conservative bound. The bytes are already inside the 600 MB/s fan-out, so there is no separate line.

**Log growth and projection.**
- Durable ops average 105 MB/s (1.0M × 210 B × 50%), which is 9.07 TB/day or 272 TB/month raw.
- The 30-day full-fidelity tail is ≈272 TB raw (27% of the 1 PB logical total).
- Older history folds ≈10× (assumed; 5–20× in the sensitivity table), leaving +27 TB/month of retained logical history.
- The scenario's 1 PB is taken as month 0. Without expiry, logical size is 1.16 PB at month 6 and 1.33 PB at month 12 (+33%). That is +152 TB stored × $72.45 = **+$11.0k/month by month 12** (+4.2% of the model), outside the steady-state base below.
- Version thinning (daily to weekly checkpoints after 12 months) is available and not credited.

**Storage assumptions**
- **Stored fraction:** 0.465 (0.9/2.5 + 0.1 × 1.05), so 1 PB logical is 465 TB stored. E2EE ciphertext is incompressible with 5% envelope overhead.
- **Hot set:** 160M docs × 145 KB = 23.2 TB per copy. The 145 KB is a 125 KB uncompressed snapshot (25% × 500 KB) plus a 20 KB mean unfolded log tail and overlay index.
- **Search index:** 35% of the 225 TB logical non-E2EE current state (25% × 0.9 PB) = 78.75 TB.
- **Backup (1.15×):** one full WORM copy (465 TB) in a separate account and a distinct region of the same jurisdiction, plus 4.1% for fold-deleted tail segments held 5 days past primary deletion (114 TB/month × 5/30), plus 10.9% for an assumed 9.3%/month deletion and supersession churn × 35/30. The lock is 35 days.
- **Restore:** WORM lags the log by ≤60 s (segment flush). A restore after a double-region loss can lose acked ops newer than the last flushed segment. That is outside RPO ≈ 0 and listed under "cannot be met" #3. EU restore is ≈6.5 h.
- **WAL NVMe:** flush within 60 s or 256 KB per doc, then 1 h retention for follower catch-up. Capacity is 210 MB/s × 3,600 s = 756 GB per replica × 3 replicas × 2 zone mirrors = 4.5 TB, ÷0.7 = 6.5 TB.
- **Memory:** priced at 4 GB/vCPU (13.2 TB provisioned). Sessions need 5M × 8 KB = 40 GB, active DSO state ≈84 GB × 1.3, and WASM pools ≈1 GB per DSO host. All fit easily.

**Raw peak vCPU**

| Component | Derivation | vCPU |
|---|---|---|
| Gateways and relays | Keepalive 5M × 1 Hz × 10 µs = 50; connect 4.2k/s × 2 ms = 8; edit in 1M × 30 µs = 30; edit out 2M × 30 µs = 60; presence 3.75M/s × 25 µs = 94 | 242 |
| DSO | 1M × 150 µs | 150 |
| Raft follower apply | 2 × 1M × 40 µs | 80 |
| Join / ACL / API | Joins 4.2k/s × 17.6 ms = 74; offline syncs 6; ACL/API/auth 1.5k/s × 20 ms = 30 | 110 |
| Snapshot / compaction | 2.2k snapshots/s × 2 ms = 4.5; zstd 0.7; fold 105 ÷ 50 MB/s = 2.1; manifest/GC/verify 2.7 | 10 |
| Shadow-apply (plaintext) | 10k ops/s × 13 builds × 150 µs | 20 |
| Search index + query (provisional until P3 benchmarks) | Index 670k docs ÷ 30 s = 22.3k/s × 3 ms = 67; merge 34; queries 8.3k/s × 10 ms = 83 | 184 |
| Auth, KMS, directory serving, telemetry agents | Provisional | 36 |
| **Total** | | **832** |

**Provisioning:** 832 × 1.5 (platform overhead) ÷ 0.40 = 3,120, plus the 187.2 skew uplift from §4A = **3,307 vCPU** (37.7% steady). The N+1 premium over a 70%-utilization design is 3,307 − 1,783 = 1,524 vCPU = $43.9k/month. That restates §3A's $38.5k, the uniform-basis figure, by +$5.4k, and the EU share from $9.6k to $10.2k.

### Monthly cost (720 h; vCPU-month = $28.8; every line derived)

| Line | Derivation | $/month |
|---|---|---|
| Compute | 3,307.2 × $28.8 | 95,247 |
| EU pilot-light spare | 75 vCPU × $28.8 = 2,160 + 8.3 TB NVMe × $100 = 830 | 2,990 |
| Internet egress | (413.9 realtime + 28.9 offline sync) = 442.8 MB/s × 2.592M s = 1,147.7 TB × $80/TB | 91,819 |
| Inter-region | Raft: 272.2 TB × 2 remote followers = 544.3 (uncompressed). Segment copy to buddy: 272.2 × 0.465 = 126.6. WORM copy: 126.6. Total 797.4 TB × $20/TB | 15,948 |
| Hot SSD | 23.2 TB × RF3 = 69.6 ÷ 0.7 = 99.4 TB × $100 | 9,943 |
| WAL NVMe | 6.5 TB × $100 | 649 |
| Search index | 25% on SSD at RF2: 78.75 × 0.25 × 2 ÷ 0.7 = 56.25 TB × $100 = 5,625; 75% in object at 2 copies: 118.1 TB × $23 = 2,717 | 8,342 |
| Metadata / ACL / directory | 2 TB × RF3 ÷ 0.7 = 8.57 TB × $100 = 857; directory 64 GB × 12 locations = 0.77 TB × $100 = 77 | 934 |
| Object (primary + buddy) | 465 TB × 2 × $23 | 21,390 |
| Backup (WORM, separate domain) | 465 TB × 1.15 × $23 | 12,299 |
| **Total** | | **259,561** |

**Normalized: $259.6k ÷ 50M DAU = $0.0052 per DAU per month.** Cost drivers are compute and spare 37.9%, egress 35.4%, storage 20.6%, and inter-region 6.1%. The E2EE storage effect (+65 TB stored, +$5.4k including segment copies) is already inside the object, backup, and inter-region lines via the 0.465 stored fraction.

**Optional second-provider EU-resident WORM copy** (provider-wide failure; restore RTO in hours, outside the SLO): 465 TB × $23 = $10,695 plus a segment stream of 126.6 TB × $80 = $10,128, so **+$20.8k/month**. It is not in the base total.

### Latency budget (p99, ms; a budget, not a proof)

| Hop | In-region | EU sender → us-east receiver (EU doc) |
|---|---|---|
| Client → gateway | 25 | 25 |
| Gateway auth/rate (cached) | 2 | 2 |
| Gateway → DSO | 2 | 2 |
| DSO validate + sequence | 8 | 8 |
| DSO → relay | 3 | 3 |
| Relay coalescing tick | 20 | 20 |
| Relay → client | 27 | 70 |
| Receiver decode + apply | 15 | 15 |
| Sum | 102 | 145 |
| + one loss-recovery round | +50 → 152 (48 ms margin) | +80 → 225 |

Tentative fan-out does not wait for the durable ack, so the ack latencies in §3A are not on this path. Summing per-hop p99s is an engineering budget, not a proof of end-to-end p99. The decisive evidence is end-to-end measurement under load and failure (§7, V4).

**Pair latencies on an EU-homed doc** (pair = sender leg + receiver leg + 50 ms server-side; one loss round adds the larger leg's recovery R). Leg values (one-way p99 L / loss round R): EU 26/50, us-east 70/80, us-west 100/160, sao 115/190, uae 80/120, bom 100/130, sin 115/165, tyo 145/251.

| Pair | No loss | One loss round |
|---|---|---|
| EU ↔ EU | 102 | 152 |
| EU ↔ us-east | 146 | 226 |
| us-east ↔ us-east | 190 | 270 |
| us-west ↔ us-west | 250 | 410 |
| uae ↔ uae | 210 | 330 |
| bom ↔ bom | 250 | 380 |
| sin ↔ sin | 280 | 445 |
| sao ↔ sao | 280 | 470 |
| tyo ↔ sao | 310 | 561 |
| **tyo ↔ tyo (worst)** | **340** | **591** |

- The worst pair is 591 ms, or 631 ms with a +40 ms allowance for p99 queueing at 82–85% CPU. The margin to 800 ms is 169–209 ms.
- US-side pairs span 146–410 ms. Same-region pairs on a foreign-homed doc span 270–591 ms. Both meet 800 ms and miss 200 ms, as listed in §3.
- LATAM clients served from us-east during a sao loss: 152 (in-region with a loss round) + 115 ms RTT ≈ 267 ms.

### Workload distributions (assumed and modeled)

| Missing distribution | Treatment |
|---|---|
| Doc popularity / members per doc | Zipf, exponent 2.2, truncated at 5,000: mean 3.1, p99.9 ≈ 190, ≈2×10⁻⁵ of docs (≈40k) have ≥5,000 members. Assume ≤5 concurrently live 5,000-session docs, adding ≤85 MB/s at event peak |
| Maximum size | 25 MB state, 64 KB op, paste chunked. These are product limits, tested against Docs-class and Figma-class workloads in P3 |
| Log growth | Projected above (+33% logical by month 12) |
| Reconnect bursts | Failover (§4A), release, mobile-carrier outage, Monday 9 am local. Ticketed admission, jitter, IP-literal fallback |
| Offline size and duration | Modeled above |
| E2EE share | 10% |

### Sensitivity (change in $/month vs base)

| Assumption | Low | High |
|---|---|---|
| Per-op CPU (434 raw vCPU, ×0.5 / ×2) | −24.8k | +49.7k |
| Presence share of sessions (25% / 100%) | −13.2k | +26.4k |
| Offline payload per sync (0.25 MB / 4 MB) | −4.5k | +18.0k |
| Compression (3.0:1 / 2.0:1) | −5.0k | +7.5k |
| Hot fraction (5% / 15%) | −3.7k | +8.7k |
| Fold ratio (20× / 5×), added by month 12 | +5.5k | +22.0k |
| Cross-region share (5% / 20%) | ≈0 | ≈0 (bytes unchanged; latency only) |

Per-op CPU and the session mix (presence, offline) dominate the uncertainty, so they are re-measured in P3.

### Heavy-tail scenario 1: 5,000-session document (one byte basis: 300 B wire, 60 B presence)

**(a) Naive, uncapped (5,000 editors, 20% typing, 50% sending presence)**
- Edits: 1,000 typists × 300 B = 300 KB/s per receiver × 5,000 = **1.5 GB/s**, which is 2.5× the platform's entire 600 MB/s edit fan-out peak.
- Presence: 2,500 senders × 0.5 Hz × 60 B = 75 KB/s per receiver × 5,000 = 375 MB/s.
- Total 1.875 GB/s ≈ **$540/hour** for one doc.

**(b) Uncapped, with the mitigation stack**
- Viewport-scoped fan-out (≈5% of ops per receiver) is delivered on the editor tick: 50 msg/s × 300 B = 15 KB/s × 5,000 = 75 MB/s.
- Presence is capped at the 50 nearest cursors: 1.5 KB/s × 5,000 = 7.5 MB/s.
- Total 82.5 MB/s ≈ $23.8/hour, 23× cheaper. Off-viewport ops are applied on scroll (≤300 ms after scroll), which is listed in §3 as unmet item 6.

**(c) Policy: 100 editor slots, 4,900 viewers (the P3 serving policy)**
- Editors: 100 typists × 300 B × 100 receivers = 3.0 MB/s.
- Viewers: viewport ops 5 msg/s × 300 B = 1.5 KB/s × 4,900 = 7.35 MB/s.
- Presence 7.5 MB/s.
- Total 17.85 MB/s (143 Mbps) ≈ **$5.1/hour**, about 105× cheaper than (a).
- A one-level relay tree of 50 relays × 100 clients keeps DSO egress at 1.5 MB/s (full edit stream to each relay) plus ≈0.3 MB/s presence aggregation, ≈1.8 MB/s (14 Mbps).
- The tree's only extra hop is the DSO → relay hop already in the budget (+3 ms). In-viewport p99 stays ≈105 ms, under 200 ms.
- A 5,000-client meeting-start burst at 500 joins/s serves a 100 KB viewport-first chunk each: 50 MB/s for 10 s, with the rest lazy.
- This is the P3 state. v1 serves ≤1,100 sessions per doc (§6).

### Heavy-tail scenario 2: reconnect storm

The surge table in §4A covers us-east (750k sessions), sin, eu-fra, eu-dub, and sao on unequal regions. Peak survivor CPU is ≤82% before shedding and ≤68% after. The in-region queueing effect at those levels is inside the 40 ms allowance.

### Budget ceiling

**$0.0086 per DAU per month ($430k/month), 1.66× the model.** The buffers compound, since each covers a different uncertainty class: 1.20 × 1.10 × 1.25 = 1.65, and $259.6k × 1.65 = $428.3k, rounded up.
- +20% for unmodeled platform services (observability, WAF/DDoS, CI/staging).
- +10% for drills and heavy-tail surges.
- +25% for price and estimate error, with CPU per op and the session mix dominating.
- If the second-provider copy is enabled (+$20.8k), the buffers shrink to 53%, and the ceiling still holds.

The prompt gives no revenue model. As an assumption, ≥3% paid conversion at $10 per month is ≈$0.30 per DAU-month, so the ceiling is ≈2.9% of revenue. The defense does not depend on it.

If we breach the ceiling, the levers are:
1. Tighter coalescing and viewport sampling. A 10% egress cut saves $9.2k.
2. Committed-use or private-interconnect discounts on compute and egress (73% of spend; not in the reference prices).

**Not a lever:** raising steady utilization above the §4A per-set sizing. At 50%, survivors would pass 75% before the surge and exceed 100% in the eu-dub storm.

## 6. Cuts and roadmap

**Cuts for v1 (end of P2, month 9).** The end-state design in §§0–5 meets the fixed requirements except the items in §3 "cannot be met". v1 intentionally does not, and is not offered as compliant.

| Cut | Restored in | Requirement affected until then |
|---|---|---|
| Offline 7 days; client window 6 months | P2 | 30-day offline and 12-month compatibility **unmet** |
| LATAM docs home in the NA voter set and MEA docs in the APAC set; no sao/uae compute or voters | P3 | LATAM/MEA latency and failover use NA/APAC. In-region SLO unmet for those users |
| Cap of 100 editors and 1,000 viewers (≤1,100 sessions per doc); no relays for unpinned docs | P3 | Heavy-doc analysis in §5 is the P3 state |
| No E2EE (search S1, tenant recovery) | P4 | E2EE requirement unmet |
| No S2 SSE search | P5 (optional) | None; S1 meets "E2EE with search" |
| No HYOK (Shamir recovery key only) | P5 | None |
| Canvas limited to frames, shapes, and LWW properties | P3 | Figma-class scope |

| Phase | Scope | Measurable exit criterion |
|---|---|---|
| P0 (0–3 mo) | NA only: text CRDT, DSO, 3-region Raft, snapshots | p99 in-region ≤200 ms at 100k concurrent sessions in load test; V1 passes (invariants I1–I5 model-checked, 10⁶ simulation seeds, 0 violations); ≥100 staging region-kill drills with 0 lost acked ops and max RTO ≤120 s |
| P1 (3–6) | EU jurisdiction, residency enforcement, EU/US collaboration | ≥100 staging EU region-kill drills plus 3 production drills, max RTO ≤120 s; 0 honeytoken hits in non-EU storage and logs for 30 days; EU hot reseed measured ≤30 min |
| P2 (6–9) | 30-day offline, compatibility CI, poison-pill quarantine, directory-kill drill | 10⁶ randomized 30-day-offline merges converge bit-identically; compatibility matrix of 13 frozen builds cut from monthly protocol tags (N-12…N, with an accelerated schema-change corpus) green; V9 plaintext criteria pass; directory-outage drill keeps ≥99.9% of joins |
| P3 (9–12) | APAC, LATAM, MEA; editor slots, viewer mode, relays for unpinned docs | V4 passes over 30 days of probes for every §5 pair; reconnect drills (400k eu-dub, 750k us-east sessions) with max first submit ≤120 s; 5,000-session replay egress ≤22 MB/s |
| P4 (12–15) | E2EE with client-side search and tenant recovery | Third-party crypto audit with 0 critical findings; 3-of-5 recovery restores 100% of test DEK epochs including after key rotation; offline-across-rotation test 100% rule-conformant; no E2EE plaintext markers server-side |
| P5 (15–18) | Availability and cost hardening, optional S2, HYOK | 99.99% SLI in every rolling 30-day window across 90 consecutive days including ≥1 region drill; measured $/DAU ≤$0.0086 at scaled extrapolation; a real ≥12-month-old client build passes V8 |

## 7. Verification plan

Statistics only bound failure rates and never prove them. The safety guarantees rest on the model-checked invariants (V1), and the tests check the implementation against them.

| ID | Area | Method | Pass criterion | Covers |
|---|---|---|---|---|
| V1 | Commit, fencing, GC safety | TLA+ spec of I1–I5, model-checked on a 3-voter, region-kill model. Deterministic simulation (10⁶ seeds with region kills, partitions, clock skew, fsync lies), plus trace validation of the implementation against the spec | 0 invariant violations; 0 lost acked ops; 0 dual-leader commits | RPO ≈ 0 |
| V2 | RPO under region kill | Jepsen-style at ≥1.2× per-group reference peak, plus an overload run at 1.5×. Kill a random region mid-load and audit acked opIDs against the recovered log | 0 missing in ≥300 staging kills plus 3 production kills. The rule-of-three bound (≤1% per kill at 95%) is reported as supplementary evidence only | RPO ≈ 0 |
| V3 | RTO (hard maximum) | ≥100 staging drills and quarterly production drills; ≥5,000 distributed probers including cellular and CGNAT. Fault variants: admission queue at 150% load, stale directory, DNS blocked, one slow follower, NAT rebind | **Max over all probers and drills ≤120 s** (alert at 45 s). No percentile substitutes | RTO ≤2 min |
| V4 | Latency | ≥1,000 probers per region pair (every pair in §5), PTP/GPS-synced stamps, load at 1.5×, also in failover state | Rolling 24 h p99: in-region <200 ms for region-homed docs; every cross-region pair <800 ms; documented exceptions at or under their §5 values | p99 requirements |
| V5 | CRDT convergence | Property tests and nightly fuzz: 10⁹ permuted interleavings including a virtual-clock 30-day offline, quarantine skip-sets and re-anchoring, schema versions N-12…N | 0 divergent state hashes; 10⁶ randomized 30-day-offline merges bit-identical | Offline convergence |
| V6 | Residency | OPA policy-as-code on IaC; daily honeytoken docs searched in non-EU storage, logs, backups, heap dumps, and edge captures; BGP announcements for EU-pinned prefixes from EU PoPs only, confirmed by non-EU probes; egress allow-list audit; quarterly external audit; strict-policy devices checked for empty caches after session end | 0 hits; 0 violations. The device-boundary exception is excluded from "hits" and tested only for policy-flag conformance | EU residency |
| V7 | E2EE | Marker strings; crypto audit; recovery drill; offline-across-rotation test (client offline 30 days across ≥3 epoch rotations and one removal); S1 offline search | 0 plaintext hits server-side; 0 critical findings; 100% DEK recovery; 100% rule conformance | E2EE |
| V8 | Compatibility | 13 builds (N-12…N) × servers N and N+1 on golden logs; live mixed-version docs where new op types are attempted; rolling-deploy test | Server log and segments round-trip byte-identical (hash compare). Client unknown fields preserved semantically (canonical decode equal), 100%. Old clients never crash and show fallbacks. Deploy error ≤0.001% | 12-month clients |
| V9 | Poison pill | 1,000 injected pills per class. Large plaintext docs; small docs (≤3 sessions) with reproducible pills; E2EE docs; 10⁴ pill/snapshot interleavings; 50% report loss and delay | Plaintext max ≤120 s (design p99 ≤15 s); small reproducible docs ≤60 s unattended; E2EE per-device local skip ≤10 s (100%) and doc-wide ≤120 s with ≥3 devices; ≤2 crash-loop iterations; 0 untrusted snapshots used in recovery | Poison-pill analysis |
| V10 | Heavy tail | Replay the 5,000-session doc ((a), (b), (c)) and storms (us-east 750k, sin 600k, eu-fra 450k sessions) | In-viewport editor p99 <200 ms; off-viewport apply ≤300 ms after scroll; doc egress ≤22 MB/s (176 Mbps) and DSO egress ≤2.2 MB/s in (c); storm CPU ≤90% before shedding and ≤75% after; max first submit ≤120 s | Heavy-tail model |
| V11 | Control plane and gray failure | Kill the global directory for 30 min; block DNS; drop one KMS region; inject 200 ms fsync stalls and 5% loss in one region | Join success ≥99.9% with the directory down; 0 failed submits on existing sessions; drain within 30 s; 0 acked-op loss | Failure analysis, §4A |
| V12 | Availability | Prober plus server-side SLI per the fixed definition; monthly chaos (zone, region, partition), drills counted | ≥99.99% in every rolling 30-day window across 90 consecutive days; error-budget burn tracked per the §4A table | 99.99% |
| V13 | Offline load | Replay 5% of DAU reconnecting at 1 MB per sync | Egress and CPU within model +20% | Capacity model |
| V14 | Cost | Quarterly bill against the model, with sensitivity re-run | Unit cost within ±15% of model and at or under the ceiling | Budget ceiling |

**Traceability.** Each row of the §3 requirement-status table maps to at least one V-row: latency to V4, offline to V5 and V13, residency to V6, E2EE to V7, availability, RPO, and RTO to V1, V2, V3, V11, and V12, and 12-month clients to V8. The poison-pill analysis maps to V9. 


``` 


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


# Original AI Response

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


# Revised AI Response 

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


# AI Response with Validation Layer ( BASIC VERSION )

``` 

# Global Collaborative Editor: Design v4

## Summary and requirement status

**Design in brief.** Five product regions run on 11 active cloud regions, plus one EU pilot-light spare.
- Each shard group is a 3-voter Raft group with voters in distinct regions. The Raft log is both the commit point and the sequencer, and a per-document Session Owner (DSO) sits with the leader.
- Replicas are CRDTs: Fugue text, plus commit-ordered LWW and a move-tree for canvas. This is why 30-day offline merges converge.
- Two jurisdictions (EU, ROW) hold content, and EU content is never stored, processed or cached outside the EU.
- E2EE is a per-tenant policy under which the server only sequences envelopes.
- Modeled cost is $0.0054 per DAU-month against a $0.0090 ceiling.

| Requirement | End state (P5) | v1 (end of P1, month 6) | Verified by |
|---|---|---|---|
| p99 <200 ms in-region | Met on the no-loss p99 budget for region-homed docs (worst intra-region pair 172 ms). Exceptions (§3 items 5–6) are ≈0.8% of deliveries | NA and EU only. LATAM ≈270 ms (homed in NA). APAC and MEA not served | V4, V10 |
| p99 <800 ms cross-region | Met (worst pair 591 ms, 631 ms with a queueing allowance) | Met for NA/EU pairs | V4 |
| Offline ≤30 days, deterministic convergence | Convergence met. Acceptance of every legitimate offline edit is not guaranteed (item 11) | Unmet: 7 days | V5, V13 |
| EU content never leaves the EU | Met for provider infrastructure. **Unmet literally at the end-user device** (item 1) | Same | V6 |
| Optional per-tenant E2EE with search and admin recovery | Met via client-side search (S1) and tenant-held recovery. S2 is optional. Server-side discovery of unsynced E2EE docs is unsupported (item 8) | Unmet (P4) | V7 |
| 99.99%, RPO ≈ 0, RTO ≤ 2 min | Met for a single-region failure under the stated failure model (a conditional SLO, §4A) | NA and EU tenants only | V1–V3, V11, V12 |
| 12-month clients, zero downtime | Met except emergency security expiry (item 10) | Unmet: 6-month window | V8 |

## 0. Fixed assumptions and definitions

**Failure domains** (the last column lists what each domain does *not* isolate)

| Domain | Definition | Independent of | Shares fate with |
|---|---|---|---|
| Zone | One AZ (≥3 per cloud region) | Power, cooling, rack network of other zones | Regional control plane |
| Region | One cloud region (metro) | Compute, storage, power, network of other regions | Provider IAM and DNS, our binaries and config (hence cell-based staged rollout), global directory, anycast control |
| Jurisdiction | Legal/admin domain with its own cloud accounts, KMS/HSM root, deploy pipeline, break-glass | Other jurisdictions' accounts, keys, operators | The cloud provider, public DNS/BGP, the signed-build trust root, the global directory (routing metadata only) |
| Backup domain | Separate account, compliance-mode object lock, separate KMS key, in a region other than primary and buddy | Region or zone loss, credential compromise, operator error | For EU content, the EU jurisdiction's fate (backups must stay in the EU). In v1, also the provider's |

**Privilege separation (what "independent" means operationally).**
- Deploy credentials are distinct and short-lived per jurisdiction and per region.
- The backup account has its own organization-level administrators under a two-person rule.
- It uses compliance-mode object lock, so not even root can shorten it. No deploy or automation role can delete it or change its policy.
- Break-glass access is quorum-gated and logged. For the EU it is held by the EU-established operating entity.
- What remains shared is stated in the table: the provider's organization-level control plane and the signed-build trust root.

**Why "region" means a cloud region.** The failure unit is the cloud region, the largest unit with independent power, network and control plane that providers actually lose. Product regions (NA, EU, APAC, LATAM, MEA) are traffic and placement groupings. Losing a whole product region needs two or more simultaneous cloud-region failures. That is the declared out-of-scope tier, and it is priced in §5 rather than assumed away.

**Topology.** There are 5 product regions on 11 active cloud regions, plus one pilot-light spare (12 deployed locations). Traffic shares are in parentheses.
- NA: us-east (15), us-central (10), us-west (10)
- EU: eu-fra (9), eu-dub (8), eu-sto (8). eu-par (France) is a pilot-light spare and not a voter until promoted (§4A).
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

- In v1, LATAM docs home in the NA set. APAC and MEA are not served until P3, when their sets are deployed (see §6).
- A physical region can sit in two sets (use and usc in NA and LATAM, bom in APAC and MEA, fra in EU and MEA). Losing it removes one voter from each set, but only groups *led* from it re-elect. Availability is therefore counted by serving region (§4A), not by voter-set membership.
- Non-EU content may sit in EU regions. The reverse is forbidden.
- Leader placement is optimized (minimax RTT to the doc's editors) only among the eligible regions of the doc's voter set. An EU-pinned doc whose median editor is in the US still leads in fra, dub, or sto.

**Jurisdictions.** There are two.
- **EU** is residency-bound, with its own accounts, KMS/HSM root and deploy pipeline.
- **ROW** covers all other tenants. The NA, LATAM, APAC and MEA voter sets run in ROW accounts, with a ROW KMS root replicated in use, usc and sin.
- ROW makes no residency promise. Its voter-set regions exist for latency and survivability, so replicas of a ROW tenant may sit in tyo, bom or fra. fra hosts a ROW-account MEA voter, in a separate account and KMS from the EU voter.
- Country-specific residency (India, Brazil, and so on) is not offered. Each would need its own jurisdiction built from the EU template, which single-region countries cannot satisfy (§3, item 15).
- ROW has no pilot-light spare, because nothing prevents provisioning a replacement region anywhere in ROW (≈10 min). The EU needs eu-par because its boundary forbids that.

**EU "region" and boundary.** An EU region is one cloud region in an EU member state (fra DE, dub IE, sto SE, spare par FR). The EU boundary is the territory of the 27 member states plus an EU-established operating entity with an EU-resident HSM root. EEA-only states are not treated as EU. Every acked EU op is on ≥2 of 3 EU voters, so any single EU region loss leaves 2 voters (a quorum) and at least one copy of every acked op, all inside the EU.

**Out of scope (stated as unmet):**
- Loss of two regions in one voter set, including a correlated loss of a whole product region. The next tier is priced as a lower bound in §5 and not adopted.
- Loss of the whole EU jurisdiction.
- Provider-wide failure. This is unmitigated in the base design. An optional second-provider, EU-resident WORM copy is priced in §5 (≈$23.5k/month, restore RTO in hours, outside the SLO).

**Residency interpretation, and the literal conflict.** Read literally ("no EU content byte exists on any device outside the EU"), the requirement is incompatible with EU and US users collaborating on one document, because a US collaborator who can read the doc holds its content on a non-EU device. Even E2EE does not help, since the plaintext is rendered there. We therefore **mark the literal reading unmet at the end-user-device boundary**. That exception covers the authorized non-EU collaborator's device (storage and processing) and the public-internet TLS transit to it, which no provider component stores or caches. We satisfy this operational reading: provider infrastructure outside the EU never stores, processes, or caches EU content. That covers ciphertext, indexes, backups, logs, crash dumps, and CDN. EU-pinned traffic enters only through EU-announced anycast prefixes and IP-literal EU endpoints, so none of our non-EU PoPs carry it. Tenants can opt into stricter policy at the cost shown in §3B:
- EU-device-only: no US collaboration.
- No-offline-cache: non-EU devices hold content in memory only. This is client-enforced and best-effort on managed devices (MDM). It is not claimed on unmanaged devices, where swap, crash dumps and screen capture cannot be controlled.

**Content and metadata classification.** Rule: content is stored, processed and cached only in its home jurisdiction. Restricted metadata follows the same rule, so where the content/metadata line falls on the op envelope does not change where it may live. Only the two global routing items and k-anonymous aggregates are exported.

| Item | Content or metadata (sensitivity) | Form and location | Justification and residual leak |
|---|---|---|---|
| Document body, ops (envelope and payload), snapshots, comments, titles, thumbnails | Content | Plaintext (std) or ciphertext (E2EE); home jurisdiction only | |
| Search indexes | Content | Std: plaintext inverted index. S1: client-side only. S2: keyed-token blocks, treated as content | S2 leaks token equality, frequency, and access patterns |
| Segments, backups, manifests | Content | Same form as source; EU copies EU-only; manifests hold docID and a per-jurisdiction keyed hash (HMAC) only, in-jurisdiction | Keyed, so cross-tenant equality of segments is not testable |
| Doc ID (random 128-bit, no embedded time or tenant) | Metadata, global routing | Global directory: docID→(jurisdiction, shard group, epoch), ≈32 B/entry, 64 GB total | Random IDs are still linkable, so the directory holds no tenant, user, or time fields, is access-logged, and has no join key to user IDs outside the jurisdiction. Residual leak: a docID exists and has a home jurisdiction |
| Tenant ID→jurisdiction | Metadata, global routing | Global | Random; needed for login routing |
| User ID | Metadata, personal data | Home jurisdiction. Docs hold pairwise pseudonymous subject IDs, and the mapping stays in the user's home jurisdiction | GDPR personal data |
| User→docs reverse index | Metadata, restricted | User's home jurisdiction; docID and home jurisdiction only, no titles. Sized and priced in the metadata line (§5) | Needed for "shared with me"; titles are fetched from the doc's jurisdiction on open |
| ACL, sharing graph, membership | Metadata, restricted (content-adjacent) | Doc's home jurisdiction only | Reveals who collaborates with whom |
| Timestamps, sizes | Metadata, restricted | In-jurisdiction. Exported globally only as aggregate buckets with k≥1,000 docs and no tenant breakdown | Residual: bucketed volume per jurisdiction |
| Op envelope | Metadata, restricted (the op as a whole, envelope plus payload, is treated as content for placement) | In-jurisdiction. Std: actor, counter, lamport, size, type. E2EE: only docID, device pseudonym, counter (dedupe), size padded to 256 B, keyEpoch, schemaVer, op class (content/ACL/key/snapshot) | E2EE lamport, deps, and semantic type are inside the ciphertext. Residual leaks (accepted, documented): typing cadence, size buckets, timing of membership, key-epoch and snapshot events, doc existence and activity, connecting jurisdiction |
| Telemetry | Metadata, hashed IDs with jurisdiction-local salt | No raw IDs leave | |

**Added workload assumptions**
- Diurnal average = 50% of peak.
- Documents and sessions: "average 3 collaborators" is read as 3 concurrently connected sessions per active doc, so 5M sessions give 1.67M active docs. Assumption: editing is clustered (40% of active docs have ≥1 editor, 1.5 editors each), so 1.0M editors sit in 670k edited docs. Each message has 2 receivers. Independent placement would give 815k edited docs (§5 sensitivity).
- E2EE is a tenant policy. 10% of docs belong to E2EE tenants, with the same average size (so 10% of bytes). Their data is incompressible ciphertext with +5% envelope overhead.
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
- Binary assets (images, attachments) are content-addressed objects of ≤25 MB each, outside the 25 MB document-state limit. They are replicated like segments and assumed to be inside the 500 KB average.

## 1. Architecture and data model

**Tiers.** Edge (anycast, L4 pass-through, TLS never terminates outside the doc's jurisdiction) → Gateway (stateless; QUIC/WebSocket, auth, rate limits) → Relay (fan-out, viewport filtering) → **Document Session Owner (DSO)**, one per active doc, co-located with the shard-group Raft leader under its lease → **Raft log** → segment and snapshot store (object storage) → search indexers. A control plane holds the directory, config, and key management.
- For plaintext docs the DSO holds CRDT state.
- For E2EE docs it holds only sequencing state: head index, dedupe window, key epochs, and ACL position.

**Sharding.** docID → 2¹⁶ vshards → ~16k Raft groups (EU ≈ 4,096, ≈122k docs per group), with online split and merge. A group is the unit of failover. A *cell* (~2% of tenants) is the unit of rollout.
- **Hot-doc isolation and sequencing admission.** The sequencer caps each doc at 500 batched messages/s (≈150 KB/s). Each group uses weighted-fair queueing, and no doc may take more than 25% of the group's append capacity (≈20k entries/s). Excess load is back-pressured by stretching the client batch interval 1 s → 2 s → 4 s. A doc above 25% for 60 s is split into its own single-doc group online. This protects unrelated docs in a shared group.

**Write path**
1. The client sends an idempotent batch keyed by opID = (actorID, counter).
2. The gateway checks auth, size, and rate.
3. The DSO validates:
   - *Plaintext docs:* decode, schema, and limits, then apply on a native reference core over a copy-on-write overlay. Ops flagged high-risk (new op types, unusual depth or shape, ≈0.9% of ops) plus a 0.1% random sample (1% in total) are also shadow-applied in sandboxed WASM cores for **all 13 supported client builds** (N-12…N), using a bounded context window around the op's anchors.
   - *E2EE docs:* envelope validation only (device signature checked against the ACL, size, rate, dedupe, key epoch).
4. The DSO appends to Raft. The leader fsyncs locally and sends the entry to both followers in parallel. The entry's `(term, index)` is its `seq`, not a separate counter. In parallel the DSO fans out *tentatively* via relays, tagged with `(term, index)`.
5. **Commit = leader plus the first follower to fsync and ack (2-of-3).** The commit watermark advances and `ACK(opIDs)` goes to the sender. The slower follower catches up asynchronously and is not on the ack path. The watermark piggybacks to everyone.

**Ordering coordinates.** The Raft index is group-wide. A doc's total order is the subsequence of its group's log that carries its docID, so per-doc linearizability follows from the group's total order. headIndex, snapIdx and version vectors are in group-index coordinates. A segment covers a group index range for all docs of the group and has a per-doc offset index in its footer.

**Tentative-state protocol.** Receivers hold committed state Sc (ops ≤ watermark) and a journal J of tentative ops keyed by opID. The view is Sc ⊕ J.
- On a term change the new leader publishes `TermStart(term, firstIndex, retained[(index, opID)])`, where `retained` lists the old-term uncommitted entries it holds. Receivers keep a tentative entry only if its opID is retained (re-tagged with the new index) or reappears. All other older-term entries at or beyond the divergence point are dropped, then the view is recomputed from Sc ⊕ J′.
- Ops that depend on a dropped op wait up to 10 s for their live sender to re-upload it (idempotent by opID), then are dropped. The author's client re-anchors its own dependents to the retracted op's recorded origin.
- A tentative entry can disappear from the log on a leader change even while its sender is alive. The sender has no ack, so it re-uploads by opID and the op reappears. A *permanent* retraction occurs only if the sender and every durable holder are lost inside the commit window (46–160 ms), and it appears to receivers as an ordinary remote deletion.
- Clients key their outbox on opID, not `seq`.

**Data model**

| Object | Key → fields | Storage |
|---|---|---|
| Document | docID → tenantID, homeJur, shardGroup, schemaVer, textAlgoGen, keyMode (inherited from the tenant's E2EE policy at creation), aclVer, headIndex, snapshotIdx/gen, size, created/updated | Metadata DB, Raft RF3, in the doc's home jurisdiction |
| Op log entry | (groupID, term, index) → docID, opID, lamport, opType, schemaVer, keyEpoch, deps, payload (plain or ciphertext, stored verbatim and never re-serialized by the server) | Raft WAL on NVMe, mirrored across 2 zones. Flushed per group as zstd segments `{group}/{startIdx}-{term}.seg` every 60 s or 384 KB raw (typically 60–250 KB compressed) to object storage in primary and buddy regions, plus a WORM copy (§5). Manifests live in the metadata DB and change by CAS on (term, version). A partial upload is invisible until its manifest CAS succeeds |
| Snapshot | (docID, gen, snapIdx) → state blob, skip-set hash, tombstone skeleton, stateHash, keyEpoch, trust flag | Hot: SSD on each voter (uncompressed). Every voter materializes its own from the log, so there is no snapshot network traffic. Cut every 2,000 ops, 15 min of activity, or 256 KB tail. Cold: object, written by the leader at DSO deactivation and every 6 h of continuous activity, packed into the group's segment objects |
| ACL, membership, restore, comments | Log entries of the same doc (comments are log entries with an anchor and a pseudonymous author; ciphertext under E2EE) | Same log, so revocation has an exact position in the total order |
| Search index | Per-doc index segments; std: plaintext; S2: encrypted token blocks | Primary and buddy region (RF2). Not in the WORM copy: it is derived data, rebuilt from restored content in hours, outside the SLO |
| Presence | (docID, session) → pseudonymous user, cursor/selection, viewport, TTL 10 s | DSO memory only, never persisted, encrypted under E2EE. Rebuilt from client heartbeats within 10 s after any DSO move or failover |
| Directory | docID→(jur, group, epoch); group→leader (cache, validated by Raft term) | Global, replicated to every region, minimal fields. Single writer per entry (the home jurisdiction's authority); epoch changes by CAS on the Raft term |

**History policy.**
- Full-fidelity ops are kept for 30 days. Older history folds into named and daily checkpoints.
- Tombstone *content* is dropped at 35 days. The ID-run *skeleton* is kept until the document is purged, so late anchors still resolve deterministically. It holds ranges only, no content: actor index, counter start, length and left-origin reference, ≈10 B per tombstoned run. That is ≈7.5 KB (≈1.5% of logical size) on the average doc, inside the 25% snapshot allowance. Canvas objects keep an equivalent tombstone record (id, parent at delete, ≈40 B) until purge.
- The skeleton carries only pseudonymous actor IDs. Erasure of a user's pseudonym mapping unlinks them.

**Zone analysis.**
- A voter's WAL is a synchronous 2-zone pair (fsync on both before the voter counts as durable, +1 ms), so a single node or zone loss degrades a voter without losing it.
- Snapshot SSD is single-zone and rebuildable from object storage.
- Gateways, relays, and DSOs are stateless or lazily reactivated. Zone loss leaves 2/3 of a region's capacity, so utilization goes from 40% to 60%, the same N+1 rule as for regions.

**Session routing.**
- The client asks the directory for the doc's jurisdiction (cached ≤60 s) and connects to that jurisdiction's gateway. For EU-pinned docs, US clients cross the Atlantic directly, with no non-EU relays. Gateways resolve group→leader from cache, and a stale cache is corrected by an epoch-mismatch redirect (one RTT).
- Clients carry a signed config with IP-literal endpoints for all three voter-set regions, so failover has no DNS-TTL dependence. The config is not baked into the build: it is refreshed on every connect, cached for ≥30 days, and signed by the build trust root. Anycast and IP-literal bootstrap endpoints are kept stable for ≥13 months, so a client that was offline for 30 days still reaches the service and refreshes its config.
- In v1, all remote clients connect straight to the home gateway. For unpinned (non-EU) tenants from P3, a client attaches to its nearest local relay, which forwards over the backbone to the home DSO's gateway.

**Identity across jurisdictions.**
- Display names and avatars are not copied between jurisdictions. A client resolves each member's pseudonym through that member's home identity service, authorized by a signed profile-share token issued at invite time. Results are cached 24 h on the device, with initials as the fallback.
- Device credentials are short-lived tokens (≤1 h) signed by the user's home IdP and validated at the doc's gateway against cached IdP keys.
- Account-level revocation propagates as a signed statement from the home jurisdiction into each doc's ACL log, asynchronously. The ≤1 h token expiry bounds staleness, and during a jurisdiction partition remote users' tokens expire and writes fail closed.
- ACL changes remain positions in the doc's own log, so revocation keeps an exact position.

**Protocol and schema evolution (12-month clients, zero downtime)**
- The supported window is N-12…N (13 monthly builds). The wire format is protobuf with append-only fields. Unknown fields and op types are preserved as opaque bytes.
- The log is immutable. Read-time upcasters (N-12…N) convert plaintext ops, so history is never rewritten. E2EE payloads are upcast by clients, since the server cannot read them.
- New semantics ship expand → dual-write → contract. Contract waits ≥13 months after release.
- Old clients keep working without downtime or crashes on every existing feature. New features reach them only as an additive embed with a fallback representation, or are gated per tenant by min client version. Old clients preserve what they cannot interpret but cannot use it.
- A text-CRDT algorithm change (e.g. away from Fugue) is a doc *generation* change, `GenerationChange(index, algo, checkpointHash)`, applied at the same log index by all replicas. It happens only after the tenant's min client version passes the new algorithm's release by ≥13 months. Until then docs stay on the old algorithm.
- Raft log-format bumps use a cluster-wide feature barrier after all replicas upgrade. Rolling deploys run N and N-1 servers side by side.
- Byte-exactness holds at the log and segment level, because the server never re-serializes. At the client it is semantic preservation of unknown fields.
- An emergency security fix can force-expire a vulnerable build (§3, item 10). That is the one deliberate exception.

**E2EE**
- **Policy:** E2EE is a per-tenant policy fixed at tenant and doc creation, and each doc's `keyMode` is inherited from it. Plaintext docs are not converted in place. Moving into an E2EE tenant is a client-side copy.
- **Keys:** a per-doc DEK (XChaCha20-Poly1305) with epochs, HPKE-wrapped per member device. Each new epoch is also wrapped to the tenant recovery public key RK_pub. MLS is used above ~20 members, since TreeKEM is O(log n), which matters for large docs. Key-epoch records form a hash chain in the log, so backups restore the wraps.
- **Server role:** it sequences and replicates envelopes only. Clients produce snapshots, compact, and upcast. Clients compute and sign `stateHash` at checkpoints.
- **Snapshot trust:**
  - A snapshot is trusted when ≥2 distinct devices sign the same hash.
  - Docs with a single active device in the last 30 days use a delayed-trust path. The one device's signature becomes trusted after a 35-day hold, during which the pre-fold segments stay in the WORM copy (the 35-day lock) and the device re-verifies by replay.
  - Trust establishes agreement between devices, not semantic correctness. A bad checkpoint therefore stays recoverable by replay for the lock period.
  - WAL truncation never needs a trusted snapshot (I5a), so single-device docs do not block GC. Only their folding lags, costing +$1.2k/month (§5).
- **Removal:** triggers a new key epoch. Removed members keep what they already had.
- **Admin recovery (tenant-admin, not provider-assisted):**
  - The RK private key is Shamir-split 3-of-5 across admin hardware keys, or the DEK wraps use a tenant HYOK KMS in the tenant's jurisdiction. The provider holds zero shares.
  - The five keys are held by distinct admins in ≥3 countries (for EU tenants, ≥3 inside the EU). A resharing ceremony runs when an admin leaves, and compromise revokes RK and re-wraps.
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
| Object properties (canvas) | LWW register per (object, prop), ordered by the op's **commit position** (term, index) in the log. Tentative order is provisional until commit. Wall clock and Lamport time are never used for LWW: a Lamport counter counts events, not time, so a prolific offline client could out-rank a recent online edit |
| Hierarchy / z-order | Kleppmann move-tree (undo-redo by (lamport, actor), a causally consistent total order). The move op carries `(newParent, fractionalIndex)` atomically, so the tree is authoritative for parentage and the index is only a sibling-order key that cannot disagree with it. Equal indices tie-break by (actorID, opID), and renormalization is an ordinary move op. A cycle-creating move is a deterministic no-op |
| Sets (reactions, tags) | Add-wins OR-set |
| Delete vs concurrent edit | Delete wins for text. For objects, the tombstone wins over property edits and a restore op exists. Connectors, component instances and comment anchors that reference a deleted or quarantined object become no-ops that keep a dangling reference with a visible placeholder. Object IDs are never reused |

**Convergence.** Merges are commutative, associative, and idempotent (dedupe by opID). Every causally valid delivery order gives the same state, *given* causal delivery (ops with missing deps are buffered), identical schema-version interpretation, and the same skip-set. The DSO publishes `stateHash(index)` every 1,000 ops for plaintext docs. For E2EE docs, clients publish signed hashes instead (§1). A mismatch triggers resync and a divergence report.
- E2EE causal buffering is purely client-side, since deps live inside the ciphertext. Because the DSO appends before it fans out, the committed log order is already a linear extension of causality except around retractions, and dependents of a retracted op wait for it.
- **Op identity.** opID = (actorID, counter). The actorID is a per-install random 128-bit value. The counter is persisted ahead of use in blocks of 1,000 and is never reused. A restored or cloned device learns the server's highest-seen counter for its actor in the SYNC reply and, on a rollback, rotates to a new actorID.
- **Dedupe.** It uses per-actor counter range sets, which compact into the snapshot's version vector. A duplicate older than a fold is therefore still rejected, and the cost is O(actors) per doc.

**30-day offline**
- The client keeps a local replica and a durable outbox. Reconnect sends `SYNC(versionVector, outbox)`. It returns a delta, or snapshot + tail past the window. The server keeps no per-client sync state beyond a device registry row, because the client pulls from the immutable log and snapshots. The only server cost is the retention already modeled (30-day log, 35-day tombstones).
- **Boundary semantics:** "30 days" means 30 × 24 h since the device's last successful sync. The 5-day margin to the 35-day tombstone horizon absorbs clock error and sync scheduling.
- **LWW:** an offline edit is sequenced when it reaches the log, so it overrides any online edit of the same property committed in the meantime (reconnect-wins, as in server-authoritative canvas tools). This is deterministic and works under E2EE, because the commit position is visible to every replica. The displaced value stays in property history. The client shows "replaced X's newer edit" with one-tap undo (a new op restoring the prior value), and flags overwrites of another actor's edit committed after the op's base. Lost-edit exposure (LWW overwrites and delete-vs-edit conflicts) is logged and measured, not assumed. No guarantee is made that either side's concurrent LWW edit survives (§3, item 11).
- **Move-tree:** integrating an offline batch undoes and redoes all later moves, O(moves in window). The budget is 20k moves × 10 µs = 0.2 s for a p99 doc (10 µs/move is an assumption, benchmarked in V5). Docs with >200k moves in the window rebuild from a checkpoint on a background thread.
- **Revocation:** ops submitted after ACL revocation are rejected and parked in an owner-review inbox (cap 10k ops per doc, 90-day retention). We cannot verify pre-revocation authorship, so some legitimate offline edits are lost. *Convergence* is guaranteed. *Acceptance* of every legitimate offline edit is not.
- Convergence is guaranteed for offline durations ≤30 days. Beyond that it is best-effort via the skeleton.

**Why not the alternatives**
- **OT** can run with a sequencer-only server and client-side transforms, so E2EE alone does not rule it out. We reject it because transform cost and history grow with a 30-day concurrency window, and it has no clean story for non-text canvas types. CRDT ids anchor edits across 30 days of divergence.
- **Figma-style server-authoritative LWW for text** loses intent. We use it only for canvas properties.
- **CRDT without a sequencer** has no commit point for RPO, no exact revocation position, and no revocation-aware snapshot or compaction. Snapshots themselves are possible.

**Costs:** per-char metadata (≈1.3–2× plain text after RLE), tombstones, and LWW surprises.

## 3. Guarantee matrix (three most severe conflicts)

**Why these three.** They are the conflicts where meeting one requirement forces violating another at the design level, with no amount of money removing the trade:
- A is physics and CAP (durability × latency × availability).
- B is law versus collaboration.
- C is cryptography versus server-side function.

Considered and covered elsewhere: 12-month compatibility × emergency security fixes (§1, list below) and offline × revocation × GC (§2).

All costs in the matrices are components of the §5 base total, not additions to it, unless marked "not in base".

**A. Durability × latency × availability under region failure**

| Property | Preconditions | Single-region failure | Partition | Degradation | Decision | Cost ($/mo) | Sacrificed |
|---|---|---|---|---|---|---|---|
| RPO≈0 | Ack after fsync on ≥2 of 3 voters (leader + first follower) in distinct regions [1] | ≥1 copy of every acked op survives; 2 voters re-form quorum | Minority cannot commit (CP); clients edit locally, shown "unsaved" | 2nd voter lost → read-only; never ack on 1 copy | Raft 2-of-3 cross-region | $19.5k: $10.9k Raft inter-region (544.3 TB × $20) + $8.6k follower apply (80 raw ×3.75 ×$28.8) | Ack latency [2]; minority write availability |
| p99 <200 ms in-region | Client ≤50 ms RTT; leader in-region; lease valid | Docs led from the failed region stall ≤12 s (typically 5 s). LATAM→use adds ~115 ms RTT, p99 ≈270 ms | Leader serves tentative fan-out ≤2 s under lease, then read-only | Tick 20→50 ms; shed presence first | Tentative fan-out decoupled from durable ack | $0 infra (same bytes) | "Visible = durable"; LATAM/MEA latency SLO during failover |
| RTO ≤2 min + 99.99% | Warm followers; ≤40% steady, ≤60% post-loss steady CPU (surge transient ≤82% before shedding, ≤68% after, §4A); ticketed admission; 3-vantage detector [3] | Groups led ≤12 s; editors submit ≤30 s; forced-admit stop 61 s; contractual cap 120 s | Detector without quorum does not evacuate (a stall beats split-brain); term fencing still guards safety | Admission: submit > join (editing) > join (idle) > presence > search | Size for N+1 per voter set | +$43.9k vs a 70%-steady-utilization design (in the compute line; $38.5k uniform plus $5.4k per-set skew uplift) | Efficiency; ≈6 worst-case region events per 30 days exhaust the budget |

[1] The ack path is DSO seq → leader fsync, entry sent to both followers in parallel, commit on the first follower's ack. The slower follower is off the ack path but still receives every byte (priced above).

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
| Residency | EU tenant; data plane in EU voter set; policy-as-code; EU HSM root | Re-replicate only inside the EU (eu-par) | EU↔world cut: EU fully served; non-EU collaborators lose EU docs | Fail closed, never cross-border | No relays, CDN, index, backup, or logs outside the EU; EU-only anycast prefixes | $3.0k (eu-par pilot light: 75 vCPU $2.2k + 8.3 TB NVMe $0.8k; its own base line) | Non-EU latency; global cross-jurisdiction search/AI; **the literal "never leaves the EU" at the end-user device** |
| EU↔US collaboration | One home jurisdiction per doc (no cross-jurisdiction multi-master) | US editors reconnect to surviving EU regions (RTT change ≤30 ms) | US editors cut off; local edits queue, subject to device policy | Viewer mode, editor cap | Remote sessions into the home jurisdiction | $3.1k (39 TB/month of cross-region traffic as internet egress, already in the base egress line) | US-side p99 ≈146–410 ms (meets 800, not 200). Under strict tenant policy, no US offline cache |
| EU survival of a region loss | 3 EU voters + eu-par; EU ≤40% CPU | 2 voters keep quorum. Cold: primary, buddy, WORM copies in 3 distinct EU regions, so ≥2 survive | If the 2 survivors partition, EU writes halt (CP) | RF2 until reseed (hot ≈30 min = 10 min provisioning + 20 min copy; cold ≈6.5 h at 40 Gbps), plus an in-flight tail of ≤1 s at RF1. eu-par joins as learner, then voter by joint consensus (§4A) | 3 voters + spare region | $10.2k (EU N+1, skew-sized: 355 extra vCPU vs a 70% design; in the compute line) | A 2nd EU failure while degraded halts EU writes |

Reseed arithmetic: the hot copy is 5.8 TB (the EU's 25% share of the 23.2 TB platform hot set), which is 1,160 s at 5 GB/s. A cold copy is 116 TB (the EU's 25% share of the 465 TB platform stored data), which is 23,250 s ≈ 6.5 h.

**C. E2EE × search × admin recovery (× server-side validation)**

| Property | Preconditions | Single-region failure | Partition | Degradation | Decision | Cost ($/mo) | Sacrificed |
|---|---|---|---|---|---|---|---|
| Search | S1: server sees ciphertext only. S2: server also sees keyed tokens | S1 unaffected. S2 index blocks replicate like segments, and if all copies are lost a member client re-uploads them (the server cannot rebuild) | Local search works offline | S2 off → local only | S1 default: client-side encrypted index. S2 opt-in: SSE with keyed trigram tokens and padded buckets, a distinct security mode | S1 $0 (client storage ≈35% of synced text). S2 +$0.9k, **not in base** (an upper bound if every E2EE doc enables S2: 10/90 × the $8.3k plaintext index line) | Server-side ranking, snippets, discovery of unsynced docs. S2 leaks token equality, frequency, and access patterns |
| Admin recovery | RK 3-of-5 or tenant HYOK; shares never at provider | Unaffected (keys outside the regional data plane) | Needs admin quorum online | <3 shares → data unrecoverable by design | Tenant-admin recovery with transparency log | $0 infra (one extra ~100 B wrap per epoch) | "Only members can read" (admins can). Provider-assisted recovery is not offered |
| Server merge, validation, compaction | Payload unreadable | Same as plaintext (log replication is payload-agnostic) | Same | Clients produce signed snapshots; DSO sequences only | Client-side compaction and poison defense | $5.4k delta vs an all-plaintext fleet, already included in base via the 0.465 stored fraction: +65 TB stored × $72.45 per stored TB (= 2 × $23 + 1.15 × $23) + $0.7k segment copies | Shadow-apply, malware scan, previews, AI. Slower poison-pill mitigation |

**What cannot be met**
1. The literal "EU content never leaves the EU" at the end-user-device boundary (see §0). It holds for provider infrastructure only.
2. Loss of the entire EU jurisdiction: EU tenants go unavailable, with no US failover.
3. Two simultaneous regions lost in one voter set (including a correlated loss of a whole product region): writes halt. If both regions' storage is also unrecoverable, a restore from WORM can lose acked ops newer than the last flushed segment, so RPO is ≤60 s in that case. The next tier is priced in §5.
4. Provider-wide failure: unmitigated in the base design.
5. In-region p99 <200 ms for these classes, with exposure of ≈0.8% of deliveries (§5):
   - LATAM/MEA during failover, and LATAM throughout v1.
   - Same-region pairs editing a doc homed in another jurisdiction (p99 ≈270–590 ms with one loss round).
   - Multi-metro APAC docs under a stacked loss round (≈242 ms).
6. For viewer-tier sessions (the viewers of docs with >100 sessions), off-viewport ops are applied on scroll (≤300 ms after scroll), not at submit + 200 ms.
7. 99.99% if region events plus other causes exceed the budget (≈6 worst-case region events per 30 days exhaust it alone).
8. E2EE together with server-side search, AI, and validation, and server-side discovery of unsynced E2EE docs. S2 is the leakier compromise.
9. Provider-assisted admin recovery for E2EE tenants.
10. The 12-month client window during an emergency security fix (we force-expire the vulnerable version), and new features for old clients (they get fallbacks).
11. Guaranteed acceptance of legitimate offline edits across ACL revocation, and any guarantee that a concurrent LWW edit survives (displaced values are kept in history).
12. "Visible = durable" at the same time as <200 ms.
13. In v1: 30-day offline and 12-month compatibility (until P2), and APAC/MEA service (until P3), §6.
14. E2EE docs are capped at 100 editors and 1,000 viewers. The 5,000-session design is plaintext-only.
15. Moving a document between jurisdictions in place (export/import only), and country-level residency jurisdictions other than the EU.

## 4. Failure analysis

### 4A. Full regional outage (eu-dub lost; compute and storage gone)

**Fencing.** The epoch is the Raft term, enforced at four layers.
1. Raft quorum.
2. Conditional object writes: segment names include the term, and manifests change by CAS on (term, version).
3. Gateways reject ops carrying a stale `leaderEpoch`.
4. A 2 s serving lease on monotonic clocks, with an election timeout ≥3 s.
   - **Assumptions:** clock drift ≤200 ppm and no clock steps. The leader re-checks its lease before every fan-out batch, so a paused leader that resumes after expiry serves nothing.
   - The 1 s margin exceeds drift (<1 ms) by orders of magnitude.
   - The residual is one in-flight tentative batch from a deposed leader, which receivers discard at `TermStart`.
   - The lease only bounds stale *tentative fan-out and reads*. Commit safety never depends on clocks, because it rests on quorum, term and conditional writes.
   - A zombie (partitioned but alive) region self-fences to read-only within 2 s, and the directory is only a cache.

**Safety invariants (the proof obligations, model-checked in §7)**
- I1: an acked entry is fsynced on ≥2 voters in distinct regions.
- I2: Raft leader completeness, so any elected leader's log contains every committed entry.
- I3: the watermark advances only on a current-term entry at quorum.
- I4: tentative entries are never reported as committed.
- I5a (WAL GC): the WAL is truncated only up to an index covered by a segment object verified in 2 regions, with its manifest CAS committed.
- I5b (fold GC): segment objects are deleted only after 30 days, a *trusted* fold checkpoint covering the range, and a confirmed WORM copy of the pre-fold segments.

**Hard RTO bound.** The failure model is one region lost with survivors and links healthy.

| Stage | Hard bound | Mechanism |
|---|---|---|
| Client detects | ≤3 s | 3 missed 1 Hz heartbeats; IP-literal endpoints for all 3 voter-set regions, no DNS wait |
| Re-election | ≤12 s | Pre-vote with priority-staggered timeouts (3/4/5 s by log recency, then region ID), no random split votes, ≤3 candidates |
| Routing | ≤0.3 s | Stale directory costs one epoch-mismatch redirect |
| Admission, editing clients (P0) | ≤25 s after first attempt | Server-assigned ticket slot, so repeated rejection cannot happen. Any P0 request older than 60 s is force-admitted ahead of the queue. |
| First submit | ≤1 s after join | 1 Hz batch; lazy DSO activation ≤1 s (derivation below) |

- Editors therefore submit by **≤30 s** (design), ≤61 s (forced-admit ceiling), and ≤120 s (contractual cap). The remaining ≥59 s of margin is deliberately unallocated.
- **Scope of "affected clients".** The RTO clock runs for every client that attempts an edit at or after the failure.
  - For a client editing at the failure instant, it is anchored at the failure.
  - For a client idle at the failure, it is anchored at its first keystroke, because no system can submit an edit the user has not made.
  - Idle (P1) sessions drain lazily at about 2k/s for a 750k-session region (≤5 min, ≈53 vCPU).
  - A first keystroke promotes the session to P0 immediately. After the initial wave, P0 arrivals are about 500/s (150k editors, 5-min mean editing episode) against a 6k/s slot rate. The ticket wait is therefore ≤1 s, and first submit is ≤3 s after the keystroke (connect 0.3 s, join, lazy activation ≤1 s, 1 Hz batch).
- **DSO activation (≤1 s).**
  - The hot snapshot is on each voter's SSD, uncompressed, so activation reads ≤145 KB.
  - It replays the ≤256 KB log tail (≈1,200 ops × 40 µs ≈ 48 ms) and loads the skip-set and ACL position from the same SSD.
  - E2EE docs rebuild only head index and dedupe window from the tail.
  - The typical figure is 50 ms and p99 ≤1 s. A cold doc adds one object read (≤200 ms).
- **Election bound.** It is deterministic under the failure model.
  - Staggered priority timers mean the first candidate (3 s) runs unopposed.
  - The second (4 s) and third (5 s) cover a failed candidate, each round ≤1 s on healthy links.
  - The worst case is 5 + 1 + 1 = 7 s, which leaves slack under the 12 s cap.
  - Raft liveness is not unconditional. Outside the failure model (degraded links), a missed cap falls to the client fallbacks (61 s, 101 s). V1 simulates 10⁵ elections.
- **Critical path:** t ≤ max(t_detect + t_admit, t_elect) + t_route + t_submit = max(3 + 25, 12) + 0.3 + 1 = 29.3 s, which is the ≤30 s design figure. Clients whose directory cache points at the failed region are inside this bound: their first attempt fails within the 3 s detection, and the epoch redirect costs 0.3 s.
- **Client-side fallback:** a client with no success at 61 s (for example, a stuck mobile NAT binding) re-handshakes on a fresh QUIC connection and tries each other endpoint in turn (≤2 × 20 s timeouts). It reaches ≤101 s, which stays inside 120 s.
- **Detector verdict:** the verdict (3 vantage points × 3 misses ≈ 10 s) only steers new sessions (BGP/DNS withdrawal, directory epoch bump). Safety never depends on it, because fencing is by term.
- **Preconditions:** the bound holds under the failure model in §3A footnote [3]. If the directory is down, load exceeds sizing, or a second fault occurs, it does not hold. Those causes are budgeted separately below. The 99.99% target is therefore a **conditional SLO** (it holds while region events plus other causes stay inside the error budget), not an unconditional guarantee.

**Timeline (EU, eu-dub lost)**

| T+ | Event |
|---|---|
| 0 | Failure |
| ≤3 s | Clients miss 3 heartbeats. The priority-1 follower's election timer (3 s) expires |
| 3–7 s | ≈1,311 EU groups (4,096 × 8/25) led from dub elect leaders on fra/sto by pre-vote then vote. The 4 s timer covers a failed first candidate, and 12 s is the cap. 2 of 3 is a normal quorum, so there is no unsafe recovery |
| ~10 s | Detector verdict: BGP/DNS withdrawal, directory epoch bump |
| 3–28 s | P0 editors admitted by ticket at ≈3.2k joins/s |
| ≤29.3 s | Last P0 editor submits. DSO activation is lazy per doc (50 ms–1 s) |
| 28–328 s | P1 idle sessions drain at ≈2k/s. Any keystroke promotes a session to P0 (≤3 s) |

**EU-internal survivors**
- **Hot data:** fra and sto each hold a full copy, so RF3 becomes RF2.
- **Cold data:** primary, buddy, and WORM copies sit in 3 distinct EU regions, so ≥2 survive.
- **Sizes.** The 5.8 TB hot copy and the 116 TB cold copy are the EU's 25% shares of the platform totals (23.2 TB hot, 465 TB stored), because only EU content is re-replicated inside the EU. eu-par's 8.3 TB NVMe is 5.8 TB ÷ 0.7.
- **Replacement voter.** eu-par is a non-voting pilot light until promoted. Groups are reconfigured in waves of 5%, so catch-up traffic stays ≤40 Gbps.
  1. Scale out eu-par compute (≈10 min).
  2. Add it to each group as a non-voting learner, which streams the hot snapshot and log tail from fra/sto (5.8 TB at 5 GB/s ≈ 20 min).
  3. When lag is <1 s, run a joint-consensus change {fra, sto, dub} → {fra, sto, par}. The old majority (fra and sto) is alive, so the change commits through Raft and cannot split-brain. Dub's removal is part of the same change.
  4. Reseed cold copies in the background (116 TB at 5 GB/s ≈ 6.5 h).
- **ROW sets** (LATAM, MEA) use the same procedure with a replacement region provisioned anywhere in ROW (≈10 min).
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

The skew uplift over a uniform 40% (3,120 vCPU) is +187.2 vCPU.
- **Shared physical regions.** A region in two sets (use, usc, bom, fra) is sized by the load it serves and hosts.
  - Follower apply for LATAM groups (7.2 effective vCPU in total) runs on use and usc. Follower apply for MEA groups (4.8) runs on bom and fra, not on sao and uae.
  - That is ≤1.1% of each host's capacity, and net-neutral to the fleet total.
  - Groups a region leads for another product region re-elect within ≤12 s. That adds <0.15 weighted seconds (LATAM groups led from use ≈10%, MEA groups led from fra ≈25%).

**Reconnect surge (every region, peak load).** Join cost is 17.6 ms raw (0.7 × 8 + 0.3 × 40), or 26.4 ms effective. P0 joins arrive at N/125 per second (20% of sessions over 25 s), and P1 starts after the P0 window.
- **Shedding.** It pauses the failed set's presence fan-out and index maintenance entirely for the ≈60 s surge. That is the set's share of the 141 + 152 = 293 effective vCPU of those tiers (NA 102.6, EU 73.3, APAC 87.9, MEA 11.7). Cursors freeze for ≤60 s, and the index catches up in <10 min.
- **Why a full pause.** A partial 35% shed would leave us-east at 76.8%, above V10's 75% bar.

| Lost region | Sessions | P0 joins/s | Join CPU | Survivor capacity | Before surge | Peak with surge | After shedding |
|---|---|---|---|---|---|---|---|
| us-east | 750k | 6.0k | 158 | 728 | 60.0% | 81.8% | 67.7% |
| us-central | 500k | 4.0k | 106 | 832 | 52.5% | 65.2% | 52.9% |
| us-west | 500k | 4.0k | 106 | 832 | 52.5% | 65.2% | 52.9% |
| eu-fra | 450k | 3.6k | 95 | 520 | 60.0% | 78.3% | 64.2% |
| eu-dub | 400k | 3.2k | 84 | 541 | 57.7% | 73.3% | 59.8% |
| eu-sto | 400k | 3.2k | 84 | 541 | 57.7% | 73.3% | 59.8% |
| sin | 600k | 4.8k | 127 | 624 | 60.0% | 80.3% | 66.2% |
| tyo | 500k | 4.0k | 106 | 655 | 57.1% | 73.3% | 59.8% |
| bom | 400k | 3.2k | 84 | 718 | 52.2% | 63.9% | 51.7% |
| sao | 300k | 2.4k | 63 | 1,196 (NA incl. LATAM) | 42.8% | 48.1% | n/a |
| uae (fra shown; bom 46.4 / 53.9 / 51.8) | 200k | 1.6k | 42, split | 280.8 each | 47.9% | 55.4% | 53.3% |

- **Reading the table.** The 60% figure is the *steady* post-loss level. The surge is a transient above it (≤82% before shedding, ≤68% after). The RTO does not rest on 60%. It rests on ticketed admission, which paces P0 joins at the slot rate that holds survivors at ≤82%.
- **Queueing.** At ρ = 0.82, the M/M/1 p99 wait for a 150 µs DSO service is ln(100ρ)/(1−ρ) × S = 4.41/0.18 × 0.15 ms ≈ 3.7 ms. Real service is closer to deterministic (M/D/1 halves the wait) and admission-controlled, so this is conservative. The 40 ms allowance in §5 covers GC and interference.
- **Large documents.** The ≈25k viewers of five live 5,000-session docs are P1 sessions, already inside the idle drain, and their relay trees re-form in ≤10 s.

**SLI and request-weighted availability.** The fixed SLI is (successful join + submit requests) ÷ (total join + submit requests) over a rolling 30 days, including regional outages and excluding offline local editing.
- **Success rule:** a submit succeeds iff it is acked (committed) within 5 s. A join succeeds iff the session is established and its first sync completes within 30 s of the client's first attempt (the client timeout). Anything slower is a failed request.
- **Counting rule:** each failed attempt, and each expired 30 s ticket wait, is one failed request. Tickets suppress retries, so 1 attempt/s per affected editor for T seconds is a ceiling.
- **Request mix:** submits dominate. Average 0.5M/s × 2.592M s = 1.296×10¹² submits, against 5.4×10⁹ joins, so submits are 99.6% of requests.
- **Weighting assumption:** per-DAU submit and join rates and the document mix are equal across regions, so a region's request share equals its DAU share. A region at 1.5× the average per-DAU rate scales its row by 1.5.
- **Partitions:** a partition isolating one region counts as that region's event (clients re-route to the majority side, and term fencing keeps it safe). A partition splitting the two EU survivors is the declared double fault.
- **Budget:** a regional event at traffic share s, duration T, and load factor ℓ (1 = average, 2 = peak) consumes s × T × ℓ "full-outage seconds". The budget is 0.0001 × 2,592,000 = 259.2 s.
- **Cross-check:** a us-east event at the cap, average load, fails 0.15 × 0.5M × 120 = 9.0M submits. That is 9.0M ÷ 1.30×10¹² = 6.9×10⁻⁶, or 6.9% of the budget, matching 18 s.

| Lost region | Share | Design (T = 30 s, ℓ = 1) | Cap (T = 120 s, ℓ = 1) | Worst (T = 120 s, ℓ = 2) |
|---|---|---|---|---|
| us-east | 15% | 4.5 s (1.7%) | 18.0 s (6.9%) | 36.0 s (13.9%) |
| us-central, us-west | 10% each | 3.0 s (1.2%) | 12.0 s (4.6%) | 24.0 s (9.3%) |
| tyo | 10% | 3.0 s (1.2%) | 12.0 s (4.6%) | 24.0 s (9.3%) |
| sin | 12% | 3.6 s (1.4%) | 14.4 s (5.6%) | 28.8 s (11.1%) |
| eu-fra | 9% | 2.7 s (1.0%) | 10.8 s (4.2%) | 21.6 s (8.3%) |
| eu-dub, eu-sto | 8% each | 2.4 s (0.9%) | 9.6 s (3.7%) | 19.2 s (7.4%) |
| bom | 8% | 2.4 s (0.9%) | 9.6 s (3.7%) | 19.2 s (7.4%) |
| sao | 6% | 1.8 s (0.7%) | 7.2 s (2.8%) | 14.4 s (5.6%) |
| uae | 4% | 1.2 s (0.5%) | 4.8 s (1.9%) | 9.6 s (3.7%) |

- **Remote editors (reproducible).** Remote editors are 3.3% of all editors (10% cross-region docs × 1 of 3 collaborators). In the worst case, ≤30% of them have their doc led from the failed region, so ≤1.0% of editors are affected. The stall is ≤12 s at P3, because they attach via relays, giving ≈0.12 weighted s. In v1, direct attachment means a ≤30 s reconnect, giving ≈0.30 weighted s.
- **EU docs with US editors.** These are 25% EU docs × 10% cross-region × 1/3 remote ≈ 0.83% of editors. The dub-served share (8/25) is ≈0.27% of editors, a further ≈0.08 weighted s at the design bound and ≈0.32 s at the cap (they connect directly to dub and must reconnect).
- **Idle joins.** An idle join waiting in its ticket slot counts one failed request per 30 s of waiting. For us-east, 600k idle sessions × ≈6 expired waits (mean wait ≈172 s) is 3.6M failed requests. That is 2.8×10⁻⁶, or 2.8% of the budget (7.2 s). P0 joins complete by ≈28 s, inside the 30 s timeout.
- **Worst event:** a us-east event at peak and the cap costs ≈43.4 s (36.0 + 0.24 + 7.2). So 6.0 such events exhaust the budget (259.2 ÷ 43.4 = 5.97), which §3A states as ≈6.

**Control plane and shared dependencies** (static stability: no cloud control-plane call on the request path)

| Dependency | Failure | Behavior | SLI effect |
|---|---|---|---|
| Global directory (64 GB, replicated to all 12 locations) | Service outage (30-minute drill) | Replicas are a read cache, and stale routes are fixed by an epoch-mismatch redirect. Each jurisdiction's authority is the single writer for its docIDs, so updates cannot conflict. Writes queue and replay in order on recovery, and replicas reconcile by max (docID, epoch). Docs created during the outage are routable in their home jurisdiction at once. Share and invite links carry a signed (docID, jurisdiction) hint, so cross-jurisdiction first-open works without the directory | Only an open *without* a hint (a typed docID) from another jurisdiction fails, until recovery. Degraded-mode target: ≥99.9% of joins; submits unaffected |
| KMS/HSM | Regional loss | EU runs KMS in all 3 EU regions and ROW in use, usc and sin, each with replicated key material. DSOs cache unwrapped DEKs of active docs. E2EE is client-keyed | None for active docs; new-doc key wrap +≤100 ms via another region |
| DNS | Outage | Signed client config carries IP-literal unicast endpoints for every voter-set region | None for failover |
| Anycast PoP | PoP loss | BGP reroutes. Clients fall back to unicast VIPs | ≤3 s per affected client |
| Deploy/config | Bad rollout | 2% cells, automatic rollback on SLI burn within ≤5 min | Budgeted below |
| Provider IAM/control API | Outage | Data plane keeps running; scale-out and reseed block, so RF2 lasts longer | None for serving |

**Brownouts and gray failures.** A region or node is drained when any of these holds:
- Submit success is <99% or ack p99 is >3× the 7-day baseline for 20 s, seen from ≥3 vantage points.
- **Burn-rate rule:** the error rate is >0.1% (10× the budget burn rate) for 5 min from ≥3 vantage points. A sustained 0.5% error rate therefore triggers within 5 min and costs ≈1.5 full-outage seconds.
- Node fsync p99 is >50 ms for 10 s, or packet loss is >2%.
- A gateway or relay is a 2σ outlier on error rate.

Draining means leadership transfer off the node, weight 0 for new sessions, and ≤30 s to complete. The bound from onset is therefore ≤50 s (20 s detect + 30 s drain) for fast failures and ≤5 min for slow burns. It reuses the failover path, and a drained region still serves as a follower.

**Error budget (259.2 s per 30 days)**

| Cause | Allocation | Basis |
|---|---|---|
| Region events | 60 s | One cap-bound peak us-east event (43.4 s) plus one average-load eu-fra event at the cap (13.1 s) = 56.5 s |
| Deploys/config | 40 s | 5 deploys × 2% cell × ≤5 min at full failure ≈ 30 s, plus margin |
| Control plane (directory, KMS, DNS, edge) | 40 s | Assumed 4 incidents per month at 10 full-outage seconds each. The directory outage alone costs <0.1 s (joins are 0.4% of requests × 0.1% failed × 1,800 s) |
| Gray failures/brownouts | 50 s | ≤50 s onset-to-drain at a 15% region share ≈ 7.5 s per event, so about 6 events per month |
| Provider/dependency incidents | 40 s | Assumed one incident per month degrading 20% of requests for 200 s |
| Unallocated | 29.2 s | |

### 4B. Poison-pill edit (crashes every receiving client)

Plaintext and E2EE documents get separate guarantees. A single pill and a malicious author are handled separately.

**Prevention: plaintext docs**
- Every op gets structural limits (depth, 64 KB, rate), per-doc capability gating so new op types never reach old clients, and a native apply on a copy-on-write overlay.
- High-risk ops (new types, unusual shape, ≈0.9%) plus a 0.1% random sample (1% in total, 10k ops/s) are shadow-applied in sandboxed WASM cores for all 13 builds, with a bounded anchor window and a 20 ms timeout (a hang counts as a crash). This costs ≈19.5 raw vCPU (10k × 13 × 150 µs). Memory is 13 pooled instances × 64 MB per DSO host, inside the 4 GB/vCPU.
- **Scope:** shadow-apply covers the CRDT apply core only. Crashes in rendering, layout, text shaping or asset decoding are not covered by it.
  - Embedded assets in plaintext docs are decoded in a sandbox at upload.
  - The rest is caught by the client crash marker below and by CI fuzzing of decode, apply and the render path on all 13 builds.
- Limit: bugs that need context outside the window or interactions with other ops are not prevented. Detection below covers them.

**Prevention: E2EE docs.** There is no semantic prevention, because the server sees ciphertext. The server validates the envelope only: device signature against the ACL, size, rate, dedupe, key epoch.

**Detection and containment (client)**
- **Atomic batch apply.** Each remote batch is applied to a copy-on-write overlay and swapped into the persistent replica in one local transaction only after the whole batch applies. A crash mid-batch leaves the replica at its pre-batch state, so bisection starts from clean state.
- **Marker.** Before applying, the client writes `applying:[opIDs]` with a plain write and no fsync. It survives a process crash through the OS page cache. After power loss it is simply absent and the restart counts as clean. The cost is ≈10 µs on the receive path (inside the 15 ms apply budget) and no extra flash write per batch on mobile.
- A restart with the marker set counts as a crash. After 2 crashes the client enters **safe mode**: read-only, with the batch bisected in a sandbox.
- It reports `suspect(opID, clientVer, stackHash)` (envelope-only data). Restart backoff bounds the crash loop to 2 iterations.
- **E2EE local skip:** after bisection, the device skips the culprit locally and uploads a signed `LocalSkip`, within ≤10 s. This works with no server help.

**Mitigation (server)**
- **Plaintext docs:** on a report, the DSO *replays* the op in the sandbox on the reporter's build. If it crashes there, `Quarantine{opIDs}` is appended immediately. One report suffices, and collusion cannot fake a crash that replay verifies. If replay does not reproduce, the thresholds apply: ≥3 devices from ≥2 users, or ≥10% of sessions in 60 s, or the owner or a doc admin. This gives small docs (≤3 sessions) an unattended path for reproducible pills.
- **E2EE docs (bounded path).** The server cannot verify the op, so any signed `LocalSkip` from a current member device is committed to the log (≤0.2 s) and honored as a **provisional skip** by every device that has not yet applied the op.
  - A single device in a small doc therefore protects every later and offline arrival within ≤15 s of its first crash (≤10 s local + ≤5 s upload).
  - Devices that already applied the op recover through their own bisection (≤10 s).
  - The owner sees each provisional skip and can confirm or revert it. A malicious member could suppress another member's ops this way, but members can already delete content, the skip is signed and attributable, and it is revertible.
  - Making the skip final (and the snapshot trusted) needs ≥3 devices from ≥2 users, or an owner or admin action.
- **Author containment (both modes).**
  - Per-device cross-doc limits apply: ≤20 distinct docs written per minute and ≤1,000 per hour.
  - A device's first strike drops it to 1 op/s. A *strike* is a quarantined op (plaintext, verified by replay), or a `LocalSkip` naming its op from ≥2 other devices (E2EE, attributable from the envelope device pseudonym).
  - A second strike in a different doc within 24 h write-suspends the device across the tenant until an admin clears it.
- **Quarantine semantics:** a grow-only skip-set with `appliesFrom`, treated as "never applied". Text ops anchored to characters inserted by the pill re-anchor to the pill's recorded left origin (nearest surviving ID in the skeleton). Move and property ops depending on the pill become no-ops or apply against the prior value. Canvas references (connectors, instances, comment anchors) follow the dangling-reference rule of §2. These rules are in the convergence tests.
- **Snapshot ordering:** a snapshot is `trusted` only after a 15 s hold with no suspect reports (30 s for E2EE, plus its device signatures). The hold is a heuristic. Safety rests on two other rules:
  - A Quarantine invalidates any snapshot whose range contains a quarantined op.
  - Sync rejects a snapshot whose skip-set hash lacks the quarantine set, and I5b waits 30 days before any fold deletion.
  - A stale snapshot therefore cannot reintroduce the pill.
- **Sync order:** quarantine records and provisional skips are sent *before* ops, so late and offline clients never apply the pill.
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

- **Bounds, plaintext:** design p99 ≤15 s, with a hard cap of 120 s (2 crash cycles ≤8 s, report delivery with retries ≤30 s, replay ≤1 s, commit ≤0.2 s).
- **Bounds, E2EE:** the device-local skip takes ≤10 s and the provisional doc-wide skip applies ≤15 s after the first crash. Final quarantine takes ≤120 s with ≥3 devices or an owner action. With one device and no admin the skip stays provisional but protective, and the range stays untrusted for snapshots until an owner confirms.
- **Blast radius:** one doc per pill (up to 5,000 sessions), plus author containment. A malicious author reaches ≤2 docs in plaintext (replay-verified strike in ≈7 s) and ≤20 docs in E2EE (20 docs/min × 60 s).

## 5. Capacity and cost

### Reference arithmetic

**Load.** Peak sessions are 5M (NA 1.75M, EU 1.25M, APAC 1.5M, LATAM 0.3M, MEA 0.2M).
- At 3 sessions per active doc, 1.67M docs are live at peak. 40% of them are edited (§0 assumption): 670k docs with 1.5 editors each, so 1.0M editors.
- That gives 1.0M msg/s × 300 B = 300 MB/s in. With 2 receivers each, fan-out is 2M msg/s = 600 MB/s.
- Presence from 2.5M sessions at 0.5 Hz × 60 B is 75 MB/s in and 150 MB/s out.
- Joins are 5M ÷ 1,200 s = 4.2k/s × 18.5 KB = 77.7 MB/s.
- Peak egress is 600 + 150 + 77.7 = 827.7 MB/s, and the 50% diurnal average is 413.9 MB/s.

**Offline load (5% of DAU/day).**
- That is 2.5M syncs/day averaging 100 KB up and 1 MB down.
- Egress is 28.9 MB/s average.
- CPU is 58 syncs/s at the 2× diurnal peak × 100 ms ≈ 6 raw vCPU.
- Ingest is 7.5 TB/month (+2.8% of durable bytes).
- The server keeps only a device-registry row (≈0.5 KB × 150M devices = 75 GB, inside the metadata line). The client pulls from the immutable log, so there is no per-client sync state.
- **Long-offline cohort:** 50k users with a 14-day mean episode reconnect 30/14 ≈ 2.1 times per month, about 107k reconnects. That is ≈2.1 TB up (× 20 MB) and ≈0.5 TB down (× 5 MB) per month, which is negligible.

**Routing assumption.** A cross-region doc has 1 of 3 collaborators remote, so every op crosses a region boundary exactly once. That is 10% of ops, or 100k msg/s, 30 MB/s peak, 15 MB/s average, ≈38.9 TB/month.
- v1 serves it by direct connection to the home gateway. Those bytes are already inside the 600 MB/s fan-out and priced as internet egress, so there is no separate line.
- From P3, unpinned tenants (75% of the cross-region bytes) take the relay path, which adds one backbone leg at $0.02/GB. That is priced as its own line below.

**Log growth and projection.**
- Durable ops average 105 MB/s (1.0M × 210 B × 50%), which is 9.07 TB/day or 272 TB/month raw.
- The 30-day full-fidelity tail is ≈272 TB raw (27% of the 1 PB logical total).
- Older history folds ≈10× (assumed; 5–20× in the sensitivity table), leaving +27 TB/month of retained logical history.
- The scenario's 1 PB is taken as month 0. Without expiry, logical size is 1.16 PB at month 6 and 1.33 PB at month 12 (+33%). That is +152 TB stored × $72.45 = **+$11.0k/month by month 12** (+4.1% of the model), outside the steady-state base below.
- Version thinning (daily to weekly checkpoints after 12 months) is available and not credited.

**Storage assumptions**
- **Stored fraction:** 0.465 (0.9/2.5 + 0.1 × 1.05), so 1 PB logical is 465 TB stored. E2EE ciphertext is incompressible with 5% envelope overhead.
- **Hot set:** 160M docs × 145 KB = 23.2 TB per copy, platform-wide. The 145 KB is a 125 KB uncompressed snapshot (25% × 500 KB) plus a 20 KB mean unfolded log tail and overlay index.
- **Search index:** 35% of the 225 TB logical non-E2EE current state (25% × 0.9 PB) = 78.75 TB.
- **Snapshots.** Edited-doc episodes are 0.83M average active docs × 1,440 min ÷ 34 min mean episode ≈ 35M/day, and 40% of them are edited (14M/day).
  - Hot snapshots are materialized by each voter from the log: 670k docs ÷ ≈840 s ≈ 800/s peak per voter × 2 ms × 3 voters = 4.8 raw vCPU. This causes no network traffic. SSD writes are ≈100 MB/s per replica fleet-wide, and SSD is priced by capacity.
  - Cold snapshots are written at deactivation: 14M/day × 30 = 420M/month × 125 KB × 0.465 = **24.4 TB/month**, packed into the segment objects. They are copied to the buddy and WORM copies, and the superseded ones feed the backup churn below.
- **Backup (1.15×):** one full WORM copy (465 TB) in a separate account and a distinct region of the same jurisdiction, plus two tails.
  - Fold-deleted tail segments are held 5 days past primary deletion: 126.6 TB/month × (1 − 1/10) = 114 TB/month deleted by folding, × 5/30 = 19 TB (4.1%).
  - Deletion and supersession churn is 43.2 TB/month × 35/30 = 50.4 TB (10.8%). That is 24.4 TB/month of superseded cold snapshots plus 18.8 TB/month of purged docs and replaced checkpoints (an assumed 4.0% of stored bytes/month). The lock is 35 days.
- **WORM copy:** written as one bundle per leader region per 60 s with a per-group offset index. It is write-once and never read on the normal path.
- **Restore:** WORM lags the log by ≤60 s. A restore after a double-region loss can lose acked ops newer than the last flushed bundle, so RPO ≤60 s in that case. This is outside RPO ≈ 0 and listed in §3, item 3. EU restore is ≈6.5 h.
- **WAL NVMe:** flush within 60 s or 384 KB per group, then 1 h retention for follower catch-up. Capacity is 210 MB/s × 3,600 s = 756 GB per replica × 3 replicas × 2 zone mirrors = 4.5 TB, ÷0.7 = 6.5 TB.
- **Memory:** priced at 4 GB/vCPU (13.2 TB provisioned). The working set is ≈0.2 TB (sessions 40 GB, active DSO state ≈84 GB × 1.3, WASM pools ≈1 GB per DSO host), so the allowance has 60× headroom.
- **Request pricing** is not in the reference list. It is assumed at $5 per million PUT and $0.40 per million GET.
- **Metadata (2 TB, derived):**

| Component | Basis | TB |
|---|---|---|
| Doc rows | 2B × 300 B | 0.60 |
| ACL and membership | 2B × 3 × 100 B | 0.60 |
| User→docs reverse index | 6B entries × 40 B | 0.24 |
| Device registry | 150M × 0.5 KB | 0.08 |
| Segment manifests | ≈12 months of bundles | 0.30 |
| Subtotal × 1.1 index/version overhead | 1.82 × 1.1 | 2.0 |

**Raw peak vCPU**

| Component | Derivation | vCPU |
|---|---|---|
| Gateways and relays | Keepalive 5M × 1 Hz × 10 µs = 50; connect 4.2k/s × 2 ms = 8; edit in 1M × 30 µs = 30; edit out 2M × 30 µs = 60; presence 3.75M/s × 25 µs = 94 | 242 |
| DSO | 1M × 150 µs | 150 |
| Raft follower apply | 2 × 1M × 40 µs | 80 |
| Join / ACL / API | Joins 4.2k/s × 17.6 ms = 74; offline syncs 6; ACL/API/auth 1.5k/s × 20 ms = 30 | 110 |
| Snapshot / compaction | Snapshots 4.8; zstd 0.7; fold 105 ÷ 50 MB/s = 2.1; manifest/GC/verify 2.7 | 10.3 |
| Shadow-apply (plaintext) | 10k ops/s × 13 builds × 150 µs | 19.5 |
| Search index + query (provisional until P3 benchmarks) | Index 670k docs ÷ 30 s = 22.3k/s × 3 ms = 67; merge 34; queries 8.3k/s × 10 ms = 83 | 184 |
| Auth, KMS, directory serving, telemetry agents | Provisional | 36 |
| **Total** | 831.8, rounded up | **832** |

**Provisioning:** 832 × 1.5 (platform overhead) ÷ 0.40 = 3,120, plus the 187.2 skew uplift from §4A = **3,307 vCPU** (37.7% steady). The N+1 premium over a 70%-utilization design is 3,307 − 1,783 = 1,524 vCPU = $43.9k/month. That is the $38.5k uniform-basis figure plus the $5.4k skew uplift, and the EU share is $10.2k. These are the figures used in §3.

### Monthly cost (720 h; vCPU-month = $28.8; every line derived)

| Line | Derivation | $/month |
|---|---|---|
| Compute | 3,307.2 × $28.8 | 95,247 |
| EU pilot-light spare | 75 vCPU × $28.8 = 2,160 + 8.3 TB NVMe × $100 = 830 | 2,990 |
| Internet egress | (413.9 realtime + 28.9 offline sync) = 442.8 MB/s × 2.592M s = 1,147.7 TB × $80/TB | 91,819 |
| Inter-region | Raft: 272.2 TB × 2 remote followers = 544.3 (uncompressed). Segment copy to buddy: 272.2 × 0.465 = 126.6. WORM copy: 126.6. Cold snapshots to buddy and WORM: 2 × 24.4 = 48.8. Total 846.3 TB × $20/TB | 16,926 |
| Relay backbone leg (P3, unpinned tenants) | 75% × 38.9 TB × $20/TB | 584 |
| Hot SSD | 23.2 TB × RF3 = 69.6 ÷ 0.7 = 99.4 TB × $100 | 9,943 |
| WAL NVMe | 6.5 TB × $100 | 649 |
| Search index (primary and buddy, RF2) | 25% on SSD at RF2: 78.75 × 0.25 × 2 ÷ 0.7 = 56.25 TB × $100 = 5,625; 75% in object at 2 copies: 118.1 TB × $23 = 2,717. Not backed up: derived data, rebuilt in hours from restored content | 8,342 |
| Metadata / ACL / directory / reverse index | 2 TB × RF3 ÷ 0.7 = 8.57 TB × $100 = 857; directory 64 GB × 12 locations = 0.77 TB × $100 = 77 | 934 |
| Object (primary + buddy) | 465 TB × 2 × $23 | 21,390 |
| Backup (WORM, separate domain) | 465 TB × 1.15 × $23 | 12,299 |
| Object requests | PUT: 16,384 groups × 1,440 flushes/day × 30 = 708M per copy, ×1.15 for size-cap flushes at busy hours = 814M, × 2 copies = 1.63B × $5/M = 8,140; WORM bundles 0.5M PUTs = 2; GET 100M × $0.40/M = 40 | 8,183 |
| E2EE single-device fold lag | 10% E2EE × 272 TB/month × 1.05 = 28.6 TB/month; 50% assumed single-device = 14.3 TB/month held 35 extra days = 16.7 TB × $72.45 | 1,210 |
| **Total** | | **270,516** |

**Normalized: $270.5k ÷ 50M DAU = $0.0054 per DAU per month.**
- Cost drivers are compute and spare 36.3%, egress 33.9%, storage 20.2%, inter-region 6.5%, and object requests 3.0%.
- The E2EE storage effect (+65 TB stored, +$5.4k including segment copies) is already inside the object, backup, and inter-region lines via the 0.465 stored fraction.
- Not modeled, and covered by the platform-services buffer below: observability, WAF/DDoS, CI/staging, support, KMS operations.

**Optional second-provider EU-resident WORM copy** (provider-wide failure; restore RTO in hours, outside the SLO): storage 465 TB × 1.041 × $23 = $11,134, a stream of 151.0 TB (126.6 segments + 24.4 snapshots) × $80 = $12,080, and sampled restore verification of 0.75% (3.5 TB) × $80 = $280. That is **+$23.5k/month**, not in the base total.

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

**Region-homed docs with editors spread over metros.** The leader sits at the minimax-RTT metro. Leg = 26 + RTT/2, loss round = RTT to the leader (50 for a same-metro leg), pair = sender leg + receiver leg + 50 ms server-side.

| Home (leader) | Editor metros (RTT to leader, ms) | Worst pair, no loss | One loss round |
|---|---|---|---|
| LATAM (sao), MEA (uae) | single metro | 102 | 152 |
| EU (fra) | dub, sto (30) | 132 | 162 |
| NA (usc) | use (28), usw (38) | 140 | 178 |
| APAC (sin) | tyo (70), bom (58) | **172** | **242** |

**Foreign-homed pairs on an EU-homed doc** (one loss round adds the larger leg's recovery R). Leg values (one-way p99 L / loss round R): EU 26/50, us-east 70/80, us-west 100/160, sao 115/190, uae 80/120, bom 100/130, sin 115/165, tyo 145/251. The pair formula uses a symmetric 26 ms EU leg, so it differs from the hop table by ≤1 ms.

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

- **Cross-region:** the worst pair is 591 ms, or 631 ms with a +40 ms allowance for p99 queueing at 82–85% CPU. The margin to 800 ms is 169–209 ms.
- **In-region, region-homed docs:** the worst no-loss pair is 172 ms (APAC tyo ↔ tyo), which meets 200 ms with 28 ms of margin. With a stacked loss round, APAC multi-metro reaches 242 ms, NA 178 and EU 162. APAC multi-metro is therefore a listed exception in §3, item 5.
- **Exposure to 200 ms misses.** Foreign-homed same-region pairs (190–340 ms no loss) miss it.
  - Assume 25% of cross-region docs have ≥2 remote collaborators in one region. That is 10% × 25% = 2.5% of docs.
  - In such a doc, 2 of 3 senders are remote, and each has 1 of 2 receivers in the same remote region, so 1/3 of its deliveries are exposed. That is 2.5% × 1/3 ≈ **0.8% of all deliveries**.
  - The stacked-loss tail on APAC multi-metro docs (30% × 15% × 1% ≈ 0.05%), failover windows (120 s ÷ 2.59M s = 0.005%) and v1 LATAM add ≤0.05%.
  - About 99.2% of deliveries are therefore in classes that meet <200 ms on the no-loss budget.
- LATAM clients served from us-east during a sao loss: 152 (in-region with a loss round) + 115 ms RTT ≈ 267 ms.

### Workload distributions (assumed and modeled)

| Missing distribution | Treatment |
|---|---|
| Doc popularity / members per doc | Zipf, exponent 2.2, untruncated: mean 3.75 members (the reference's 3 is used for the base model), p99.9 ≈ 190, P(≥5,000) = 2×10⁻⁵ (≈40k docs). The product caps live sessions at 5,000. About 200 of the 40k are live at a given peak (0.5%, the active share of hot docs). Assume 2.5% of those, 5 docs, are at the cap together (an all-hands event). That adds ≤89 MB/s under policy (c) (5 × 17.85); the stress case is 10 docs |
| Maximum size | 25 MB document state and 64 KB op, paste chunked. Binary assets are separate content-addressed objects of ≤25 MB. These are product limits, tested against Docs-class and Figma-class workloads in P3 |
| Log growth | Projected above (+33% logical by month 12) |
| Reconnect bursts | Failover (§4A), release, mobile-carrier outage, Monday 9 am local. Ticketed admission, jitter, IP-literal fallback |
| Offline size and duration | Modeled above |
| E2EE share | 10% of docs and bytes |
| Hot-doc sequencing load | Per-doc cap of 500 msg/s and 25% of group append capacity, with online split to a single-doc group (§1) |

### Sensitivity (change in $/month vs base)

| Assumption | Low | High |
|---|---|---|
| Per-op CPU (340 raw vCPU on the edit path: gateway in/out 90, DSO 150, follower apply 80, shadow 20; ×0.5 / ×2; $114.5 per raw vCPU-month = 3.975 provisioned × $28.8) | −19.5k | +38.9k |
| Presence share of sessions (25% / 100%) | −13.2k | +26.3k |
| Search CPU (provisional 184 raw; ×0.5 / ×2) | −10.5k | +21.1k |
| Offline payload per sync (0.25 MB / 4 MB) | −4.5k | +18.0k |
| Compression (3.0:1 / 2.0:1) | −5.1k | +7.7k |
| Hot fraction (5% / 15%) | −3.7k | +8.7k |
| Edited docs (670k clustered vs 815k independent placement) | 0 | +2.6k |
| Memory (4 → 8 GB/vCPU at an assumed $1/GB-month DRAM; not needed at the 0.2 TB working set) | 0 | +13.2k |
| Cross-region share (5% / 20%) | ≈0 in v1 (direct routing, bytes already in the egress line). P3 relay leg: −0.3k / +0.6k | |

- **Fold ratio** (added by month 12 on top of the base, absolute, not a delta): 20× adds $5.5k, 10× adds $11.0k, and 5× adds $22.0k.
- Per-op CPU, presence and search CPU dominate the uncertainty, so they are re-measured in P3.

### Heavy-tail scenario 1: 5,000-session document (one byte basis: 300 B wire, 60 B presence)

**(a) Naive, uncapped (5,000 editors, 20% typing, 50% sending presence)**
- Edits: 1,000 typists × 300 B = 300 KB/s per receiver × 5,000 = **1.5 GB/s**, which is 2.5× the platform's entire 600 MB/s edit fan-out peak.
- Presence: 2,500 senders × 0.5 Hz × 60 B = 75 KB/s per receiver × 5,000 = 375 MB/s.
- Total 1.875 GB/s ≈ **$540/hour** for one doc.

**(b) Uncapped, with the mitigation stack**
- Viewport-scoped fan-out (≈5% of ops per receiver) is delivered on the editor tick: 50 msg/s × 300 B = 15 KB/s × 5,000 = 75 MB/s.
- Presence is capped at the 50 nearest cursors: 1.5 KB/s × 5,000 = 7.5 MB/s.
- Total 82.5 MB/s ≈ $23.8/hour, 23× cheaper. Off-viewport ops are applied on scroll (≤300 ms after scroll), which is listed in §3 as unmet item 6.

**(c) Policy: 100 editor slots, 4,900 viewers (the P3 serving policy, plaintext docs only)**
- Editors: 100 slots, all typing as a stress case (the 20% rule would give 20 typists and 0.6 MB/s): 100 × 300 B × 100 receivers = 3.0 MB/s.
- Viewers: viewport ops 5 msg/s × 300 B = 1.5 KB/s × 4,900 = 7.35 MB/s.
- Presence 7.5 MB/s.
- Total 17.85 MB/s (143 Mbps) ≈ **$5.1/hour**, about 105× cheaper than (a).
- A one-level relay tree of 50 relays × 100 clients keeps DSO egress at 1.5 MB/s (full edit stream to each relay) plus ≈0.3 MB/s presence aggregation, ≈1.8 MB/s (14 Mbps).
- The tree's only extra hop is the DSO → relay hop already in the budget (+3 ms). In-viewport p99 stays ≈105 ms, under 200 ms.
- A 5,000-client meeting-start burst at 500 joins/s serves a 100 KB viewport-first chunk each: 50 MB/s for 10 s, with the rest lazy.
- This is the P3 state. v1 serves ≤1,100 sessions per doc (§6).
- **Relay compute:** 50 relays × 100 clients forward ≈1.5 KB/s each. That is ≈75 MB/s fleet-wide for five docs, <1 raw vCPU at 30 µs per message, inside the gateway/relay line.
- **Hot-shard isolation:** the 500 msg/s per-doc cap (§1) bounds the doc's append rate to ≈150 KB/s, below 25% of the group's capacity.

**(d) E2EE variant (cap 100 editors + 900 viewers).** There is no viewport scoping, so all 1,000 sessions get every op: 100 typists × 300 B × 1,000 = 30 MB/s, plus presence 500 senders × 0.5 Hz × 60 B × 1,000 = 15 MB/s. The total is 45 MB/s ≈ $13/hour. A ciphertext-forwarding relay tree (10 relays × 100) keeps DSO egress at 0.3 MB/s. E2EE docs do not reach 5,000 sessions (§3, item 14).

**Combined with the base.** Five (c) docs add 89 MB/s at the event peak. Even sustained 24×7 at the 50% diurnal average, that is 44.6 MB/s × 2.592M s × $80 = $9.2k/month. This is a separate increment, inside the 10% buffer below.

### Heavy-tail scenario 2: reconnect storm

The surge table in §4A covers all 11 regions on unequal capacity. Peak survivor CPU is ≤82% before shedding and ≤68% after. The in-region queueing effect at those levels is ≈3.7 ms, inside the 40 ms allowance. The five large docs' viewers (≈25k sessions) are P1 sessions already inside the idle drain.

### Budget ceiling

**$0.0090 per DAU per month ($450k/month), 1.66× the model.** The buffers compound, since each covers a different uncertainty class: 1.20 × 1.10 × 1.25 = 1.65, and $270.5k × 1.65 = $446.4k, rounded up.
- +20% for unmodeled platform services (observability, WAF/DDoS, CI/staging, support).
- +10% for drills and heavy-tail surges ($9.2k sustained heavy-tail increment plus drill traffic).
- +25% for price and estimate error. It covers the two largest high-side sensitivities together (+$38.9k per-op CPU and +$26.3k presence = $65.2k = 24%).
- If the second-provider copy is enabled (+$23.5k), the buffers shrink to 53% ((450 − 294.0) ÷ 294.0), and the ceiling still holds.

**Unit economics (an assumption, since the prompt gives no revenue model).** With DAU/MAU = 30% there are 167M MAU. At 3% paid conversion of MAU × $10/month, revenue is $50M/month ($1.00 per DAU-month). The ceiling is then 0.9% of revenue, and the modeled cost is 0.54%. A 10%-of-revenue infrastructure cap would allow $0.10 per DAU-month, so the ceiling is set by estimate error, not by affordability. The defense does not depend on the revenue figures.

If we breach the ceiling, the levers are:
1. Tighter coalescing and viewport sampling. A 10% egress cut saves $9.2k.
2. Committed-use or private-interconnect discounts on compute and egress (69% of spend; not in the reference prices).
3. Compressing Raft replication in transit (−$5.8k at the 0.465 ratio).

**Not a lever:** raising steady utilization above the §4A per-set sizing. At 50%, survivors would pass 75% before the surge. A us-east loss would then exceed 100% under the surge (a eu-dub loss would peak at ≈95%).

## 6. Cuts and roadmap

**v1 is the end of P1 (month 6): NA and EU tenants only.** The end-state design in §§0–5 meets the fixed requirements except the items in §3 "cannot be met". v1 intentionally does not, and is not offered as compliant. Between P2 and P3 the platform meets offline and compatibility but still serves NA and EU only.

| Cut | Restored in | Requirement affected until then |
|---|---|---|
| APAC and MEA not served (34% of DAU); LATAM docs home in the NA voter set; no sao/uae compute or voters | P3 (month 12) | Global availability. LATAM in-region SLO unmet (≈270 ms) |
| Offline 7 days; client window 6 months | P2 (month 9) | 30-day offline and 12-month compatibility **unmet** |
| Cap of 100 editors and 1,000 viewers (≤1,100 sessions per doc); no relays for unpinned docs | P3 | Heavy-doc analysis in §5 is the P3 state |
| No E2EE (search S1, tenant recovery) | P4 | E2EE requirement unmet |
| No S2 SSE search | P5 (optional) | None; S1 meets "E2EE with search" |
| No HYOK (Shamir recovery key only) | P5 | None |
| Canvas limited to frames, shapes, and LWW properties | P3 | Figma-class scope |

**LATAM migration (P3).** Docs move from the NA set {use, usc, usw} to the LATAM set {sao, use, usc} per group by a single-member Raft change. Add sao as a learner, run joint consensus replacing usw, transfer leadership to sao in a low-activity window, then bump the directory epoch. There is no downtime and no acked-op loss, because ACL positions and E2EE key epochs travel in the log.

| Phase | Scope | Measurable exit criterion |
|---|---|---|
| P0 (0–3 mo) | NA only: text CRDT, DSO, 3-region Raft, hot and cold snapshots, segments to two regions, WAL GC | p99 in-region ≤200 ms at 100k concurrent sessions in load test; V1 passes for I1–I4 and I5a (10⁶ simulation seeds, 0 violations); ≥100 staging region-kill drills with 0 lost acked ops and max RTO ≤120 s |
| P1 (3–6) | EU jurisdiction, residency enforcement, EU/US collaboration, WORM copy, eu-par and replacement-voter procedure. **v1 ships here** | ≥100 staging EU region-kill drills plus 3 production drills, max RTO ≤120 s; 0 honeytoken hits in non-EU storage and logs for 30 days; EU hot reseed measured at ≥5 GB/s (5.8 TB ≤30 min including provisioning); a membership-change drill promoting eu-par with 0 lost acked ops |
| P2 (6–9) | 30-day offline, compatibility CI over the full 12-month window, poison-pill quarantine, trusted snapshots and fold GC (I5b), directory-kill drill | 10⁶ randomized 30-day-offline merges converge bit-identically; compatibility matrix of 13 frozen builds cut from monthly protocol tags (N-12…N, with an accelerated schema-change corpus) green; V1 extended to I5b with 0 violations; V9 plaintext criteria pass; directory-outage drill keeps ≥99.9% of joins |
| P3 (9–12) | APAC, LATAM, MEA; editor slots, viewer mode, relays for unpinned docs; LATAM migration | V4 passes over 30 days of probes for every §5 pair; reconnect drills (400k eu-dub, 750k us-east sessions) with max first submit ≤120 s across all affected sessions; 5,000-session replay under policy (c) egress ≤22 MB/s |
| P4 (12–15) | E2EE with client-side search, single-device delayed trust, tenant recovery, E2EE poison path | Third-party crypto audit with 0 critical findings; 3-of-5 recovery restores 100% of test DEK epochs including after key rotation; offline-across-rotation test 100% rule-conformant; no E2EE plaintext markers server-side; single-device docs reach trusted snapshots without blocking WAL GC |
| P5 (15–18) | Availability and cost hardening, optional S2, HYOK | 99.99% SLI in every rolling 30-day window across 90 consecutive days including ≥1 region drill; measured $/DAU ≤$0.0090 at scaled extrapolation; a real ≥12-month-old client build (the v1 build from month 6) passes V8 |

## 7. Verification plan

Statistics only bound failure rates and never prove them. The safety guarantees rest on the model-checked invariants (V1), and the tests check the implementation against them.

| ID | Area | Method | Pass criterion | Covers |
|---|---|---|---|---|
| V1 | Commit, fencing, GC safety, membership | TLA+ spec of I1–I4, I5a, I5b and the joint-consensus replacement of a dead voter, model-checked on a 3-voter region-kill model. The model includes shared physical regions (use and usc in two sets), the 2-zone WAL mirror, a separate WORM account, manifest CAS and term-fenced conditional writes. Deterministic simulation (10⁶ seeds with region kills, partitions, clock drift ≤200 ppm, process pauses longer than the lease, fsync lies), 10⁵ simulated elections, plus trace validation of the implementation against the spec. Gated by phase: I1–I4 and I5a in P0, membership in P1, I5b in P2 | 0 invariant violations; 0 lost acked ops; 0 dual-leader commits; max election ≤12 s | RPO ≈ 0 |
| V2 | RPO under region kill | Jepsen-style at ≥1.2× per-group reference peak, plus an overload run at 1.5×. Kill a random region mid-load and audit acked opIDs against the recovered log | 0 missing in ≥300 staging kills plus 3 production kills. The rule-of-three bound (≤1% per kill at 95%) is reported as supplementary evidence only | RPO ≈ 0 |
| V3 | RTO (hard maximum) | ≥100 staging drills and quarterly production drills; ≥5,000 distributed probers including cellular and CGNAT. Server-side first-submit logs of **every affected session** in each drill. Fault variants: admission queue at 150% load, stale directory, DNS blocked, one slow follower, NAT rebind. Idle→edit promotion test | **Max over all probers and all affected sessions ≤120 s** (alert at 45 s); idle→edit first submit ≤3 s after keystroke. The bound is a protocol property (deterministic timers and ticket schedule, checked in V1's timing model), and drills confirm it. No percentile substitutes | RTO ≤2 min |
| V4 | Latency | ≥1,000 probers per region pair (every pair in §5, including multi-metro region-homed pairs), PTP/GPS-synced stamps, load at 1.5×, also in failover state | Rolling 24 h p99: in-region <200 ms for region-homed docs (no-loss budget); every cross-region pair <800 ms. The listed exception classes are *not* a pass of the 200 ms requirement. They are checked for conformance to their §5 values, and the 0.8% exposure share is measured | p99 requirements |
| V5 | CRDT convergence | Property tests and nightly fuzz: 10⁹ permuted interleavings including a virtual-clock 30-day offline, quarantine skip-sets and re-anchoring, schema versions N-12…N. LWW test: an offline client with 10⁵ ops versus one online edit of the same property, in plaintext and E2EE. Move-tree benchmark (10 µs/move assumption) | 0 divergent state hashes; 10⁶ randomized 30-day-offline merges bit-identical; the commit-position LWW winner is identical on every replica; move-tree p99 ≤0.2 s at 20k moves | Offline convergence |
| V6 | Residency | OPA policy-as-code on IaC; daily honeytoken docs searched in non-EU storage, logs, backups, heap dumps, and edge captures; BGP announcements for EU-pinned prefixes from EU PoPs only, confirmed by non-EU probes; egress allow-list audit; quarterly external audit; strict-policy managed devices checked for empty caches after session end | 0 hits; 0 violations. The device-boundary exception is excluded from "hits" and tested only for policy-flag conformance | EU residency |
| V7 | E2EE | Marker strings; crypto audit; recovery drill; offline-across-rotation test (client offline 30 days across ≥3 epoch rotations and one removal); S1 offline search; single-device doc through the 35-day delayed-trust path, including replay re-verification of a bad checkpoint | 0 plaintext hits server-side; 0 critical findings; 100% DEK recovery; 100% rule conformance; single-device docs fold and WAL-truncate without blocking | E2EE |
| V8 | Compatibility | 13 builds (N-12…N) × servers N and N+1 on golden logs; live mixed-version docs where new op types are attempted; rolling-deploy test. A separate emergency-expiry test: the force-expired build is refused with an upgrade prompt, and all other builds are unaffected | Server log and segments round-trip byte-identical (hash compare). Client unknown fields preserved semantically (canonical decode equal), 100%. Old clients never crash and show fallbacks. Deploy error ≤0.001%. The emergency exception is verified as the *only* path that breaks the window | 12-month clients |
| V9 | Poison pill | 1,000 injected pills per class. Large plaintext docs; small docs (≤3 sessions) with reproducible pills; E2EE docs including single-device docs; partial-batch crash followed by safe-mode bisection; a malicious author spraying pills across docs; 10⁴ pill/snapshot interleavings; 50% report loss and delay | Plaintext max ≤120 s (design p99 ≤15 s); small reproducible docs ≤60 s unattended; E2EE local skip ≤10 s (100%) and provisional doc-wide skip ≤15 s, final ≤120 s with ≥3 devices; replica identical to its pre-batch state after a mid-batch crash; author suspended at the 2nd strike (≤2 docs plaintext, ≤20 E2EE); ≤2 crash-loop iterations; 0 untrusted snapshots used in recovery | Poison-pill analysis |
| V10 | Heavy tail | Replay the 5,000-session doc and storms (us-east 750k, sin 600k, eu-fra 450k sessions), with scenario-specific criteria. (a) is diagnostic only (≈1.9 GB/s expected). (b) doc egress ≤90 MB/s. (c) doc egress ≤22 MB/s (176 Mbps) and DSO egress ≤2.2 MB/s. (d) E2EE variant ≤50 MB/s | In-viewport editor p99 <200 ms; off-viewport apply ≤300 ms after scroll; storm CPU ≤90% before shedding and ≤75% after (model: ≤82% / ≤68%); max first submit ≤120 s | Heavy-tail model |
| V11 | Control plane and gray failure | Kill the global directory for 30 min while creating docs and opening them cross-jurisdiction, with and without invite hints; block DNS; drop one KMS region; inject 200 ms fsync stalls and 5% loss in one region; a slow burn at 0.5% errors | Join success ≥99.9% with the directory down; docs created during the outage routable at home immediately and reachable cross-jurisdiction via hints, with queued writes reconciled after recovery; 0 failed submits on existing sessions; drain ≤50 s (fast) or ≤5 min (slow burn); 0 acked-op loss | Failure analysis, §4A |
| V12 | Availability | Prober plus server-side SLI per the fixed definition: submit acked ≤5 s, join complete ≤30 s, expired ticket waits counted as failures. Includes deferred joins under 150% load, client timeouts, and monthly chaos (zone, region, partition), drills counted | ≥99.99% in every rolling 30-day window across 90 consecutive days; error-budget burn tracked per the §4A table. Presented as empirical SLO evidence, not a universal guarantee | 99.99% |
| V13 | Offline load | Replay 5% of DAU reconnecting at 100 KB up and 1 MB down, plus the long-offline cohort (20 MB up), key-epoch rotations, removals and owner-review inbox load | Egress and CPU within model +20% | Capacity model |
| V14 | Cost | Quarterly bill against the model, with sensitivity re-run. Provisional inputs (search queries, Zipf tail, E2EE size mix, backup churn) are re-measured first | Unit cost within ±15% of model and at or under the ceiling | Budget ceiling |
| V15 | Restore from WORM | Quarterly restore of a 1% sample of each jurisdiction from the WORM copy into an isolated cell, after simulated double-region loss | Restored docs hash-equal to the last flushed log; acked ops lost ≤ those newer than the last 60 s bundle; EU restore rate ≥5 GB/s measured (projects to ≤6.5 h) | Restore RPO, §3 item 3 |

**Traceability.** Each row of the requirement-status table maps to at least one V-row:
- Latency maps to V4 and V10.
- Offline maps to V5 and V13.
- Residency maps to V6.
- E2EE maps to V7.
- Availability, RPO and RTO map to V1, V2, V3, V11, V12 and V15.
- 12-month clients map to V8.
- The poison-pill analysis maps to V9.
- Cost maps to V14.
 

``` 


--- 


# AI Response with Validation Layer ( ENTERPRISE VERSION ) 

``` 

# Global Collaborative Editor: Design v4

## Summary and requirement status

**Design in brief.** Five product regions run on 11 active cloud regions, plus one EU pilot-light spare.
- Each shard group is a 3-voter Raft group with voters in distinct regions. The Raft log is both the commit point and the sequencer, and a per-document Session Owner (DSO) sits with the group's leader. Leaders are a property of the group, so docs are placed into leader-affinity classes at creation (§0, §1).
- Replicas are CRDTs: Fugue text, plus commit-ordered LWW and a move-tree for canvas. This is why 30-day offline merges converge.
- Two jurisdictions (EU, ROW) hold content, and EU content is never stored, processed or cached outside the EU by provider infrastructure. Whether a non-EU end-user device may receive it is a separate, explicit policy choice (§0: Strict vs Infrastructure residency).
- E2EE is a per-tenant policy under which the server only sequences envelopes.
- Modeled cost is $0.0054 per DAU-month against a $0.0090 planning ceiling.

**Status legend.**
- **Fully met:** holds by construction under the stated failure model, with no listed exception.
- **Conditionally met:** holds under explicit preconditions.
- **Partially met:** holds for a stated population, with named exceptions.
- **Fundamentally incompatible:** cannot hold together with another fixed requirement.

| Requirement | Status (end state, P5) | Basis and exceptions | v1 (end of P1, month 6) | Verified by |
|---|---|---|---|---|
| p99 <200 ms in-region | **Partially met** | Aggregate normal-state in-region p99 is modeled <200 ms: ≈99.15% of deliveries are in classes under 200 ms, and the worst region-homed pair is 172 ms. Per-class p99 exceeds 200 ms for foreign-homed same-region pairs, LATAM/MEA during failover, APAC multi-metro with a loss round, and viewer-tier off-viewport applies (§3 items 5–6; ≈0.85% of deliveries excluding the viewer tier) | NA and EU only. LATAM ≈270 ms (homed in NA). APAC and MEA not served | V4, V10 |
| p99 <800 ms cross-region | **Conditionally met** | Worst pair of any kind is 591 ms (tyo↔tyo on an EU-homed doc). Worst pair with clients in different regions is 561 ms (tyo↔sao). With a +40 ms queueing allowance: 631 and 601 ms. A per-hop budget, not a proof | Met for NA/EU pairs | V4 |
| Offline ≤30 days, deterministic convergence | **Conditionally met** | Convergence is guaranteed for ≤30 days. Acceptance of every legitimate offline edit is not (item 11), and two concurrent LWW values cannot both survive | Unmet: 7 days | V5, V13 |
| EU content never leaves the EU | **Fundamentally incompatible** with EU/US collaboration under the literal reading | Two named policies (§0). **Strict residency (SR)** satisfies the literal text and forbids non-EU access. **Infrastructure residency (IR, default)** satisfies collaboration and does not satisfy the literal text. IR is never presented as SR | Same | V6 |
| Optional per-tenant E2EE with search and admin recovery | **Conditionally met** under a scoped interpretation | "Search" means client-side search over content synced to the user's devices (S1). Server-side workspace-wide discovery of unsynced E2EE docs is unsupported (item 8). Recovery is tenant-held. E2EE docs are capped at 100 editors + 900 viewers | Unmet (P4) | V7 |
| 99.99%, RPO ≈ 0, RTO ≤ 2 min | **Conditionally met** | Single-region failure under the stated failure model (§4A). The RTO clock starts at the failure and the affected cohort is defined there. A conditional SLO, not an unconditional guarantee | NA and EU tenants only | V1–V3, V11, V12, V15 |
| 12-month clients, zero downtime | **Conditionally met** | Met except emergency security expiry (item 10). The bootstrap handshake is frozen for the window (§1) | Unmet: 6-month window | V8 |

**Two kinds of guarantee are kept apart.**
- *Architectural guarantees* hold by protocol under the failure model: RPO ≈ 0, fencing, convergence, residency enforcement, and the RTO bound as a protocol property.
- *Operational SLO targets* are measured: latency percentiles, availability, cost.
- Each row above and each matrix row in §3 says which it is.

## 0. Fixed assumptions and definitions

**Failure domains** (the last column lists what each domain does *not* isolate)

| Domain | Definition | Independent of | Shares fate with |
|---|---|---|---|
| Zone | One AZ (≥3 per cloud region) | Power, cooling, rack network of other zones | Regional control plane |
| Region | One cloud region (metro) | Compute, storage, power, network of other regions | Provider IAM and DNS, our binaries and config (hence cell-based staged rollout), global directory, anycast control |
| Jurisdiction | Legal/admin domain with its own cloud accounts, KMS/HSM root, deploy pipeline, break-glass | Other jurisdictions' accounts, keys, operators | The cloud provider, public DNS/BGP, the signed-build trust root, the global directory (routing metadata only) |
| Backup domain | Separate account, compliance-mode object lock, separate KMS key, in a region other than primary and buddy | Region or zone loss, credential compromise, operator error | For EU content, the EU jurisdiction's fate (backups must stay in the EU). In v1, also the provider's |

**Privilege separation (what "independent" means operationally).**
- Deploy credentials are distinct and short-lived per jurisdiction and per region.
- The backup account has its own organization-level administrators under a two-person rule.
- It uses compliance-mode object lock, so not even root can shorten it. No deploy or automation role can delete it or change its policy.
- Break-glass access is quorum-gated and logged. For the EU it is held by the EU-established operating entity.
- What remains shared is stated in the table: the provider's organization-level control plane and the signed-build trust root.

**Why "region" means a cloud region.** The failure unit is the cloud region, the largest unit with independent power, network and control plane that providers actually lose.
- Product regions (NA, EU, APAC, LATAM, MEA) are traffic, capacity and placement groupings. Capacity is planned per voter set, but every failure analysis uses the cloud region as the fault boundary.
- Losing a whole product region needs two or more simultaneous cloud-region failures. That is the declared out-of-scope tier, and it is priced in §5 rather than assumed away.

**Topology.** There are 5 product regions on 11 active cloud regions, plus one pilot-light spare (12 deployed locations). Traffic shares are in parentheses.
- NA: us-east (15), us-central (10), us-west (10)
- EU: eu-fra (9), eu-dub (8), eu-sto (8). eu-par (France) is a pilot-light spare and not a voter until promoted (§4A).
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

- In v1, LATAM docs home in the NA set. APAC and MEA are not served until P3, when their sets are deployed (see §6).
- A physical region can sit in two sets (use and usc in NA and LATAM, bom in APAC and MEA, fra in EU and MEA). Losing it removes one voter from each set, but only groups *led* from it re-elect. Availability is therefore counted by serving region (§4A), not by voter-set membership.
- Non-EU content may sit in EU regions. The reverse is forbidden.
- **Leader placement is per group, not per doc.**
  - A Raft group has one leader, and all docs in a group share it.
  - Each voter set's groups are divided into leader-affinity classes, one per eligible region and sized by that region's traffic share (EU 9:8:8 over fra:dub:sto).
  - A doc is placed at creation in a group of the class whose leader region minimizes the maximum RTT to the tenant's expected editor metros. The directory records the group.
  - Latency in §5 is therefore evaluated per placement class.
  - An EU-pinned doc is always placed in an EU voter set, so its leader is fra, dub or sto even if its median editor is in the US.
  - Re-homing a doc whose editors have drifted is a P3 handoff (§1). Until then, drifted docs fall into the exception classes of §3 item 5.

**Jurisdictions.** There are two.
- **EU** is residency-bound, with its own accounts, KMS/HSM root and deploy pipeline.
- **ROW** covers all other tenants. The NA, LATAM, APAC and MEA voter sets run in ROW accounts, with a ROW KMS root replicated in use, usc and sin. ROW regions without a local KMS replica reach the nearest one (§4A control-plane table).
- ROW makes no residency promise. Its voter-set regions exist for latency and survivability, so replicas of a ROW tenant may sit in tyo, bom or fra. fra hosts a ROW-account MEA voter, in a separate account and KMS from the EU voter.
- Country-specific residency (India, Brazil, and so on) is not offered. Each would need its own jurisdiction built from the EU template, which single-region countries cannot satisfy (§3, item 15).
- ROW has no pilot-light spare, because nothing prevents provisioning a replacement region anywhere in ROW (≈10 min). The EU needs eu-par because its boundary forbids that.

**EU "region" and boundary.** An EU region is one cloud region in an EU member state (fra DE, dub IE, sto SE, spare par FR). The EU boundary is the territory of the 27 member states plus an EU-established operating entity with an EU-resident HSM root. EEA-only states are not treated as EU. Every acked EU op is on ≥2 of 3 EU voters, so any single EU region loss leaves 2 voters (a quorum) and at least one copy of every acked op, all inside the EU.

**Out of scope (stated as unmet):**
- Loss of two regions in one voter set, including a correlated loss of a whole product region. The next tier is priced as a lower bound in §5 and not adopted.
- Loss of the whole EU jurisdiction.
- Provider-wide failure. This is unmitigated in the base design. An optional second-provider, EU-resident WORM copy is priced in §5 (≈$23.5k/month, restore RTO in hours, outside the SLO).

**Residency: the literal conflict and two named policies.** Read literally ("no EU content byte exists on any device outside the EU"), the requirement cannot hold together with EU and US users collaborating on one document. A US collaborator who can read the doc holds its content on a non-EU device, and even E2EE does not help, since the plaintext is rendered there. We therefore define two policies and never present one as the other:
- **Strict residency (SR):** no non-EU device ever receives the tenant's EU content (EU-device-only, so no US collaboration). SR satisfies the literal text and gives up EU/US collaboration for that tenant.
- **Infrastructure residency (IR, default):** provider infrastructure outside the EU never stores, processes, or caches EU content. That covers ciphertext, indexes, backups, logs, crash dumps, and CDN. Authorized non-EU collaborators receive content on their own devices over TLS.
  - IR satisfies collaboration and **does not satisfy the literal text**.
  - The exception covers the authorized non-EU collaborator's device (storage and processing) and the public-internet TLS transit to it, which no provider component stores or caches.
  - EU-pinned traffic enters only through EU-announced anycast prefixes and IP-literal EU endpoints, so none of our non-EU PoPs carry it.

Tenants can choose SR, or IR plus the stricter device option below, at the cost shown in §3B:
- No-offline-cache: non-EU devices hold content in memory only. This is client-enforced and best-effort on managed devices (MDM). It is not claimed on unmanaged devices, where swap, crash dumps and screen capture cannot be controlled.

**Content and metadata classification.** Rule: content is stored, processed and cached only in its home jurisdiction. Restricted metadata follows the same rule, so where the content/metadata line falls on the op envelope does not change where it may live. Only the two global routing items and k-anonymous aggregates are exported.

| Item | Content or metadata (sensitivity) | Form and location | Justification and residual leak |
|---|---|---|---|
| Document body, ops (envelope and payload), snapshots, comments, titles, thumbnails | Content | Plaintext (std) or ciphertext (E2EE); home jurisdiction only | |
| Search indexes | Content | Std: plaintext inverted index. S1: client-side only. S2: keyed-token blocks, treated as content | S2 leaks token equality, frequency, and access patterns |
| Segments, backups, manifests | Content | Same form as source; EU copies EU-only; manifests hold docID and a per-jurisdiction keyed hash (HMAC) only, in-jurisdiction | Keyed, so cross-tenant equality of segments is not testable |
| Doc ID (random 128-bit, no embedded time or tenant) | Metadata, global routing | Global directory: docID→(jurisdiction, shard group, epoch), ≈32 B/entry, 64 GB total | Random IDs are still linkable, so the directory holds no tenant, user, or time fields, is access-logged, and has no join key to user IDs outside the jurisdiction. Residual leak: a docID exists and has a home jurisdiction |
| Tenant ID→jurisdiction | Metadata, global routing | Global | Random; needed for login routing |
| User ID | Metadata, personal data | Home jurisdiction. Docs hold pairwise pseudonymous subject IDs, and the mapping stays in the user's home jurisdiction | GDPR personal data |
| User→docs reverse index | Metadata, restricted | User's home jurisdiction; docID and home jurisdiction only, no titles. Sized and priced in the metadata line (§5) | Needed for "shared with me"; titles are fetched from the doc's jurisdiction on open |
| ACL, sharing graph, membership | Metadata, restricted (content-adjacent) | Doc's home jurisdiction only | Reveals who collaborates with whom |
| Timestamps, sizes | Metadata, restricted | In-jurisdiction. Exported globally only as aggregate buckets with k≥1,000 docs and no tenant breakdown | Residual: bucketed volume per jurisdiction |
| Op envelope | Metadata, restricted (the op as a whole, envelope plus payload, is treated as content for placement) | In-jurisdiction. Std: actor, counter, lamport, size, type. E2EE: only docID, device pseudonym, counter and prevHash (dedupe, fork detection), size padded to 256 B, keyEpoch, schemaVer, op class (content/ACL/key/snapshot) | E2EE lamport, deps, and semantic type are inside the ciphertext. Residual leaks (accepted, documented): typing cadence, size buckets, timing of membership, key-epoch and snapshot events, doc existence and activity, connecting jurisdiction |
| Telemetry | Metadata, hashed IDs with jurisdiction-local salt | No raw IDs leave | |

**Added workload assumptions**
- Diurnal average = 50% of peak.
- Documents and sessions: "average 3 collaborators" is read as 3 concurrently connected sessions per active doc, so 5M sessions give 1.67M active docs. Assumption: editing is clustered (40% of active docs have ≥1 editor, 1.5 editors each), so 1.0M editors sit in 670k edited docs. Each message has 2 receivers. Independent placement would give 815k edited docs (§5 sensitivity).
- **Regional equivalence (assumption).** Compute demand and stored bytes per DAU are equal across product regions: same session length, editing intensity, document size, E2EE share, search load and popularity. Compute is therefore split by traffic share (§4A), and **EU content is 25% of platform bytes**, equal to the EU's DAU share. The prompt gives a traffic mix only, not a byte mix. §5 sensitivity re-runs the EU byte share at 35%, and V14 re-measures it before P1 exit.
- E2EE is a tenant policy. 10% of docs belong to E2EE tenants, with the same average size and edit rate (so 10% of bytes and ops). Their data is incompressible ciphertext with +5% envelope overhead.
- 8% of docs are hot (touched in the last 30 days) = 160M docs.
- Current-state snapshot = 25% of logical size (125 KB). Hot SSD holds it uncompressed for fast DSO activation.
- **Logical size and CRDT metadata.** The scenario's 500 KB is logical storage including history, in the system's own representation. We read it as already including CRDT identifiers, tombstone skeletons, op metadata and structural overhead. The 1.3–2× per-character overhead of §2 is relative to plain text and sits inside the 500 KB. §5 sensitivity prices the opposite reading.
- Compression is 2.5:1 on non-E2EE stored data, applied alike to segments and cold snapshots (the hot tier is uncompressed). Stored fraction = 0.9/2.5 + 0.1×1.05 = **0.465**, so 1 PB logical stores as 465 TB.
- Durable op bytes = 70% of wire bytes (210 B).
- Presence: 0.5 Hz × 60 B from 50% of sessions.
- Join: 70% delta-sync (5 KB, 8 ms CPU), 30% full (50 KB, 40 ms CPU).
- Mean session is 20 min.
- Offline: 5% of DAU per day = 2.5M reconnect syncs/day, averaging 100 KB up and 1 MB down. The 0.1% long-offline cohort is 50k users, mean episode 14 days, 20 MB up and 5 MB down per reconnect.
- History folds ≈10× after 30 days.
- Memory is priced inside the vCPU-hour at 4 GB/vCPU.
- Binary assets (images, attachments) are content-addressed objects of ≤25 MB each, outside the 25 MB document-state limit. They are replicated like segments and assumed to be inside the 500 KB average.

## 1. Architecture and data model

**Tiers.** Edge (anycast, L4 pass-through, TLS never terminates outside the doc's jurisdiction) → Gateway (stateless; QUIC/WebSocket, auth, rate limits) → Relay (fan-out, viewport filtering) → **Document Session Owner (DSO)**, one per active doc, co-located with the shard-group Raft leader under its lease → **Raft log** → segment and snapshot store (object storage) → search indexers. A control plane holds the directory, config, and key management.
- For plaintext docs the DSO holds CRDT state.
- For E2EE docs it holds only sequencing state: head index, dedupe window, key epochs, and ACL position.

**Sharding and leader affinity.** docID → vshard (one of 2¹⁶, assigned at creation by the placement service and recorded in the directory, not by hash alone) → one of ~16k Raft groups (EU ≈ 4,096, ≈122k docs per group), with online split and merge.
- A group is the unit of failover and has exactly one leader. Leader-affinity classes (§0) are per group.
- A *cell* (~2% of tenants) is the unit of rollout.
- **Doc handoff** (re-homing to another affinity class, and isolation into a single-doc group):
  - The old group appends `MigrateOut(docID, snapshotRef, lastIndex)`, then the new group's first entry for the doc is `MigrateIn(docID, snapshot)`.
  - The directory epoch is bumped by CAS on the Raft term, and the old group then refuses the doc.
  - The doc is read-only during the handoff (≤2 s). A client holding an old-group cursor receives snapshot + tail.
  - ACL positions and E2EE key epochs travel in the entries.
- **Hot-doc isolation and sequencing admission.**
  - The sequencer caps each doc at 500 batched messages/s (≈150 KB/s). That is 2.5% of a group's assumed ≈20k entries/s append capacity (benchmarked in P0).
  - Each group uses weighted-fair queueing with the cap as the per-doc ceiling, so a capped doc cannot starve its neighbors' appends.
  - Excess load is back-pressured by stretching the client batch interval 1 s → 2 s → 4 s.
  - The **isolation trigger sits below the cap**, because the cap bounds append load but not the doc's DSO, fan-out and WAL-flush load on its neighbors: a doc sustaining ≥250 msg/s (half the cap) for 60 s is moved online into its own single-doc group by the handoff above.

**Write path**
1. The client sends an idempotent batch keyed by opID = (actorID, counter), signed by its device key.
2. The gateway checks auth, size, and rate.
3. The DSO validates:
   - *Plaintext docs:* decode, schema, deps and limits, then apply on a native reference core over a copy-on-write overlay. Ops flagged high-risk (new op types, unusual depth or shape, ≈0.9% of ops) plus a 0.1% random sample (1% in total) are also shadow-applied in sandboxed WASM cores for **all 13 supported client builds** (N-12…N), using a bounded context window around the op's anchors.
   - *E2EE docs:* envelope validation only (device signature checked against the ACL, size, rate, dedupe, per-actor counter order, key epoch).
4. The DSO appends to Raft. The leader fsyncs locally and sends the entry to both followers in parallel. The entry's `(term, index)` is its `seq`, not a separate counter. In parallel the DSO fans out *tentatively* via relays, tagged with `(term, index)`.
5. **Commit = leader plus the first follower to fsync and ack (2-of-3).** The commit watermark advances and `ACK(opIDs)` goes to the sender. The watermark piggybacks to everyone.
   - The slower follower catches up asynchronously and is not on the ack path, but **its lag is bounded**.
   - If it falls more than 2,000 entries or 2 MB behind the head, the leader stretches the group's batch interval until it is back inside the bound. After 5 s beyond the bound it is marked *lagging* and reseeded.
   - With the bound, the slower follower is at most ≈20 ms of transfer (at ≥100 MB/s per group stream) from complete. After any single-region loss the surviving pair can therefore resume durable acks within one catch-up transfer plus one RTT (§4A).

**Coordinates.** Three structures are kept separate. A log index is never read as a per-actor counter, or the reverse.
1. *Log position:* the Raft `(term, index)` is group-wide. A doc's total order is the subsequence of its group's log that carries its docID, so per-doc linearizability follows from the group's total order. headIndex, snapIdx and the commit watermark are log positions. A segment covers a group index range for all docs of the group and has a per-doc offset index in its footer.
2. *Causal context:* a per-doc version vector over actors (actorID → counter range set plus the hash of the actor's last op). The CRDT, dedupe and the snapshot's skip-set use it.
3. *Client sync cursor:* the pair (groupID, last applied log index) plus the client's version vector. SYNC sends both. The server serves the doc's log suffix after the cursor via the per-doc offset index. It uses the version vector only to skip duplicates and to compute a delta when the cursor has been folded away.

**Tentative-state protocol.** Receivers hold committed state Sc (ops ≤ watermark) and a journal J of tentative ops keyed by opID. The view is Sc ⊕ J.
- On a term change the new leader publishes `TermStart(term, firstIndex, retained[(index, opID)])`, where `retained` lists the old-term uncommitted entries it holds. Receivers keep a tentative entry only if its opID is retained (re-tagged with the new index) or reappears. All other older-term entries at or beyond the divergence point are dropped, then the view is recomputed from Sc ⊕ J′.
- Ops that depend on a dropped op wait up to 10 s for their live sender to re-upload it (idempotent by opID), then are dropped. The author's client re-anchors its own dependents to the retracted op's recorded origin.
- A tentative entry can disappear from the log on a leader change even while its sender is alive. The sender has no ack, so it re-uploads by opID and the op reappears. A *permanent* retraction occurs only if the sender and every durable holder are lost inside the commit window (46–160 ms), and it appears to receivers as an ordinary remote deletion.
- Clients key their outbox on opID, not `seq`.

**Data model**

| Object | Key → fields | Storage |
|---|---|---|
| Document | docID → tenantID, homeJur, shardGroup, schemaVer, textAlgoGen, keyMode (inherited from the tenant's E2EE policy at creation), aclVer, headIndex, snapshotIdx/gen, size, created/updated | Metadata DB, Raft RF3, in the doc's home jurisdiction |
| Op log entry | (groupID, term, index) → docID, opID, prevHash, opClass, schemaVer, keyEpoch, payload (plain or ciphertext, stored verbatim and never re-serialized by the server). **Plaintext entries also carry lamport, opType and deps. For E2EE those three live inside the ciphertext** (§0 envelope row) | Raft WAL on NVMe, mirrored across 2 zones. Flushed per group as zstd segments `{group}/{startIdx}-{term}.seg` every 60 s or 384 KB raw (typically 60–250 KB compressed) to object storage in primary and buddy regions, plus a WORM copy (§5). Manifests live in the metadata DB and change by CAS on (term, version). A partial upload is invisible until its manifest CAS succeeds |
| Snapshot | (docID, gen, snapIdx) → state blob, skip-set hash, tombstone skeleton, stateHash, keyEpoch, trust flag, `snapFmt` | **Plaintext:** hot copy on SSD on each voter (uncompressed). Every voter materializes its own from the log, so there is no snapshot network traffic. Cut every 2,000 ops, 15 min of activity, or 256 KB tail. Cold: object, written by the leader at DSO deactivation and every 6 h of continuous activity, packed into the group's segment objects. **E2EE:** the server cannot materialize state, so a client uploads an encrypted snapshot as chunked log entries (≤64 KB each) ending in `SnapshotCommit(chunk hashes, signed stateHash)`. The chunks replicate through Raft like any entry (durable on 2 of 3 at ack) and land in the segments, and the latest committed snapshot is the hot copy. A client that disconnects mid-upload leaves chunks with no `SnapshotCommit`. They are ignored and garbage-collected after 24 h, and another member client re-cuts. No server-side cold snapshot is written for E2EE docs. Replication of these uploads is priced in §5 |
| ACL, membership, restore, comments | Log entries of the same doc (comments are log entries with an anchor and a pseudonymous author; ciphertext under E2EE) | Same log, so revocation has an exact position in the total order |
| Search index | Per-doc index segments; std: plaintext; S2: encrypted token blocks | Primary and buddy region (RF2). Not in the WORM copy: it is derived data, rebuilt from restored content in hours, outside the SLO |
| Presence | (docID, session) → pseudonymous user, cursor/selection, viewport, TTL 10 s | DSO memory only, never persisted, encrypted under E2EE. Rebuilt from client heartbeats within 10 s after any DSO move or failover |
| Directory | docID→(jur, group, epoch); group→leader (cache, validated by Raft term) | Global, replicated to every region, minimal fields. Single writer per entry (the home jurisdiction's authority); epoch changes by CAS on the Raft term |

**History policy.**
- Full-fidelity ops are kept for 30 days. Older history folds into named and daily checkpoints.
- Tombstone *content* is dropped at 35 days. The ID-run *skeleton* is kept until the document is purged, so late anchors still resolve deterministically. It holds ranges only, no content: actor index, counter start, length and left-origin reference, ≈10 B per tombstoned run. That is ≈7.5 KB (≈1.5% of logical size) on the average doc, inside the 25% snapshot allowance. Canvas objects keep an equivalent tombstone record (id, parent at delete, ≈40 B) until purge.
- The skeleton carries only pseudonymous actor IDs. Erasure of a user's pseudonym mapping unlinks them.

**Zone analysis.**
- A voter's WAL is a synchronous 2-zone pair (fsync on both before the voter counts as durable, +1 ms), so a single node or zone loss degrades a voter without losing it.
- Snapshot SSD is single-zone and rebuildable from object storage.
- Gateways, relays, and DSOs are stateless or lazily reactivated. Zone loss leaves 2/3 of a region's capacity, so utilization goes from 40% to 60%, the same N+1 rule as for regions.

**Session routing.**
- The client asks the directory for the doc's jurisdiction (cached ≤60 s) and connects to that jurisdiction's gateway. For EU-pinned docs under IR, US clients cross the Atlantic directly, with no non-EU relays. Gateways resolve group→leader from cache, and a stale cache is corrected by an epoch-mismatch redirect (one RTT).
- Clients carry a signed config with IP-literal endpoints for all three voter-set regions, so failover has no DNS-TTL dependence. The config is not baked into the build: it is refreshed on every connect, cached for ≥30 days, and signed by the build trust root. Anycast and IP-literal bootstrap endpoints are kept stable for ≥13 months, so a client that was offline for 30 days still reaches the service and refreshes its config.
- In v1, all remote clients connect straight to the home gateway. For unpinned (non-EU) tenants from P3, a client attaches to its nearest local relay, which forwards over the backbone to the home DSO's gateway.

**Identity across jurisdictions.**
- Display names and avatars are not copied between jurisdictions. A client resolves each member's pseudonym through that member's home identity service, authorized by a signed profile-share token issued at invite time. Results are cached 24 h on the device, with initials as the fallback.
- Device credentials are short-lived tokens (≤1 h) signed by the user's home IdP and validated at the doc's gateway against cached IdP keys.
- **Account-level revocation** is a signed statement from the home jurisdiction. It is pushed to the revocation cache of every gateway serving a doc the user belongs to, and appended to each such doc's ACL log. The maximum revocation delay is an explicit security property:
  - Writes by a revoked account are rejected once the statement reaches the doc's gateways (design ≤30 s by push). If the push is lost, the ≤1 h token expiry bounds the delay.
  - Writes accepted inside that window precede the revocation's log position, so they are valid. They are tagged `postRevocationWindow` for owner review and are not rolled back.
  - An admin who needs immediate, exact-position removal uses the per-doc ACL change.
  - During a jurisdiction partition remote users' tokens expire and writes fail closed.
- Document-level ACL changes are positions in the doc's own log, so revocation there has an exact position.

**Protocol and schema evolution (12-month clients, zero downtime)**
- The supported window is N-12…N (13 monthly builds). The wire format is protobuf with append-only fields. Unknown fields and op types are preserved as opaque bytes.
- The log is immutable. Read-time upcasters (N-12…N) convert plaintext ops, so history is never rewritten. E2EE payloads are upcast by clients, since the server cannot read them.
- **Bootstrap is a frozen interface.** The QUIC/TLS handshake (versioned by ALPN), the device-token refresh endpoint and the signed-config fetch are never removed or changed incompatibly inside the 13-month window. V8 runs an N-12 client through bootstrap, token refresh and config refresh against the current server.
- **Snapshot format** is versioned independently of ops (`snapFmt`). A snapshot is a base section (the N-12 representation) plus append-only extension sections that older clients skip. `stateHash` covers the canonical encoding of the base section, so builds N-12…N compute equal hashes. A generation change re-bases the snapshot at the `GenerationChange` index with its `checkpointHash`.
- New semantics ship expand → dual-write → contract. Contract waits ≥13 months after release.
- Old clients keep working without downtime or crashes on every existing feature. New features reach them only as an additive embed with a fallback representation, or are gated per tenant by min client version. Old clients preserve what they cannot interpret but cannot use it.
- A text-CRDT algorithm change (e.g. away from Fugue) is a doc *generation* change, `GenerationChange(index, algo, checkpointHash)`, applied at the same log index by all replicas. It happens only after the tenant's min client version passes the new algorithm's release by ≥13 months. Until then docs stay on the old algorithm.
- Raft log-format bumps use a cluster-wide feature barrier after all replicas upgrade. Rolling deploys run N and N-1 servers side by side.
- Byte-exactness holds at the log and segment level, because the server never re-serializes. At the client it is semantic preservation of unknown fields.
- An emergency security fix can force-expire a vulnerable build (§3, item 10). That is the one deliberate exception.

**E2EE**
- **Policy:** E2EE is a per-tenant policy fixed at tenant and doc creation, and each doc's `keyMode` is inherited from it. Plaintext docs are not converted in place. Moving into an E2EE tenant is a client-side copy.
- **Keys:** a per-doc DEK (XChaCha20-Poly1305) with epochs, HPKE-wrapped per member device. Each new epoch is also wrapped to the tenant recovery public key RK_pub. MLS is used above ~20 members, since TreeKEM is O(log n), which matters for large docs. Key-epoch records form a hash chain in the log, so backups restore the wraps.
- **Server role:** it sequences and replicates envelopes only. Clients produce and upload snapshots (data model above), compact, and upcast. Clients compute and sign `stateHash` at checkpoints.
- **Snapshot trust:**
  - A snapshot is trusted when ≥2 distinct devices sign the same hash.
  - Docs with a single active device in the last 30 days use a delayed-trust path. The one device's signature becomes trusted after a 35-day hold, during which the pre-fold segments stay in the WORM copy (the 35-day lock) and the device re-verifies by replay.
  - Trust establishes agreement between devices, not semantic correctness. A bad checkpoint therefore stays recoverable by replay for the lock period.
  - WAL truncation never needs a trusted snapshot (I5a), so single-device docs do not block GC. Only their folding lags, costing +$1.2k/month (§5).
- **Removal:** triggers a new key epoch. Removed members keep what they already had.
- **Admin recovery (tenant-admin, not provider-assisted):**
  - The RK private key is Shamir-split 3-of-5 across admin hardware keys, or the DEK wraps use a tenant HYOK KMS in the tenant's jurisdiction. The provider holds zero shares.
  - The five keys are held by distinct admins in ≥3 countries (for EU tenants, ≥3 inside the EU). A resharing ceremony runs when an admin leaves, and compromise revokes RK and re-wraps.
  - Recovery: ≥3 admins sign a request, which is logged to a key-transparency log visible to members, with a tenant-configurable cooling period (default 24 h). The admins reconstruct RK, unwrap the DEK epochs, and re-wrap them for new members.
  - RK rotation re-wraps DEK epochs in the background from an admin client.
- **Offline key-epoch rule:** an op authored offline carries the epoch it was encrypted under. It is accepted iff the author device was a member at that epoch and has not been removed by upload time. Otherwise it goes to the owner-review inbox. Accepted old-epoch ops stay decryptable, and members receive wraps for past epochs.
- **Presence and routing:** relays cannot read E2EE cursor or viewport data. E2EE docs therefore get no server-side viewport scoping, and fan-out is full, so they are capped at **100 editors and 900 viewers (1,000 sessions)**. An opt-in 6-bit coarse region bucket in the envelope restores ≈16× reduction at the cost of leaking coarse scroll position (default off).
- **Search:** S1 and S2 are specified in §3C. Search over offline-edited docs works locally via S1. S2 does not cover offline edits until sync.

## 2. Consistency model and conflict resolution

**Model, in named layers**
1. *Committed log:* linearizable per doc (Raft; ack = commit). There are no cross-document transactions.
2. *Replicas:* CRDT strong eventual consistency. Any two replicas that have applied the same set of committed ops and the same skip-set have identical state.
3. *Tentative view:* read-your-writes by local apply. Monotonic reads are defined on the committed prefix (the watermark never regresses). Tentative remote ops may be retracted (§1 protocol).

| Data | Mechanism |
|---|---|
| Text | Fugue (non-interleaving sequence CRDT) with Peritext formatting marks, run-length-encoded IDs |
| Object properties (canvas) | LWW register per (object, prop), ordered by the op's **commit position** (term, index) in the log. Tentative order is provisional until commit. Wall clock and Lamport time are never used for LWW: a Lamport counter counts events, not time, so a prolific offline client could out-rank a recent online edit |
| Hierarchy / z-order | Kleppmann move-tree (undo-redo by (lamport, actor), a causally consistent total order). The move op carries `(newParent, fractionalIndex)` atomically, so the tree is authoritative for parentage and the index is only a sibling-order key that cannot disagree with it. Equal indices tie-break by (actorID, opID), and renormalization is an ordinary move op. **Cycle rule:** a move is evaluated at its position in the (lamport, actor) order and skipped *there* if it would create a cycle in the state at that position. Skipping is not permanent. Move records are retained 35 days (the tombstone horizon), and when an earlier-ordered move arrives, undo-redo re-evaluates every later move. A previously skipped move can then take effect and a previously applied one can be skipped. Every replica evaluates the same ordered set, so results are identical |
| Sets (reactions, tags) | Add-wins OR-set |
| Delete vs concurrent edit | **Text:** a delete tombstones exactly the character IDs the deleter had observed. A concurrent insert, including one inside the deleted range, survives, because a delete never captures unseen inserts. Concurrent formatting marks on deleted characters stay on the tombstones, invisible. The delete wins only for the same character. Undoing a delete re-inserts the content as new IDs at the original left origin, and anchors resolve through the skeleton, so the result is deterministic after 30 days offline. **Objects:** the tombstone wins over property edits and a restore op exists. Connectors, component instances and comment anchors that reference a deleted or quarantined object become no-ops that keep a dangling reference with a visible placeholder. Object IDs are never reused |

**Convergence (formal statement).**
- Let S be the set of committed ops, Q the skip-set, σ the schema interpretation, and ρ the deterministic conflict rules (commit-position LWW, move-tree order, OR-set, tombstone rules).
- Any two replicas with the same (S, Q, σ, ρ) have identical state. Merges are commutative, associative, and idempotent (dedupe by opID), and every causally valid delivery order of S∖Q gives the same state.
- Retractions touch only tentative ops, which are by definition not in S. A replica's view is Sc ⊕ J and becomes Sc ⊕ J′ after each `TermStart`.
- S is closed under dependencies, enforced separately per mode:
  - *Plaintext:* the DSO checks deps at validation. An op with a missing dep is held ≤10 s for its re-upload, then dropped.
  - *E2EE:* the DSO cannot read deps. It enforces only per-actor order from the envelope: an op whose counter is not the successor of the actor's last appended counter is held ≤5 s, then NACKed for in-order re-upload. Cross-actor dependencies are on ops the author received after the DSO appended them, so a dependency precedes its dependent in the log except around retractions. Clients buffer any op whose decrypted deps are missing, receive missing ops via SYNC (the log suffix after the cursor), and an op whose dependency stays missing past 10 s after a `TermStart` that did not retain it is re-anchored by its author.
  - This supplies what Fugue assumes: reliable, causally ordered delivery with retransmission of ops missed offline, via the append-only log plus SYNC.
- The DSO publishes `stateHash(index)` every 1,000 ops for plaintext docs. For E2EE docs, clients publish signed hashes instead (§1). A mismatch triggers resync and a divergence report.
- **Op identity.** opID = (actorID, counter). The actorID is a per-install random 128-bit value, and each op is signed by a device key (hardware-bound where the platform allows) and carries `prevHash`, the hash of the actor's previous op, forming a per-actor hash chain. The counter is persisted ahead of use in blocks of 1,000 and is never reused. A restored device learns the server's highest-seen counter for its actor in the SYNC reply and, on a rollback, rotates to a new actorID.
  - **Cloned installations.** Two clones that go offline carry the same actorID and counter range.
  - On upload the same opID with the same hash is a duplicate. The same opID with a different hash is an `ActorFork`: the first arrival keeps its IDs, and the second batch is rejected with `ActorFork(sharedPrefixCounter)`.
  - The second client rotates to a new actorID, deterministically remaps its unsynced outbox past the shared prefix (new IDs, intra-outbox references and signatures; under E2EE this is done client-side), and re-uploads.
  - Rejected ops never entered the log, so convergence is unaffected and no edit is lost. The cost is one outbox rewrite on the second client.
- **Dedupe.** It uses per-actor counter range sets plus the last-op hash per range, which compact into the snapshot's version vector. A duplicate older than a fold is therefore still rejected and a fork is still detected, at O(actors) per doc.

**30-day offline**
- The client keeps a local replica and a durable outbox. Reconnect sends `SYNC(cursor, versionVector, outbox)`. It returns a delta, or snapshot + tail past the window. The server keeps no per-client sync state beyond a device registry row, because the client pulls from the immutable log and snapshots. The only server cost is the retention already modeled (30-day log, 35-day tombstones).
- **Boundary semantics:** "30 days" means 30 × 24 h since the device's last successful sync. The 5-day margin to the 35-day tombstone horizon absorbs clock error and sync scheduling. Correctness does not depend on a log-retention margin. If the client's cursor has been folded, SYNC returns the newest snapshot (with version vector and tombstone skeleton) plus tail, the client merges its replica with it by IDs, and its outbox ops integrate against the skeleton.
- **Four separate properties.** We do not let one stand in for another:

  | Property | Status |
  |---|---|
  | Deterministic convergence | Guaranteed (statement above) |
  | Guaranteed acceptance of every offline edit | Not guaranteed (revocation, below) |
  | Preservation of user intent | Text insert/delete preserve intent. LWW properties do not: one of two concurrent values is displaced |
  | Conflict visibility and recovery | Provided: displaced values stay in property history, with a banner and one-tap undo |

- **LWW:** an offline edit is sequenced when it reaches the log, so it overrides any online edit of the same property committed in the meantime (reconnect-wins, as in server-authoritative canvas tools). This is deterministic and works under E2EE, because the commit position is visible to every replica. The displaced value stays in property history. The client shows "replaced X's newer edit" with one-tap undo (a new op restoring the prior value), and flags overwrites of another actor's edit committed after the op's base. Lost-edit exposure (LWW overwrites and object delete-vs-edit conflicts) is logged and measured, not assumed. No guarantee is made that either side's concurrent LWW edit survives (§3, item 11).
- **Move-tree:** integrating an offline batch undoes and redoes all later moves, O(moves in window). The budget is 20k moves × 10 µs = 0.2 s for a p99 doc (10 µs/move is an assumption, benchmarked in V5). Docs with >200k moves in the window rebuild from a checkpoint on a background thread.
- **Revocation:** ops submitted after ACL revocation are rejected and parked in an owner-review inbox (cap 10k ops per doc, 90-day retention). We cannot verify pre-revocation authorship, so some legitimate offline edits are lost. *Convergence* is guaranteed. *Acceptance* of every legitimate offline edit is not.
- Convergence is guaranteed for offline durations ≤30 days. Beyond that it is best-effort via the skeleton.

**Why not the alternatives**
- **OT** can run with a sequencer-only server and client-side transforms, so E2EE alone does not rule it out. We reject it because transform cost and history grow with a 30-day concurrency window, and it has no clean story for non-text canvas types. CRDT ids anchor edits across 30 days of divergence.
- **Figma-style server-authoritative LWW for text** loses intent. We use it only for canvas properties.
- **CRDT without a sequencer** has no commit point for RPO, no exact revocation position, and no revocation-aware snapshot or compaction. Snapshots themselves are possible.

**Costs:** per-char metadata (≈1.3–2× plain text after RLE, inside the 500 KB logical size per §0), tombstones, and LWW surprises.

## 3. Guarantee matrix (three most severe conflicts)

**Why these three.** They are the conflicts where meeting one requirement forces violating another at the design level, with no amount of money removing the trade:
- A is physics and CAP (durability × latency × availability).
- B is law versus collaboration.
- C is cryptography versus server-side function.

Considered and covered elsewhere: 12-month compatibility × emergency security fixes (§1, list below) and offline × revocation × GC (§2).

All costs in the matrices are components of the §5 base total, not additions to it, unless marked "not in base". The status in each property cell uses the legend of the summary.

**A. Durability × latency × availability under region failure**

| Property | Preconditions | Single-region failure | Partition | Degradation | Decision | Cost ($/mo) | Sacrificed |
|---|---|---|---|---|---|---|---|
| RPO≈0 (conditionally met; architectural) | Ack after fsync on ≥2 of 3 voters (leader + first follower) in distinct regions [1] | ≥1 copy of every acked op survives; 2 voters re-form quorum | Minority cannot commit (CP); clients edit locally, shown "unsaved" | 2nd voter lost → read-only; never ack on 1 copy | Raft 2-of-3 cross-region | $19.5k: $10.9k Raft inter-region (544.3 TB × $20) + $8.6k follower apply (80 raw ×3.75 ×$28.8) | Ack latency [2]; minority write availability |
| p99 <200 ms in-region (partially met; operational) | Client ≤50 ms RTT; leader in-region; lease valid | Docs led from the failed region stall ≤12 s (typically 5 s). LATAM→use adds ~115 ms RTT, p99 ≈270 ms | Leader serves tentative fan-out ≤2 s under lease, then read-only | Tick 20→50 ms; shed presence first | Tentative fan-out decoupled from durable ack | $0 infra (same bytes) | "Visible = durable"; LATAM/MEA latency SLO during failover; per-class exceptions of §5 |
| RTO ≤2 min + 99.99% (conditionally met; RTO a protocol property, availability an operational SLO) | Warm followers; replication lag bound (§1); ≤40% steady, ≤60% post-loss steady CPU (surge transient ≤82% before shedding, ≤68% after, §4A); ticketed admission; 3-vantage detector [3] | Groups led ≤12 s; editors submit ≤30 s; forced-admit stop 61 s; contractual cap 120 s for the §4A cohort | Detector without quorum does not evacuate (a stall beats split-brain); term fencing still guards safety | Admission: submit > join (editing) > join (idle) > presence > search | Size for N+1 per voter set | +$43.9k vs a 70%-steady-utilization design (in the compute line; $38.5k uniform plus $5.4k per-set skew uplift) | Efficiency; ≈6 worst-case region events per 30 days would exhaust the *whole* budget, and the 60 s region allocation covers ≈1.4 |

[1] The ack path is DSO seq → leader fsync, entry sent to both followers in parallel, commit on the first follower's ack. The slower follower is off the ack path but still receives every byte (priced above) and is held within the lag bound (§1).

[2] Commit p99 (server side, DSO seq → watermark) ≈ 1.3 × median RTT to nearest follower + 10 ms (2 fsyncs + batching). The 1.3 factor is an assumption about the WAN p99/p50 ratio, not a demonstrated relationship. It is re-fit from measured RTT, fsync and queueing distributions in V4. Commit latency is off the propagation path and affects only ack latency.
- NA 46 ms (RTT 28).
- EU 49 (RTT 30).
- MEA 49 (uae-led, bom nearest, RTT 30).
- APAC 85 (sin-led, bom RTT 58) to 101 (tyo-led, sin RTT 70).
- LATAM 160 (sao-led, use RTT 115), or 190 with use down (usc RTT 137).

[3] Failure model for the RTO bound: one region lost; survivors and links healthy; directory healthy; offered load ≤ sizing; no follower already beyond the lag bound.

**B. EU residency × EU/US collaboration × EU-local survival**

| Property | Preconditions | Single-region failure | Partition | Degradation | Decision | Cost ($/mo) | Sacrificed |
|---|---|---|---|---|---|---|---|
| Residency (IR: met for provider infrastructure; SR: literal text, incompatible with collaboration; architectural) | EU tenant; data plane in EU voter set; policy-as-code; EU HSM root | Re-replicate only inside the EU (eu-par) | EU↔world cut: EU fully served; non-EU collaborators lose EU docs | Fail closed, never cross-border | IR default, SR per tenant. No relays, CDN, index, backup, or logs outside the EU; EU-only anycast prefixes | $3.0k (eu-par pilot light: 75 vCPU $2.2k + 8.3 TB NVMe $0.8k; its own base line) | Non-EU latency; global cross-jurisdiction search/AI; under IR **the literal "never leaves the EU" at the end-user device** |
| EU↔US collaboration (met under IR, excluded under SR) | One home jurisdiction per doc (no cross-jurisdiction multi-master) | US editors reconnect to surviving EU regions (RTT change ≤30 ms) | US editors cut off; local edits queue, subject to device policy | Viewer mode, editor cap | Remote sessions into the home jurisdiction | $3.1k (39 TB/month of cross-region traffic as internet egress, already in the base egress line) | US-side p99 ≈146–410 ms (meets 800, not 200). Under strict device policy, no US offline cache |
| EU survival of a region loss (conditionally met; architectural) | 3 EU voters + eu-par; EU ≤40% CPU | 2 voters keep quorum. Cold: primary, buddy, WORM copies in 3 distinct EU regions, so ≥2 survive | If the 2 survivors partition, EU writes halt (CP) | RF2 until reseed (hot ≈30 min = 10 min provisioning + 20 min copy; cold ≈6.5 h at 40 Gbps), plus an in-flight tail of ≤1 s at RF1. eu-par joins as learner, then voter by joint consensus (§4A) | 3 voters + spare region | $10.2k (EU N+1, skew-sized: 355 extra vCPU vs a 70% design; in the compute line) | A 2nd EU failure while degraded halts EU writes |

Reseed arithmetic: the hot copy is 5.8 TB (the EU's assumed 25% share of the 23.2 TB platform hot set, §0), which is 1,160 s at 5 GB/s. A cold copy is 116 TB (25% of the 465 TB platform stored data), which is 23,250 s ≈ 6.5 h. At a 35% EU byte share these become 8.1 TB (≈27 min) and 163 TB (≈9.0 h), and the spare NVMe grows by +$0.3k (§5 sensitivity).

**C. E2EE × search × admin recovery (× server-side validation)**

| Property | Preconditions | Single-region failure | Partition | Degradation | Decision | Cost ($/mo) | Sacrificed |
|---|---|---|---|---|---|---|---|
| Search (conditionally met, scoped to synced content) | S1: server sees ciphertext only. S2: server also sees keyed tokens | S1 unaffected. S2 index blocks replicate like segments, and if all copies are lost a member client re-uploads them (the server cannot rebuild) | Local search works offline | S2 off → local only | S1 default: client-side encrypted index. S2 opt-in: SSE with keyed trigram tokens and padded buckets, a distinct security mode | S1 $0 (client storage ≈35% of synced text). S2 +$0.9k, **not in base** (an upper bound if every E2EE doc enables S2: 10/90 × the $8.3k plaintext index line) | Server-side ranking, snippets, discovery of unsynced docs. S2 leaks token equality, frequency, and access patterns |
| Admin recovery (met; tenant-held) | RK 3-of-5 or tenant HYOK; shares never at provider | Unaffected (keys outside the regional data plane) | Needs admin quorum online | <3 shares → data unrecoverable by design | Tenant-admin recovery with transparency log | $0 infra (one extra ~100 B wrap per epoch) | "Only members can read" (admins can). Provider-assisted recovery is not offered |
| Server merge, validation, compaction (fundamentally incompatible with E2EE) | Payload unreadable | Same as plaintext (log replication is payload-agnostic) | Same | Clients produce signed snapshots; DSO sequences only | Client-side compaction and poison defense | $5.6k delta vs an all-plaintext fleet, already included in base via the 0.465 stored fraction: +65 TB stored × $72.45 per stored TB (= 2 × $23 + 1.15 × $23) + $0.7k segment copies + $0.2k client-snapshot replication | Shadow-apply, malware scan, previews, AI. Slower poison-pill mitigation |

**What cannot be met**
1. The literal "EU content never leaves the EU" together with EU/US collaboration. SR gives the literal text without collaboration. IR gives collaboration and holds for provider infrastructure only (see §0). IR is never claimed as SR.
2. Loss of the entire EU jurisdiction: EU tenants go unavailable, with no US failover.
3. Two simultaneous regions lost in one voter set (including a correlated loss of a whole product region): writes halt. If both regions' storage is also unrecoverable, a restore from WORM can lose acked ops newer than the last flushed segment, so RPO is ≤60 s in that case. The next tier is priced in §5.
4. Provider-wide failure: unmitigated in the base design.
5. In-region p99 <200 ms for these classes, with exposure of ≈0.8% of deliveries (≈0.85% including failover and stacked-loss tails, §5):
   - LATAM/MEA during failover, and LATAM throughout v1.
   - Same-region pairs editing a doc homed in another jurisdiction (p99 ≈270–590 ms with one loss round).
   - Multi-metro APAC docs under a stacked loss round (≈242 ms).
   - Docs whose editors have drifted from their placement class, until re-homed.
6. For viewer-tier sessions (the viewers of docs with >100 sessions), off-viewport ops are applied on scroll (≤300 ms after scroll), not at submit + 200 ms. These applies are excluded from the aggregate in-region population and reported separately (V4).
7. 99.99% if region events plus other causes exceed the budget (≈6 worst-case region events per 30 days would exhaust all of it; the 60 s region allocation covers ≈1.4).
8. E2EE together with server-side search, AI, and validation, and server-side discovery of unsynced E2EE docs. S2 is the leakier compromise.
9. Provider-assisted admin recovery for E2EE tenants.
10. The 12-month client window during an emergency security fix (we force-expire the vulnerable version), and new features for old clients (they get fallbacks).
11. Guaranteed acceptance of legitimate offline edits across ACL revocation, and any guarantee that a concurrent LWW edit survives (displaced values are kept in history).
12. "Visible = durable" at the same time as <200 ms.
13. In v1: 30-day offline and 12-month compatibility (until P2), and APAC/MEA service (until P3), §6.
14. E2EE docs are capped at 100 editors and 900 viewers (1,000 sessions). The 5,000-session design is plaintext-only.
15. Moving a document between jurisdictions in place (export/import only), and country-level residency jurisdictions other than the EU.

## 4. Failure analysis

### 4A. Full regional outage (eu-dub lost; compute and storage gone)

**Fencing.** The epoch is the Raft term, enforced at four layers.
1. Raft quorum.
2. Conditional object writes: segment names include the term, and manifests change by CAS on (term, version).
3. Gateways reject ops carrying a stale `leaderEpoch`.
4. A 2 s serving lease on monotonic clocks, with an election timeout ≥3 s.
   - **Assumptions:** clock drift ≤200 ppm and no clock steps. The leader re-checks its lease before every fan-out batch, so a paused leader that resumes after expiry serves nothing.
   - The 1 s margin exceeds drift (<1 ms) by orders of magnitude.
   - The residual is one in-flight tentative batch from a deposed leader, which receivers discard at `TermStart`.
   - The lease only bounds stale *tentative fan-out and reads*. Commit safety never depends on clocks, because it rests on quorum, term and conditional writes.
   - A zombie (partitioned but alive) region self-fences to read-only within 2 s, and the directory is only a cache.

**Safety invariants (the proof obligations, model-checked in §7)**
- I1: an acked entry is fsynced on ≥2 voters in distinct regions.
- I2: Raft leader completeness, so any elected leader's log contains every committed entry.
- I3: the watermark advances only on a current-term entry at quorum.
- I4: tentative entries are never reported as committed.
- I5a (WAL GC): the WAL is truncated only up to an index covered by a segment object verified in 2 regions, with its manifest CAS committed.
- I5b (fold GC): segment objects are deleted only after 30 days, a *trusted* fold checkpoint covering the range, and a confirmed WORM copy of the pre-fold segments.

**Hard RTO bound.** The failure model is one region lost with survivors and links healthy.

| Stage | Hard bound | Mechanism |
|---|---|---|
| Client detects | ≤3 s | 3 missed 1 Hz heartbeats; IP-literal endpoints for all 3 voter-set regions, no DNS wait |
| Re-election | ≤12 s | Pre-vote with priority-staggered timeouts (3/4/5 s by log recency, then region ID), no random split votes, ≤3 candidates |
| Routing | ≤0.3 s | Stale directory costs one epoch-mismatch redirect |
| Admission, editing clients (P0) | ≤25 s after first attempt | Server-assigned ticket slot, so repeated rejection cannot happen. Any P0 request older than 60 s is force-admitted ahead of the queue |
| First submit | ≤1 s after join | 1 Hz batch; lazy DSO activation ≤1 s (derivation below) |

- **Affected cohort and clock.** The RTO clock starts at the failure timestamp T0 and is **never restarted**. The affected cohort is every client with an editing intent on surviving capacity during the recovery window:
  1. clients with a pending outbox or an in-flight submit at T0, and
  2. clients idle at T0 whose first keystroke falls in [T0, T0+117 s].
- For each cohort member, first successful submit − T0 = max(t_recovery, t_keystroke − T0 + 3 s).
  - t_recovery is ≤29.3 s (design), ≤61 s (forced-admit ceiling) or ≤101 s (client fallback).
  - The 3 s is the keystroke-to-submit path: connect 0.3 s, join, lazy activation ≤1 s, 1 Hz batch.
  - The maximum over the cohort is therefore ≤120 s. Editors at the failure submit by **≤30 s** (design), ≤61 s (forced-admit ceiling), and ≤120 s (contractual cap). The remaining ≥59 s of margin is deliberately unallocated.
- Clients whose first keystroke is later than T0+117 s are outside the cohort, because the service has recovered by then. Their keystroke-to-submit latency (≤3 s) is the steady join path and is measured separately (V3).
- A cohort that included every connected client, whether or not it ever edits, would have no finite maximum, since no system can submit an edit the user has not made. We state this reading rather than hide it.
- **Idle sessions.** Idle (P1) sessions drain lazily at about 2k/s for the 600k idle sessions of a 750k-session region (≤5 min, ≈53 effective vCPU = 2k/s × 26.4 ms).
  - A first keystroke promotes the session to P0 immediately.
  - After the initial wave, P0 arrivals are about 500/s (150k editors, 5-min mean editing episode) against a 6k/s slot rate. A 10× promotion burst (5k/s) still fits.
  - The ticket wait is therefore ≤1 s, and first submit is ≤3 s after the keystroke.
- **DSO activation (≤1 s).**
  - The hot snapshot is on each voter's SSD, uncompressed, so activation reads ≤145 KB.
  - It replays the ≤256 KB log tail (≈1,200 ops × 40 µs ≈ 48 ms) and loads the skip-set and ACL position from the same SSD.
  - E2EE docs rebuild only head index and dedupe window from the tail.
  - The typical figure is 50 ms and p99 ≤1 s. A cold doc adds one object read (≤200 ms).
- **Election bound.** It is deterministic under the failure model.
  - Staggered priority timers mean the first candidate (3 s) runs unopposed.
  - The second (4 s) and third (5 s) cover a failed candidate, each round ≤1 s on healthy links.
  - The worst case is 5 + 1 + 1 = 7 s, which leaves slack under the 12 s cap.
  - Raft liveness is not unconditional. Outside the failure model (degraded links), a missed cap falls to the client fallbacks (61 s, 101 s). V1 simulates 10⁵ elections.
- **Resumption of durable acks (replication-lag bound).**
  - *Leader's region lost:* the new leader is the follower with the most complete log, by the priority timers. The other follower is ≤2 MB behind. The catch-up stream (≤20 ms transfer + 1 RTT ≤150 ms) rides on the first AppendEntries.
  - *Fast follower's region lost:* the leader still holds every acked entry, and the slower follower is ≤2 MB behind (≤20 ms + 1 RTT).
  - Either way, durable acks resume ≤0.5 s after the new leader is established. That falls inside the ≤1 s activation and ≤1 s first-submit allowances already in the critical path.
  - A follower already beyond the bound is a second fault, outside the failure model (§3A footnote [3]). Safety holds and writes in that group pause until it catches up, budgeted under gray failures.
- **Critical path:** t ≤ max(t_detect + t_admit, t_elect) + t_route + t_submit = max(3 + 25, 12) + 0.3 + 1 = 29.3 s, which is the ≤30 s design figure. Clients whose directory cache points at the failed region are inside this bound: their first attempt fails within the 3 s detection, and the epoch redirect costs 0.3 s.
- **Client-side fallback:** a client with no success at 61 s (for example, a stuck mobile NAT binding) re-handshakes on a fresh QUIC connection and tries each other endpoint in turn (≤2 × 20 s timeouts). It reaches ≤101 s, which stays inside 120 s.
- **Detector verdict:** the verdict (3 vantage points × 3 misses ≈ 10 s) only steers new sessions (BGP/DNS withdrawal, directory epoch bump). Safety never depends on it, because fencing is by term.
- **Preconditions:** the bound holds under the failure model in §3A footnote [3]. If the directory is down, load exceeds sizing, or a second fault occurs, it does not hold. Those causes are budgeted separately below. The 99.99% target is therefore a **conditional SLO** (it holds while region events plus other causes stay inside the error budget), not an unconditional guarantee.

**Timeline (EU, eu-dub lost)**

| T+ | Event |
|---|---|
| 0 | Failure |
| ≤3 s | Clients miss 3 heartbeats. The priority-1 follower's election timer (3 s) expires |
| 3–7 s | ≈1,311 EU groups (the dub-led affinity class, 4,096 × 8/25) elect leaders on fra/sto by pre-vote then vote. The 4 s timer covers a failed first candidate, and 12 s is the cap. 2 of 3 is a normal quorum, so there is no unsafe recovery |
| ~10 s | Detector verdict: BGP/DNS withdrawal, directory epoch bump |
| 3–28 s | P0 editors admitted by ticket at ≈3.2k joins/s |
| ≤29.3 s | Last P0 editor submits. DSO activation is lazy per doc (50 ms–1 s) |
| 28–328 s | The idle (P1) sessions drain at ≈2k/s. Any keystroke promotes a session to P0 (≤3 s) |

**EU-internal survivors**
- **Hot data:** fra and sto each hold a full copy, so RF3 becomes RF2.
- **Cold data:** primary, buddy, and WORM copies sit in 3 distinct EU regions, so ≥2 survive.
- **Sizes.** The 5.8 TB hot copy and the 116 TB cold copy are the EU's assumed 25% shares of the platform totals (23.2 TB hot, 465 TB stored, §0 assumption), because only EU content is re-replicated inside the EU. eu-par's 8.3 TB NVMe is 5.8 TB ÷ 0.7.
- **Replacement voter.** eu-par is a non-voting pilot light until promoted. Groups are reconfigured in waves of 5%, so catch-up traffic stays ≤40 Gbps.
  1. Scale out eu-par compute (≈10 min).
  2. Add it to each group as a non-voting learner, which streams the hot snapshot and log tail from fra/sto (5.8 TB at 5 GB/s ≈ 20 min).
  3. When lag is <1 s, run a joint-consensus change {fra, sto, dub} → {fra, sto, par}. The old majority (fra and sto) is alive, so the change commits through Raft and cannot split-brain. Dub's removal is part of the same change.
  4. Reseed cold copies in the background (116 TB at 5 GB/s ≈ 6.5 h).
- **ROW sets** (LATAM, MEA) use the same procedure with a replacement region provisioned anywhere in ROW (≈10 min).
- **RF1 window:** ops acked on the lost region plus one survivor are RF1 until the new leader re-replicates them to the other survivor. With the lag bound that is ≤2 MB of tail plus the in-flight batch: ≤1 s worst case, typically ≤0.2 s. A second region loss inside that window is the declared out-of-scope double fault.

**Capacity per region (unequal regions, shared voters).** The 40% steady and 60% post-loss targets are enforced per voter set, not as a fleet average. Effective demand is 832 raw vCPU × 1.5 = 1,248, split by traffic share (the regional-equivalence assumption of §0). Each set's capacity is sized so that losing its largest region leaves survivors ≤60%, with the failed region's load shifted in proportion to survivor capacity.

| Set | Demand D | Capacity per region | Total | Steady | Worst loss → survivors |
|---|---|---|---|---|---|
| NA | 436.8 | use 468, usc 364, usw 364 | 1,196 | 36.5% | use → 728 cap, **60.0%** |
| EU | 312.0 | fra 280.8, dub 260, sto 260 | 800.8 | 39.0% | fra → 520 cap, **60.0%** |
| APAC | 374.4 | sin 374.4, tyo 343.2, bom 280.8 | 998.4 | 37.5% | sin → 624 cap, **60.0%** |
| LATAM | 74.9 | sao 187.2 | 187.2 | 40% | sao → NA absorbs (see below) |
| MEA | 49.9 | uae 124.8 | 124.8 | 40% | uae → bom/fra, +25 each |
| **Fleet** | **1,248** | | **3,307.2** | **37.7%** | |

The skew uplift over a uniform 40% (3,120 vCPU) is +187.2 vCPU. If one region's measured per-DAU demand differs by ±20%, its row moves by ±20%. The per-set sizing is re-derived from measured regional demand (V14).
- **Shared physical regions.** A region in two sets (use, usc, bom, fra) is sized by the load it serves and hosts.
  - LATAM groups are predominantly sao-led (their leader/DSO work is in sao's 187.2). Their two followers are use and usc, so follower apply for them (7.2 effective vCPU in total) runs there. The ≈10% of LATAM groups led from use or usc place a follower on sao, which is ≤0.7 vCPU inside sao's headroom.
  - MEA groups are predominantly uae-led. Their followers (4.8 effective vCPU) run on bom and fra. The ≈25% led from fra place a follower on uae.
  - That is ≤1.1% of each host's capacity, and net-neutral to the fleet total.
  - Groups a region leads for another product region re-elect within ≤12 s. That adds <0.15 weighted seconds (LATAM groups led from use ≈10%, MEA groups led from fra ≈25%).

**Reconnect surge (every region, peak load).** Join cost is 17.6 ms raw (0.7 × 8 + 0.3 × 40), or 26.4 ms effective. P0 joins arrive at N/125 per second (20% of sessions over 25 s), and P1 starts after the P0 window.
- **Shedding.** It pauses the failed set's presence fan-out and index maintenance entirely for the ≈60 s surge. That is the set's share of the 141 + 152 = 293 effective vCPU of those tiers (NA 102.6, EU 73.3, APAC 87.9, MEA 11.7). Cursors freeze for ≤60 s, and the index catches up in <10 min.
- **Why a full pause.** A partial 35% shed would leave us-east at 76.8%, above V10's 75% bar.

| Lost region | Sessions | P0 joins/s | Join CPU | Survivor capacity | Before surge | Peak with surge | After shedding |
|---|---|---|---|---|---|---|---|
| us-east | 750k | 6.0k | 158 | 728 | 60.0% | 81.8% | 67.7% |
| us-central | 500k | 4.0k | 106 | 832 | 52.5% | 65.2% | 52.9% |
| us-west | 500k | 4.0k | 106 | 832 | 52.5% | 65.2% | 52.9% |
| eu-fra | 450k | 3.6k | 95 | 520 | 60.0% | 78.3% | 64.2% |
| eu-dub | 400k | 3.2k | 84 | 541 | 57.7% | 73.3% | 59.8% |
| eu-sto | 400k | 3.2k | 84 | 541 | 57.7% | 73.3% | 59.8% |
| sin | 600k | 4.8k | 127 | 624 | 60.0% | 80.3% | 66.2% |
| tyo | 500k | 4.0k | 106 | 655 | 57.1% | 73.3% | 59.8% |
| bom | 400k | 3.2k | 84 | 718 | 52.2% | 63.9% | 51.7% |
| sao | 300k | 2.4k | 63 | 1,196 (NA incl. LATAM) | 42.8% | 48.1% | n/a |
| uae (fra shown; bom 46.4 / 53.9 / 51.8) | 200k | 1.6k | 42, split | 280.8 each | 47.9% | 55.4% | 53.3% |

- **Reading the table.** The 60% figure is the *steady* post-loss level. The surge is a transient above it (≤82% before shedding, ≤68% after). The RTO does not rest on 60%. It rests on ticketed admission, which paces P0 joins at the slot rate that holds survivors at ≤82%.
- **Queueing.** At ρ = 0.82, the M/M/1 p99 wait for a 150 µs DSO service is ln(100ρ)/(1−ρ) × S = 4.41/0.18 × 0.15 ms ≈ 3.7 ms. Real service is closer to deterministic (M/D/1 halves the wait) and admission-controlled, so this is conservative. The 40 ms allowance in §5 covers GC and interference.
- **Large documents.** The ≈25k viewers of five live 5,000-session docs are P1 sessions, already inside the idle drain, and their relay trees re-form in ≤10 s.

**SLI and request-weighted availability.** The fixed SLI is (successful join + submit requests) ÷ (total join + submit requests) over a rolling 30 days, including regional outages and excluding offline local editing.
- **Success rule:** a submit succeeds iff it is acked (committed) within 5 s. A join succeeds iff the session is established and its first sync completes within 30 s of the client's first attempt (the client timeout). Anything slower is a failed request. These thresholds are our interpretation, since the prompt fixes the SLI form and not per-request time limits. Tighter thresholds would count more failures.
- **Counting rule:** each failed attempt, and each expired 30 s ticket wait, is one failed request. Tickets suppress retries, so 1 attempt/s per affected editor for T seconds is a ceiling.
- **Completeness of the failed-request count.** The classes below are all counted:
  - In-flight and new submits of affected editors, including their P0 join attempts, which sit inside the 1 attempt/s ceiling.
  - The one stale-endpoint attempt per client before redirect, also inside that ceiling.
  - Idle ticket waits (below).
  - Remote editors and EU docs with US editors (below).
  - Leader stalls for other sets in shared regions (<0.15 weighted s, above).
- **Request mix:** submits dominate. Average 0.5M/s × 2.592M s = 1.296×10¹² submits, against 5.4×10⁹ joins, so submits are 99.6% of requests.
- **Weighting assumption:** per-DAU submit and join rates and the document mix are equal across regions (§0), so a region's request share equals its DAU share. A region at 1.5× the average per-DAU rate scales its row by 1.5.
- **Partitions:** a partition isolating one region counts as that region's event (clients re-route to the majority side, and term fencing keeps it safe). A partition splitting the two EU survivors is the declared double fault.
- **Budget:** a regional event at traffic share s, duration T, and load factor ℓ (1 = average, 2 = peak) consumes s × T × ℓ "full-outage seconds". The budget is 0.0001 × 2,592,000 = 259.2 s.
- **Cross-check:** a us-east event at the cap, average load, fails 0.15 × 0.5M × 120 = 9.0M submits. That is 9.0M ÷ 1.30×10¹² = 6.9×10⁻⁶, or 6.9% of the budget, matching 18 s.

| Lost region | Share | Design (T = 30 s, ℓ = 1) | Cap (T = 120 s, ℓ = 1) | Worst (T = 120 s, ℓ = 2) |
|---|---|---|---|---|
| us-east | 15% | 4.5 s (1.7%) | 18.0 s (6.9%) | 36.0 s (13.9%) |
| us-central, us-west | 10% each | 3.0 s (1.2%) | 12.0 s (4.6%) | 24.0 s (9.3%) |
| tyo | 10% | 3.0 s (1.2%) | 12.0 s (4.6%) | 24.0 s (9.3%) |
| sin | 12% | 3.6 s (1.4%) | 14.4 s (5.6%) | 28.8 s (11.1%) |
| eu-fra | 9% | 2.7 s (1.0%) | 10.8 s (4.2%) | 21.6 s (8.3%) |
| eu-dub, eu-sto | 8% each | 2.4 s (0.9%) | 9.6 s (3.7%) | 19.2 s (7.4%) |
| bom | 8% | 2.4 s (0.9%) | 9.6 s (3.7%) | 19.2 s (7.4%) |
| sao | 6% | 1.8 s (0.7%) | 7.2 s (2.8%) | 14.4 s (5.6%) |
| uae | 4% | 1.2 s (0.5%) | 4.8 s (1.9%) | 9.6 s (3.7%) |

- **Remote editors (reproducible).** Remote editors are 3.3% of all editors (10% cross-region docs × 1 of 3 collaborators). In the worst case, ≤30% of them have their doc led from the failed region, so ≤1.0% of editors are affected. The stall is ≤12 s at P3, because they attach via relays, giving ≈0.12 weighted s. In v1, direct attachment means a ≤30 s reconnect, giving ≈0.30 weighted s.
- **EU docs with US editors.** These are 25% EU docs × 10% cross-region × 1/3 remote ≈ 0.83% of editors. The dub-served share (8/25) is ≈0.27% of editors, a further ≈0.08 weighted s at the design bound and ≈0.32 s at the cap (they connect directly to dub and must reconnect).
- **Idle joins.** An idle join waiting in its ticket slot counts one failed request per 30 s of waiting. For us-east, 600k idle sessions × ≈6 expired waits (mean wait ≈172 s) is 3.6M failed requests. That is 2.8×10⁻⁶, or 2.8% of the budget (7.2 s). P0 joins complete by ≈28 s, inside the 30 s timeout.
- **Worst event:** a us-east event at peak and the cap costs ≈43.4 s (36.0 + 0.24 + 7.2). So 6.0 such events would exhaust the entire budget (259.2 ÷ 43.4 = 5.97), which §3A states as ≈6. The 60 s region allocation below covers ≈1.4 of them.

**Control plane and shared dependencies** (static stability: no cloud control-plane call on the request path)

| Dependency | Failure | Behavior | SLI effect |
|---|---|---|---|
| Global directory (64 GB, replicated to all 12 locations) | Service outage (30-minute drill) | Replicas are a read cache, and stale routes are fixed by an epoch-mismatch redirect. Each jurisdiction's authority is the single writer for its docIDs, so updates cannot conflict. Writes queue and replay in order on recovery, and replicas reconcile by max (docID, epoch). Docs created during the outage are routable in their home jurisdiction at once. Share and invite links carry a signed (docID, jurisdiction) hint, so cross-jurisdiction first-open works without the directory | Only an open *without* a hint (a typed docID) from another jurisdiction fails, until recovery. Degraded-mode target: ≥99.9% of joins; submits unaffected |
| KMS/HSM | Regional loss | EU runs KMS in all 3 EU regions and ROW in use, usc and sin, each with replicated key material. ROW regions without a local replica (tyo, bom, uae, sao, fra-ROW) always call the nearest one. DSOs cache unwrapped DEKs of active docs. E2EE is client-keyed | None for active docs; new-doc key wrap +≤150 ms via the nearest other KMS region |
| DNS | Outage | Signed client config carries IP-literal unicast endpoints for every voter-set region | None for failover |
| Anycast PoP | PoP loss | BGP reroutes. Clients fall back to unicast VIPs | ≤3 s per affected client |
| Deploy/config | Bad rollout | 2% cells, automatic rollback on SLI burn within ≤5 min | Budgeted below |
| Provider IAM/control API | Outage | Data plane keeps running; scale-out and reseed block, so RF2 lasts longer | None for serving |

**Brownouts and gray failures.** A region or node is drained when any of these holds:
- Submit success is <99% or ack p99 is >3× the 7-day baseline for 20 s, seen from ≥3 vantage points.
- **Burn-rate rule:** the error rate is >0.1% (10× the budget burn rate) for 5 min from ≥3 vantage points. A sustained 0.5% error rate therefore triggers within 5 min and costs ≈1.5 full-outage seconds.
- Node fsync p99 is >50 ms for 10 s, or packet loss is >2%.
- A follower is beyond the replication-lag bound for >5 s.
- A gateway or relay is a 2σ outlier on error rate.

Draining means leadership transfer off the node, weight 0 for new sessions, and ≤30 s to complete. The bound from onset is therefore ≤50 s (20 s detect + 30 s drain) for fast failures and ≤5 min for slow burns. It reuses the failover path, and a drained region still serves as a follower.

**Error budget (259.2 s per 30 days)**

| Cause | Allocation | Basis |
|---|---|---|
| Region events | 60 s | One cap-bound peak us-east event (43.4 s) plus one average-load eu-fra event at the cap (≈12.1 s = 10.8 s from the table + ≈0.8 s idle-join waits + ≈0.5 s remote-editor effects) = 55.5 s |
| Deploys/config | 40 s | 5 deploys × 2% cell × ≤5 min at full failure ≈ 30 s, plus margin |
| Control plane (directory, KMS, DNS, edge) | 40 s | Assumed 4 incidents per month at 10 full-outage seconds each. The directory outage alone costs <0.1 s (joins are 0.4% of requests × 0.1% failed × 1,800 s) |
| Gray failures/brownouts | 50 s | ≤50 s onset-to-drain at a 15% region share ≈ 7.5 s per event, so about 6 events per month |
| Provider/dependency incidents | 40 s | Assumed one incident per month degrading 20% of requests for 200 s |
| Unallocated | 29.2 s | |

### 4B. Poison-pill edit (crashes every receiving client)

Plaintext and E2EE documents get separate guarantees. **Server-verified quarantine (tier V)** is kept separate from **client-reported provisional skips (tier R)**. A single pill and a malicious author are handled separately.

**Client pipeline threat model** (every stage a remote batch passes through on a receiver)

| Stage | Plaintext: server-side prevention | Detection and containment (both modes) |
|---|---|---|
| Decode / decrypt | Structural limits (depth, 64 KB, rate), per-doc capability gating so new op types never reach old clients | Marker and overlay (below) |
| CRDT apply | Native apply on a copy-on-write overlay; 1% of ops (all high-risk plus a 0.1% sample) shadow-applied in WASM for all 13 builds, 20 ms timeout (a hang counts as a crash) | Marker; the overlay is discarded on a crash |
| Layout, text shaping, render | Not verifiable per op at ingest. On a report, the replay includes a headless layout and render of the op's anchor region | Marker stays set until the first frame containing the batch is presented |
| Asset decode | Embedded assets decoded in a sandbox at upload | Asset decode runs in a child process. A crash marks the asset `quarantined` locally |
| Snapshot load | Snapshots are cut by the voters from the validated log | Safe mode loads the previous trusted snapshot plus the tail minus the suspect batch |

- Limit: bugs that need context outside the shadow-apply window, interactions with other ops, or device-specific render crashes are not prevented. Detection below covers them.
- CI fuzzes decode, apply, layout and render on all 13 builds.

**Prevention: E2EE docs.** There is no semantic prevention, because the server sees ciphertext. The server validates the envelope only: device signature against the ACL, size, rate, dedupe, counter order, key epoch.

**Detection and containment (client)**
- **Atomic batch apply.** Each remote batch is applied to a copy-on-write overlay and rendered from it. It is committed to the persistent replica in one local transaction only after the first frame containing the batch has been presented (layout, shaping, render and asset decode complete). A crash anywhere on that path leaves the persistent replica at its pre-batch state, so bisection starts from clean state.
- **Marker.** Before applying, the client writes `applying:[opIDs]` with a plain write and no fsync, and clears it at the same commit point (the first presented frame). It survives a process crash through the OS page cache, including a crash in layout or render. After power loss it is simply absent and the restart counts as clean. The cost is ≈10 µs on the receive path (inside the 15 ms apply budget) and no extra flash write per batch on mobile.
- A restart with the marker set counts as a crash. After 2 crashes the client enters **safe mode**: read-only, not rendering the suspect batch, loading the last trusted snapshot plus tail minus the batch, and bisecting the batch in a child-process sandbox that includes layout and render of the anchor region.
- It reports `suspect(opID, clientVer, stackHash)` (envelope-only data). Restart backoff bounds the crash loop to 2 iterations, and the report is persisted locally so that a crash before upload does not lose it.
- **E2EE local skip:** after bisection, the device skips the culprit locally and uploads a signed `LocalSkip`, within ≤10 s. This works with no server help.

**Mitigation (server)**
- **Plaintext docs, tier V (server-verified):**
  - On a report, the DSO *replays* the op in the sandbox on the reporter's build: decode, apply, then headless layout and render of the anchor region.
  - If it crashes there, `Quarantine{opIDs}` is appended immediately. One report suffices, and collusion cannot fake a crash that replay verifies.
  - If replay does not reproduce (device-, GPU- or font-specific render crashes), the server does not quarantine. The reporter's `LocalSkip` follows the tier-R path below. A final quarantine needs ≥3 devices from ≥2 users, or ≥10% of sessions in 60 s, or the owner or a doc admin.
  - Small plaintext docs (≤3 sessions) therefore get an unattended quarantine for reproducible pills. For non-reproducible pills they get the tier-R provisional skip, with final quarantine by the owner or a doc admin.
- **Tier R (client-reported provisional skip; E2EE docs, and non-reproducible plaintext pills).**
  - A signed `LocalSkip` from a current member device is committed to the log (≤0.2 s) as a *claim*, not a command.
  - Every device that has not applied the op **verifies the claim itself**. It trial-applies the op (decode, apply, headless layout of its anchor) in a sandboxed child process with a 2 s timeout, and honors the skip only if the trial crashes or hangs. If the trial succeeds, it applies the op normally and reports `SkipRefuted(opID)`, which counts against the skipping device.
  - A fabricated skip therefore cannot suppress an op on any device where the op applies cleanly. A real pill crashes the trial on vulnerable builds, and a build on which it applies cleanly correctly applies it.
  - Per-device skip quota: ≤5 skips per doc per hour. A device with ≥2 refuted skips loses skip rights in that doc.
  - A single device in a small doc protects every later and offline arrival within ≤15 s of its first crash (≤10 s local + ≤5 s upload), plus ≤2 s per receiving device for the trial.
  - Devices that already applied the op recover through their own bisection (≤10 s). The owner sees each skip and can revert it.
  - Making the skip final (and the snapshot trusted) needs ≥3 devices from ≥2 users, or an owner or admin action.
- **Author containment (both modes).**
  - A per-device **first-write limiter** applies to docs the device has not written in the last 10 min: a token bucket with burst 2, refill 1 per 10 s (≤6/min), and ≤120/hour. Service accounts get separately approved limits and are covered by the strike rule.
  - A device's first strike drops it to 1 op/s. A *strike* is a quarantined op (plaintext, tier V), or a `LocalSkip` naming its op that ≥2 other devices confirm by their own crash or trial (E2EE, attributable from the envelope device pseudonym).
  - A second strike in a different doc within 24 h write-suspends the device across the tenant until an admin clears it.
- **Quarantine semantics:** a grow-only skip-set with `appliesFrom`, treated as "never applied". Text ops anchored to characters inserted by the pill re-anchor to the pill's recorded left origin (nearest surviving ID in the skeleton). Move and property ops depending on the pill become no-ops or apply against the prior value. Canvas references (connectors, instances, comment anchors) follow the dangling-reference rule of §2. These rules are in the convergence tests.
- **Snapshot ordering:** a snapshot is `trusted` only after a 15 s hold with no suspect reports (30 s for E2EE, plus its device signatures). The hold is a heuristic. Safety rests on two other rules:
  - A Quarantine invalidates any snapshot whose range contains a quarantined op.
  - Sync rejects a snapshot whose skip-set hash lacks the quarantine set, and I5b waits 30 days before any fold deletion.
  - A stale snapshot therefore cannot reintroduce the pill.
- **Sync order:** quarantine records and provisional skips are sent *before* ops, so late and offline clients never apply the pill.
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

- **Bounds, plaintext (tier V):** design p99 ≤15 s, with a hard cap of 120 s (2 crash cycles ≤8 s, report delivery with retries ≤30 s, replay ≤1 s, commit ≤0.2 s).
- **Bounds, tier R (E2EE and non-reproducible):** the device-local skip takes ≤10 s. The provisional doc-wide skip is committed ≤15 s after the first crash and honored ≤2 s after receipt on each device. Final quarantine takes ≤120 s with ≥3 devices or an owner action. With one device and no admin the skip stays provisional but protective, and the range stays untrusted for snapshots until an owner confirms.
- **Blast radius:** one doc per pill (up to 5,000 sessions), plus author containment. With burst 2 and a refill of 1 per 10 s, a malicious author reaches ≤2 docs in plaintext: the third first-write is not admitted until 10 s after the second, and the replay-verified strike lands at ≈7 s. In E2EE it reaches ≤8 docs (burst 2 + 6/min × a 60 s strike window).

## 5. Capacity and cost

### Reference arithmetic

**Load.** Peak sessions are 5M (NA 1.75M, EU 1.25M, APAC 1.5M, LATAM 0.3M, MEA 0.2M).
- At 3 sessions per active doc, 1.67M docs are live at peak. 40% of them are edited (§0 assumption): 670k docs with 1.5 editors each, so 1.0M editors.
- That gives 1.0M msg/s × 300 B = 300 MB/s in. With 2 receivers each, fan-out is 2M msg/s = 600 MB/s.
- Presence from 2.5M sessions at 0.5 Hz × 60 B is 75 MB/s in and 150 MB/s out.
- Joins are 5M ÷ 1,200 s = 4.2k/s × 18.5 KB = 77.7 MB/s.
- Peak egress is 600 + 150 + 77.7 = 827.7 MB/s, and the 50% diurnal average is 413.9 MB/s.

**Offline load (5% of DAU/day).**
- That is 2.5M syncs/day averaging 100 KB up and 1 MB down.
- Egress is 28.9 MB/s average.
- CPU is 58 syncs/s at the 2× diurnal peak × 100 ms ≈ 6 raw vCPU.
- Ingest is 7.5 TB/month (+2.8% of durable bytes).
- The server keeps only a device-registry row (≈0.5 KB × 150M devices = 75 GB, inside the metadata line). The client pulls from the immutable log, so there is no per-client sync state.
- **Long-offline cohort:** 50k users with a 14-day mean episode reconnect 30/14 ≈ 2.1 times per month, about 107k reconnects. That is ≈2.1 TB up (× 20 MB) and ≈0.5 TB down (× 5 MB) per month, which is negligible.

**Routing assumption.** A cross-region doc has 1 of 3 collaborators remote, so every op crosses a region boundary exactly once. That is 10% of ops, or 100k msg/s, 30 MB/s peak, 15 MB/s average, ≈38.9 TB/month.
- v1 serves it by direct connection to the home gateway. Those bytes are already inside the 600 MB/s fan-out and priced as internet egress, so there is no separate line.
- From P3, unpinned tenants (75% of the cross-region bytes) take the relay path, which adds one backbone leg at $0.02/GB. That is priced as its own line below.
- **Egress accounting.** All client-bound bytes (fan-out, presence, joins, offline sync), including those to remote-region clients, are priced at the internet egress rate. The inter-region lines are server-to-server only.

**Log growth and projection.**
- Durable ops average 105 MB/s (1.0M × 210 B × 50%), which is 9.07 TB/day or 272 TB/month raw.
- The 30-day full-fidelity tail is ≈272 TB raw (27% of the 1 PB logical total).
- Older history folds ≈10× (assumed; 5–20× in the sensitivity table), leaving +27 TB/month of retained logical history.
- The scenario's 1 PB is taken as month 0. Without expiry, logical size is 1.16 PB at month 6 and 1.33 PB at month 12 (+33%). That is +152 TB stored × $72.45 = **+$11.0k/month by month 12** (+4.1% of the model), outside the steady-state base below.
- Version thinning (daily to weekly checkpoints after 12 months) is available and not credited.

**Storage assumptions**
- **Stored fraction:** 0.465 (0.9/2.5 + 0.1 × 1.05), so 1 PB logical is 465 TB stored. E2EE ciphertext is incompressible with 5% envelope overhead. CRDT metadata is inside the 500 KB (§0).
- **Hot set:** 160M docs × 145 KB = 23.2 TB per copy, platform-wide. The 145 KB is a 125 KB uncompressed snapshot (25% × 500 KB) plus a 20 KB mean unfolded log tail and overlay index.
- **Search index:** 35% of the 225 TB logical non-E2EE current state (25% × 0.9 PB) = 78.75 TB.
- **Snapshots.** Edited-doc episodes are 0.83M average active docs × 1,440 min ÷ 34 min mean episode ≈ 35M/day, and 40% of them are edited (14M/day).
  - Hot snapshots of plaintext docs are materialized by each voter from the log: 670k docs ÷ ≈840 s ≈ 800/s peak per voter × 2 ms × 3 voters = 4.8 raw vCPU, computed for all docs (the E2EE share, cut by clients instead, is not credited). This causes no network traffic. SSD writes are ≈100 MB/s per replica fleet-wide, and SSD is priced by capacity.
  - Cold snapshots are written at deactivation: 14M/day × 30 = 420M/month × 125 KB × 0.465 = **24.4 TB/month**, packed into the segment objects. They are copied to the buddy and WORM copies, and the superseded ones feed the backup churn below.
  - **E2EE client snapshots** are uploaded as log entries (§1), one per edited episode: 10% × 14M/day × 30 = 42M/month × 125 KB × 1.05 = 5.5 TB/month. Raft replicates them to 2 remote followers = **11.0 TB/month** of inter-region traffic (+$0.2k). Their stored bytes are already in the 0.465 stored fraction and the cold-snapshot line.
- **Backup (1.15×):** one full WORM copy (465 TB) in a separate account and a distinct region of the same jurisdiction, plus two tails.
  - Fold-deleted tail segments are held 5 days past primary deletion: 126.6 TB/month × (1 − 1/10) = 114 TB/month deleted by folding, × 5/30 = 19 TB (4.1%).
  - Deletion and supersession churn is 43.2 TB/month × 35/30 = 50.4 TB (10.8%). That is 24.4 TB/month of superseded cold snapshots plus 18.8 TB/month of purged docs and replaced checkpoints (an assumed 4.0% of stored bytes/month). The lock is 35 days.
- **WORM copy:** written as one bundle per leader region per 60 s with a per-group offset index. It is write-once and never read on the normal path.
- **Restore:** WORM lags the log by ≤60 s. A restore after a double-region loss can lose acked ops newer than the last flushed bundle, so RPO ≤60 s in that case. This is outside RPO ≈ 0 and listed in §3, item 3. EU restore is ≈6.5 h.
- **WAL NVMe:** flush within 60 s or 384 KB per group, then 1 h retention for follower catch-up. Capacity is 210 MB/s × 3,600 s = 756 GB per replica × 3 replicas × 2 zone mirrors = 4.5 TB, ÷0.7 = 6.5 TB.
- **Memory:** priced at 4 GB/vCPU (13.2 TB provisioned). The working set is ≈0.2 TB (sessions 40 GB, active DSO state ≈84 GB × 1.3, WASM pools ≈1 GB per DSO host), so the allowance has 60× headroom.
- **Request pricing** is not in the reference list. It is assumed at $5 per million PUT and $0.40 per million GET.
- **Metadata (2 TB, derived):**

| Component | Basis | TB |
|---|---|---|
| Doc rows | 2B × 300 B | 0.60 |
| ACL and membership | 2B × 3 × 100 B | 0.60 |
| User→docs reverse index | 6B entries × 40 B | 0.24 |
| Device registry | 150M × 0.5 KB | 0.08 |
| Segment manifests | ≈12 months of bundles | 0.30 |
| Subtotal × 1.1 index/version overhead | 1.82 × 1.1 | 2.0 |

**Raw peak vCPU**

| Component | Derivation | vCPU |
|---|---|---|
| Gateways and relays | Keepalive 5M × 1 Hz × 10 µs = 50; connect 4.2k/s × 2 ms = 8; edit in 1M × 30 µs = 30; edit out 2M × 30 µs = 60; presence 3.75M/s × 25 µs = 94 | 242 |
| DSO | 1M × 150 µs | 150 |
| Raft follower apply | 2 × 1M × 40 µs | 80 |
| Join / ACL / API | Joins 4.2k/s × 17.6 ms = 74; offline syncs 6; ACL/API/auth 1.5k/s × 20 ms = 30 | 110 |
| Snapshot / compaction | Snapshots 4.8; zstd 0.7; fold 105 ÷ 50 MB/s = 2.1; manifest/GC/verify 2.7 | 10.3 |
| Shadow-apply (plaintext) | 10k ops/s × 13 builds × 150 µs | 19.5 |
| Search index + query (provisional until P3 benchmarks) | Index 670k docs ÷ 30 s = 22.3k/s × 3 ms = 67; merge 34; queries 8.3k/s × 10 ms = 83 | 184 |
| Auth, KMS, directory serving, telemetry agents | Provisional | 36 |
| **Total** | 831.8, rounded up | **832** |

**Provisioning:** 832 × 1.5 (platform overhead) ÷ 0.40 = 3,120, plus the 187.2 skew uplift from §4A = **3,307 vCPU** (37.7% steady). The N+1 premium over a 70%-utilization design is 3,307 − 1,783 = 1,524 vCPU = $43.9k/month. That is the $38.5k uniform-basis figure plus the $5.4k skew uplift, and the EU share is $10.2k. These are the figures used in §3.

### Monthly cost (720 h; vCPU-month = $28.8; every line derived)

| Line | Derivation | $/month |
|---|---|---|
| Compute | 3,307.2 × $28.8 | 95,247 |
| EU pilot-light spare | 75 vCPU × $28.8 = 2,160 + 8.3 TB NVMe × $100 = 830 | 2,990 |
| Internet egress | (413.9 realtime + 28.9 offline sync) = 442.8 MB/s × 2.592M s = 1,147.7 TB × $80/TB | 91,819 |
| Inter-region | Raft: 272.2 TB × 2 remote followers = 544.3 (uncompressed). Segment (op bytes) copy to buddy: 272.2 × 0.465 = 126.6. WORM copy: 126.6. Cold snapshots to buddy and WORM: 2 × 24.4 = 48.8. E2EE client-snapshot replication: 11.0. Total 857.3 TB × $20/TB | 17,146 |
| Relay backbone leg (P3, unpinned tenants) | 75% × 38.9 TB × $20/TB | 584 |
| Hot SSD | 23.2 TB × RF3 = 69.6 ÷ 0.7 = 99.4 TB × $100 | 9,943 |
| WAL NVMe | 6.5 TB × $100 | 649 |
| Search index (primary and buddy, RF2) | 25% on SSD at RF2: 78.75 × 0.25 × 2 ÷ 0.7 = 56.25 TB × $100 = 5,625; 75% in object at 2 copies: 118.1 TB × $23 = 2,717. Not backed up: derived data, rebuilt in hours from restored content | 8,342 |
| Metadata / ACL / directory / reverse index | 2 TB × RF3 ÷ 0.7 = 8.57 TB × $100 = 857; directory 64 GB × 12 locations = 0.77 TB × $100 = 77 | 934 |
| Object (primary + buddy) | 465 TB × 2 × $23 | 21,390 |
| Backup (WORM, separate domain) | 465 TB × 1.15 × $23 | 12,299 |
| Object requests | PUT: 16,384 groups × 1,440 flushes/day × 30 = 708M per copy (an upper bound, since idle groups skip flushes), ×1.15 for size-cap flushes at busy hours = 814M, × 2 copies = 1.63B × $5/M = 8,140; WORM bundles 0.5M PUTs = 2; GET 100M × $0.40/M = 40 | 8,183 |
| E2EE single-device fold lag | 10% E2EE × 272 TB/month × 1.05 = 28.6 TB/month; 50% assumed single-device = 14.3 TB/month held 35 extra days = 16.7 TB × $72.45 | 1,210 |
| **Total** | | **270,736** |

**Normalized: $270.7k ÷ 50M DAU = $0.0054 per DAU per month.**
- Cost drivers are compute and spare 36.3%, egress 33.9%, storage 20.2%, inter-region 6.5%, and object requests 3.0%.
- The E2EE storage effect (+65 TB stored, +$5.6k including segment copies and snapshot replication) is already inside the object, backup, and inter-region lines via the 0.465 stored fraction.
- **Accounting note.** The cold snapshots share objects with the segments but are separate bytes, so the lines are additive: 126.6 TB of op bytes + 24.4 TB of snapshots = 151.0 TB per copy, for the buddy and for the WORM.
- Not modeled, and covered by the platform-services buffer below: observability, WAF/DDoS, CI/staging, support, KMS operations.

**Storage and traffic reconciliation (one table; every physical total ties to its cost line)**

| Item | Physical total | Cost line |
|---|---|---|
| Primary + buddy object | 465 TB × 2 = 930 TB | Object, 21,390 |
| WORM + tails | 465 × 1.15 = 534.8 TB | Backup, 12,299 |
| Hot SSD | 99.4 TB provisioned (23.2 TB × RF3 ÷ 0.7) | Hot SSD, 9,943 |
| WAL NVMe | 6.5 TB | WAL, 649 |
| Search | 56.25 TB SSD + 118.1 TB object | Search index, 8,342 |
| Metadata and directory | 8.57 + 0.77 TB | Metadata, 934 |
| Single-device E2EE fold lag | 16.7 TB | Fold lag, 1,210 |
| Server-to-server traffic | 857.3 TB/month (+29.2 TB relay leg from P3) | Inter-region, 17,146 (+584 relay) |
| Client-bound traffic | 1,147.7 TB/month | Internet egress, 91,819 |

**Optional second-provider EU-resident WORM copy** (provider-wide failure; restore RTO in hours, outside the SLO): storage 465 TB × 1.041 × $23 = $11,134, a stream of 151.0 TB (126.6 segments + 24.4 snapshots) × $80 = $12,080, and sampled restore verification of 0.75% (3.5 TB) × $80 = $280. That is **+$23.5k/month**, not in the base total.

### Latency budget (p99, ms; a budget, not a proof)

**Populations.** Latency claims are stated for a specific population, never pooled silently:
1. the aggregate in-region p99 over all deliveries in normal state (the requirement as written);
2. per-class p99 (region-homed vs foreign-homed docs, and per client-region pair);
3. failover state, separately from normal state;
4. the viewer tier, separately (§3 item 6).

The no-loss budget below is the p99 basis under an assumed ≤0.5% of intra-region deliveries needing a loss-recovery round. V4 measures actual percentiles with loss.

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

**Region-homed docs with editors spread over metros.** A doc is placed in a group whose leader sits at the minimax-RTT metro for its editors (§0). Leg = 26 + RTT/2, loss round = RTT to the leader (50 for a same-metro leg), pair = sender leg + receiver leg + 50 ms server-side.

| Home (leader) | Editor metros (RTT to leader, ms) | Worst pair, no loss | One loss round |
|---|---|---|---|
| LATAM (sao), MEA (uae) | single metro | 102 | 152 |
| EU (fra) | dub, sto (30) | 132 | 162 |
| NA (usc) | use (28), usw (38) | 140 | 178 |
| APAC (sin) | tyo (70), bom (58) | **172** | **242** |

**Foreign-homed pairs on an EU-homed doc** (one loss round adds the larger leg's recovery R). Leg values (one-way p99 L / loss round R): EU 26/50, us-east 70/80, us-west 100/160, sao 115/190, uae 80/120, bom 100/130, sin 115/165, tyo 145/251. The pair formula uses a symmetric 26 ms EU leg, so it differs from the hop table by ≤1 ms. "EU ↔ EU" here is a same-metro pair (102/152). The multi-metro EU case above (132/162) is a region-homed pair with a 30 ms RTT to the leader.

| Pair | No loss | One loss round |
|---|---|---|
| EU ↔ EU (same metro) | 102 | 152 |
| EU ↔ us-east | 146 | 226 |
| us-east ↔ us-east | 190 | 270 |
| us-west ↔ us-west | 250 | 410 |
| uae ↔ uae | 210 | 330 |
| bom ↔ bom | 250 | 380 |
| sin ↔ sin | 280 | 445 |
| sao ↔ sao | 280 | 470 |
| tyo ↔ sao | 310 | 561 |
| **tyo ↔ tyo (worst)** | **340** | **591** |

- **Cross-region:** the worst pair of any kind is tyo↔tyo at 591 ms (both clients in the same region on a foreign-homed doc), or 631 ms with a +40 ms allowance for p99 queueing at 82–85% CPU. The worst pair with clients in *different* regions is tyo↔sao at 561 ms (601 ms with the allowance). The margin to 800 ms is ≥169 ms either way. Other home regions have legs of the same magnitude, and V4 measures every pair on every home.
- **In-region, region-homed docs:** the worst no-loss pair is 172 ms (APAC tyo ↔ tyo), which meets 200 ms with 28 ms of margin. With a stacked loss round, APAC multi-metro reaches 242 ms, NA 178 and EU 162. APAC multi-metro is therefore a listed exception in §3, item 5.
- **Exposure to 200 ms misses.** Foreign-homed same-region pairs (190–340 ms no loss) are conservatively all counted as exposed. Only us-east↔us-east (190 ms) is under 200 ms on the no-loss budget.
  - Assume 25% of cross-region docs have ≥2 remote collaborators in one region. That is 10% × 25% = 2.5% of docs.
  - In such a doc, 2 of 3 senders are remote, and each has 1 of 2 receivers in the same remote region, so 1/3 of its deliveries are exposed. That is 2.5% × 1/3 ≈ **0.8% of all deliveries**.
  - The stacked-loss tail on APAC multi-metro docs (30% × 15% × 1% ≈ 0.05%), failover windows (120 s ÷ 2.59M s = 0.005%) and v1 LATAM add ≤0.05%.
  - Total exposure is ≈0.85%, so ≥99.15% of deliveries are in classes that meet <200 ms on the no-loss budget.
  - **Aggregate claim:** the aggregate in-region p99 is <200 ms because the exceeding fraction is <1%. The margin is only 0.15 percentage points, so V4's pass criterion is a measured exceedance <1%.
  - **Per-class claim:** the listed classes do *not* meet 200 ms individually, and are not claimed to.
- LATAM clients served from us-east during a sao loss: 152 (in-region with a loss round) + 115 ms RTT ≈ 267 ms.

### Workload distributions (assumed and modeled)

| Missing distribution | Treatment |
|---|---|
| Doc popularity / members per doc | Zipf, exponent 2.2, untruncated: mean 3.75 members (the reference's 3 is used for the base model), p99.9 ≈ 190, P(≥5,000) = 2×10⁻⁵ (≈40k docs). The product caps live sessions at 5,000. About 200 of the 40k are live at a given peak (0.5%, the active share of hot docs). Assume 2.5% of those, 5 docs, are at the cap together (an all-hands event). That adds ≤89 MB/s under policy (c) (5 × 17.85); the stress case is 10 docs |
| Maximum size | 25 MB document state and 64 KB op, paste chunked. Binary assets are separate content-addressed objects of ≤25 MB. These are product limits, tested against Docs-class and Figma-class workloads in P3 |
| Log growth | Projected above (+33% logical by month 12) |
| Reconnect bursts | Failover (§4A), release, mobile-carrier outage, Monday 9 am local. Ticketed admission, jitter, IP-literal fallback |
| Offline size and duration | Modeled above |
| E2EE share | 10% of docs and bytes |
| Hot-doc sequencing load | Per-doc cap of 500 msg/s (2.5% of group append capacity) and an isolation trigger at ≥250 msg/s for 60 s, by online handoff to a single-doc group (§1) |

### Sensitivity (change in $/month vs base)

| Assumption | Low | High |
|---|---|---|
| Per-op CPU (340 raw vCPU on the edit path: gateway in/out 90, DSO 150, follower apply 80, shadow 20; ×0.5 / ×2; $114.5 per raw vCPU-month = 3.975 provisioned × $28.8) | −19.5k | +38.9k |
| Presence share of sessions (25% / 100%) | −13.2k | +26.3k |
| Search CPU (provisional 184 raw; ×0.5 / ×2) | −10.5k | +21.1k |
| Offline payload per sync (0.25 MB / 4 MB) | −4.5k | +18.0k |
| Compression (3.0:1 / 2.0:1) | −5.1k | +7.7k |
| Hot fraction (5% / 15%) | −3.7k | +8.7k |
| CRDT metadata *outside* the 500 KB (×1.3 / ×2.0 on the text-bearing 60% of logical bytes, applied to the object, backup and hot-SSD lines) | 0 | +7.9k / +26.2k |
| EU byte share (25% → 35% of platform bytes): spare NVMe 8.3 → 11.6 TB; EU hot reseed 20 → 27 min, cold reseed 6.5 → 9.0 h | 0 | +0.3k |
| Edited docs (670k clustered vs 815k independent placement) | 0 | +2.6k |
| Memory (4 → 8 GB/vCPU at an assumed $1/GB-month DRAM; not needed at the 0.2 TB working set) | 0 | +13.2k |
| Cross-region share (5% / 20%) | ≈0 in v1 (direct routing, bytes already in the egress line). P3 relay leg: −0.3k / +0.6k | |

- **Fold ratio** (added by month 12 on top of the base, absolute, not a delta): 20× adds $5.5k, 10× adds $11.0k, and 5× adds $22.0k.
- Per-op CPU, presence, search CPU and CRDT metadata dominate the uncertainty. All four are re-measured before the ceiling becomes a commitment (V14).

### Heavy-tail scenario 1: 5,000-session document (one byte basis: 300 B wire, 60 B presence)

**(a) Naive, uncapped (5,000 editors, 20% typing, 50% sending presence)**
- Edits: 1,000 typists × 300 B = 300 KB/s per receiver × 5,000 = **1.5 GB/s**, which is 2.5× the platform's entire 600 MB/s edit fan-out peak.
- Presence: 2,500 senders × 0.5 Hz × 60 B = 75 KB/s per receiver × 5,000 = 375 MB/s.
- Total 1.875 GB/s ≈ **$540/hour** for one doc.
- This and (b) are hypothetical counterfactuals. 1,000 typists send 1,000 msg/s, which the production 500 msg/s cap would stretch to a 2 s batch interval, so neither is a serviceable operating point under policy.

**(b) Uncapped, with the mitigation stack (counterfactual)**
- Viewport-scoped fan-out (≈5% of ops per receiver) is delivered on the editor tick: 50 msg/s × 300 B = 15 KB/s × 5,000 = 75 MB/s.
- Presence is capped at the 50 nearest cursors: 1.5 KB/s × 5,000 = 7.5 MB/s.
- Total 82.5 MB/s ≈ $23.8/hour, 23× cheaper than (a). It measures what the mitigation stack saves and is not evidence that the capped system serves this workload. Off-viewport ops are applied on scroll (≤300 ms after scroll), which is listed in §3 as unmet item 6.

**(c) Policy: 100 editor slots, 4,900 viewers (the P3 serving policy, plaintext docs only)**
- Editors: 100 slots, all typing as a stress case (the 20% rule would give 20 typists and 0.6 MB/s): 100 × 300 B × 100 receivers = 3.0 MB/s.
- Viewers: viewport ops 5 msg/s × 300 B = 1.5 KB/s × 4,900 = 7.35 MB/s.
- Presence 7.5 MB/s.
- Total 17.85 MB/s (143 Mbps) ≈ **$5.1/hour**, about 105× cheaper than (a).
- A one-level relay tree of 50 relays × 100 clients keeps DSO egress at 1.5 MB/s (full edit stream, 100 × 300 B = 30 KB/s, to each of 50 relays) plus ≈0.3 MB/s presence aggregation, ≈1.8 MB/s (14 Mbps).
- The tree's only extra hop is the DSO → relay hop already in the budget (+3 ms). In-viewport p99 stays ≈105 ms, under 200 ms.
- A 5,000-client meeting-start burst at 500 joins/s serves a 100 KB viewport-first chunk each: 50 MB/s for 10 s, with the rest lazy.
- This is the P3 state. v1 serves ≤1,100 sessions per doc (§6).
- **Relay compute.** Per doc the relays deliver ≈160k msg/s: 24.5k viewport ops (5/s × 4,900), 10k editor deliveries (100 × 100), and 125k presence updates (25/s per receiver × 5,000).
  - At 30 µs per edit message and 25 µs per presence message that is 34.5k × 30 µs + 125k × 25 µs ≈ 4.2 raw vCPU per doc, or **≈21 raw vCPU for five docs**.
  - If provisioned permanently at the event peak, that is 21 × $114.5 = **$2.4k/month**. It is not inside the base gateway/relay line, and it is priced in the "Combined with the base" paragraph below.
- **Hot-shard isolation:** this policy's doc generates ≈100 msg/s, below the 250 msg/s trigger and the 500 msg/s cap. The cap bounds its append rate to ≈150 KB/s, or 2.5% of group append capacity.

**(d) E2EE variant (cap 100 editors + 900 viewers = 1,000 sessions).** There is no viewport scoping, so all 1,000 sessions get every op: 100 typists × 300 B × 1,000 = 30 MB/s, plus presence 500 senders × 0.5 Hz × 60 B × 1,000 = 15 MB/s. The total is 45 MB/s ≈ $13/hour. A ciphertext-forwarding relay tree (10 relays × 100) keeps DSO egress at 0.3 MB/s. E2EE docs do not reach 5,000 sessions (§3, item 14).

**Combined with the base.** Five (c) docs add 89 MB/s at the event peak.
- Even sustained 24×7 at the 50% diurnal average, that is 44.6 MB/s × 2.592M s × $80 = $9.2k/month of egress.
- Add $2.4k/month of relay CPU provisioned permanently at the peak: **$11.6k/month** in total.
- This is a separate increment, inside the 10% buffer below (≈$27k).

### Heavy-tail scenario 2: reconnect storm

The surge table in §4A covers all 11 regions on unequal capacity. Peak survivor CPU is ≤82% before shedding and ≤68% after. The in-region queueing effect at those levels is ≈3.7 ms, inside the 40 ms allowance. The five large docs' viewers (≈25k sessions) are P1 sessions already inside the idle drain.

### Budget ceiling

**$0.0090 per DAU per month ($450k/month), 1.66× the model.** The buffers compound, since each covers a different uncertainty class: 1.20 × 1.10 × 1.25 = 1.65, and $270.7k × 1.65 = $446.7k, rounded up.
- +20% for unmodeled platform services (observability, WAF/DDoS, CI/staging, support).
- +10% for drills and heavy-tail surges ($11.6k sustained heavy-tail increment plus drill traffic).
- +25% for price and estimate error. It covers the two largest high-side sensitivities together (+$38.9k per-op CPU and +$26.3k presence = $65.2k = 24%).
- If the second-provider copy is enabled (+$23.5k), the buffers shrink to 53% ((450 − 294.2) ÷ 294.2), and the ceiling still holds.
- **Status of the ceiling.** It is a planning ceiling. It becomes a commitment only after P3, when per-op CPU, presence share, search CPU, CRDT metadata inclusion, EU byte share, E2EE size mix and backup churn have been measured (V14).

**Unit economics (an assumption, since the prompt gives no revenue model).** With DAU/MAU = 30% there are 167M MAU. At 3% paid conversion of MAU × $10/month, revenue is $50M/month ($1.00 per DAU-month). The ceiling is then 0.9% of revenue, and the modeled cost is 0.54%. A 10%-of-revenue infrastructure cap would allow $0.10 per DAU-month, so the ceiling is set by estimate error, not by affordability. The defense does not depend on the revenue figures.

If we breach the ceiling, the levers are:
1. Tighter coalescing and viewport sampling. A 10% egress cut saves $9.2k.
2. Committed-use or private-interconnect discounts on compute and egress (69% of spend; not in the reference prices).
3. Compressing Raft replication in transit (−$5.8k at the 0.465 ratio).

**Not a lever:** raising steady utilization above the §4A per-set sizing. At 50%, survivors would pass 75% before the surge. A us-east loss would then exceed 100% under the surge (an eu-dub loss would peak at ≈95%).

## 6. Cuts and roadmap

**v1 is the end of P1 (month 6): NA and EU tenants only.** The end-state design in §§0–5 meets the fixed requirements to the extent stated in the summary status table and §3 "cannot be met". v1 intentionally does not, and is not offered as compliant. Between P2 and P3 the platform meets offline and compatibility but still serves NA and EU only.

| Cut | Restored in | Requirement affected until then |
|---|---|---|
| APAC and MEA not served (34% of DAU); LATAM docs home in the NA voter set; no sao/uae compute or voters | P3 (month 12) | Global availability. LATAM in-region SLO unmet (≈270 ms) |
| Offline 7 days; client window 6 months | P2 (month 9) | 30-day offline and 12-month compatibility **unmet** |
| Cap of 100 editors and 1,000 viewers (≤1,100 sessions per doc); no relays for unpinned docs | P3 | Heavy-doc analysis in §5 is the P3 state |
| No E2EE (search S1, tenant recovery) | P4 | E2EE requirement unmet |
| No S2 SSE search | P5 (optional) | None; S1 meets the scoped "E2EE with search" |
| No HYOK (Shamir recovery key only) | P5 | None |
| No doc re-homing; drifted docs stay in their placement class | P3 | Per-class latency exceptions (§3 item 5) |
| Canvas limited to frames, shapes, and LWW properties. Vector paths, auto-layout, component variants, shared styles and rich typography are out of scope and would arrive as new op types through the compatibility mechanism | After P5 | Figma-class scope: this is a narrower collaborative canvas, not Figma parity |

**LATAM migration (P3).** Docs move from the NA set {use, usc, usw} to the LATAM set {sao, use, usc} per group by a single-member Raft change. Add sao as a learner, run joint consensus replacing usw, transfer leadership to sao in a low-activity window, then bump the directory epoch. There is no downtime and no acked-op loss, because ACL positions and E2EE key epochs travel in the log.

| Phase | Scope | Unsupported to users at this phase | Measurable exit criterion |
|---|---|---|---|
| P0 (0–3 mo) | NA only: text CRDT, DSO, 3-region Raft with the lag bound, leader-affinity placement, hot and cold snapshots, segments to two regions, WAL GC | All non-NA users; EU/US collaboration; offline; E2EE; docs over 1,100 sessions | p99 in-region ≤200 ms at 100k concurrent sessions in load test; V1 passes for I1–I4 and I5a (10⁶ simulation seeds, 0 violations); ≥100 staging region-kill drills with 0 lost acked ops and max RTO ≤120 s; a synthetic 300 msg/s doc is isolated to its own group within 90 s with 0 lost acked ops and a 600 msg/s doc is held at 500 msg/s |
| P1 (3–6) | EU jurisdiction, residency enforcement (IR and SR), EU/US collaboration, WORM copy, eu-par and replacement-voter procedure. **v1 ships here** | APAC and MEA; LATAM local latency; offline beyond 7 days; clients older than 6 months; E2EE; relays and docs over 1,100 sessions; canvas beyond frames, shapes and LWW | ≥100 staging EU region-kill drills plus 3 production drills, max RTO ≤120 s; 0 honeytoken hits in non-EU storage and logs for 30 days; EU byte share re-measured and EU hot reseed measured at ≥5 GB/s (≤30 min including provisioning at the 25% assumption); a membership-change drill promoting eu-par with 0 lost acked ops |
| P2 (6–9) | 30-day offline, compatibility CI over the full 12-month window, ActorFork handling, poison-pill quarantine (tiers V and R), trusted snapshots and fold GC (I5b), directory-kill drill | APAC, MEA and LATAM-local; E2EE; docs over 1,100 sessions | 10⁶ randomized 30-day-offline merges converge bit-identically; the clone-fork test loses 0 edits; compatibility matrix of 13 frozen builds cut from monthly protocol tags (N-12…N, with an accelerated schema-change corpus) green; V1 extended to I5b with 0 violations; V9 plaintext criteria pass; directory-outage drill keeps ≥99.9% of joins |
| P3 (9–12) | APAC, LATAM, MEA; editor slots, viewer mode, relays for unpinned docs; doc re-homing; LATAM migration | E2EE; S2; HYOK | V4 passes over 30 days of probes for every §5 pair and class, with measured aggregate exceedance <1%; reconnect drills (400k eu-dub, 750k us-east sessions) with max first submit ≤120 s across the §4A cohort; 5,000-session replay under policy (c) egress ≤22 MB/s and relay CPU ≤25 raw vCPU for five docs |
| P4 (12–15) | E2EE with client-side search, client snapshot upload, single-device delayed trust, tenant recovery, E2EE poison path | S2; HYOK | Third-party crypto audit with 0 critical findings; 3-of-5 recovery restores 100% of test DEK epochs including after key rotation; offline-across-rotation test 100% rule-conformant; no E2EE plaintext markers server-side; single-device docs reach trusted snapshots without blocking WAL GC |
| P5 (15–18) | Availability and cost hardening, optional S2, HYOK | The items in §3 "cannot be met" | 99.99% SLI in every rolling 30-day window across 90 consecutive days including ≥1 region drill; measured $/DAU ≤$0.0090 at scaled extrapolation; a real ≥12-month-old client build (the v1 build from month 6) passes V8 |

## 7. Verification plan

Statistics only bound failure rates and never prove them. The safety guarantees rest on the model-checked invariants (V1), and the tests check the implementation against them.

| ID | Area | Method | Pass criterion | Covers |
|---|---|---|---|---|
| V1 | Commit, fencing, GC safety, membership, timing | TLA+ spec of I1–I4, I5a, I5b, the joint-consensus replacement of a dead voter, the `TermStart` tentative-retraction protocol (no tentative entry reported committed; receivers converge to Sc ⊕ J′), the replication-lag bound, and the doc handoff (`MigrateOut`/`MigrateIn`). Model-checked on a 3-voter region-kill model that includes shared physical regions (use and usc in two sets), the 2-zone WAL mirror, a separate WORM account, manifest CAS and term-fenced conditional writes. A **timing model of the full RTO path**: detection → election → directory redirect → ticket scheduling → admission → reconnect → DSO activation → durable ack of the first submit. Deterministic simulation (10⁶ seeds with region kills, partitions, clock drift ≤200 ppm, process pauses longer than the lease, fsync lies), 10⁵ simulated elections, plus trace validation of the implementation against the spec. Gated by phase: I1–I4 and I5a in P0, membership in P1, I5b in P2 | 0 invariant violations; 0 lost acked ops; 0 dual-leader commits; max election ≤12 s; modeled max event-to-first-durable-submit ≤120 s for every cohort member | RPO ≈ 0 |
| V2 | RPO under region kill | Jepsen-style at ≥1.2× per-group reference peak, plus an overload run at 1.5×. Kill a random region mid-load and audit acked opIDs against the recovered log. Variant: hold the slower follower at the lag bound, then kill the fast follower's region and, separately, the leader's region | 0 missing in ≥300 staging kills plus 3 production kills; durable acks resume ≤0.5 s after the new leader is established in the lag variants. The rule-of-three bound (≤1% per kill at 95%) is reported as supplementary evidence only | RPO ≈ 0 |
| V3 | RTO (hard maximum) | ≥100 staging drills and quarterly production drills; ≥5,000 distributed probers including cellular and CGNAT. Server-side first-submit logs of **every cohort member** (§4A definition), with the clock started at T0 and never restarted. Fault variants: admission queue at 150% load, stale directory, DNS blocked, one slow follower held at the lag bound, NAT rebind, a 10× idle→edit promotion burst. Idle→edit probes at T0+10 s, +60 s and +117 s | **Max over the cohort of (first successful submit − T0) ≤120 s** (alert at 45 s); idle→edit keystroke-to-submit ≤3 s at any time. The bound is a protocol property (deterministic timers and ticket schedule, checked in V1's timing model), and drills confirm it. No percentile substitutes | RTO ≤2 min |
| V4 | Latency | ≥1,000 probers per region pair (every pair in §5, including multi-metro region-homed pairs and every placement class), PTP/GPS-synced stamps, load at 1.5×, also in failover state. Report separately: (1) aggregate in-region p99, (2) per-class and per-client-region-pair p99, (3) normal vs failover, (4) viewer-tier off-viewport applies. Measure the loss-round rate and re-fit the commit-latency factor from raw RTT, fsync and queueing distributions | Rolling 24 h aggregate in-region p99 <200 ms with a measured exceedance fraction <1%; region-homed per-class p99 <200 ms (no-loss budget); every cross-region pair <800 ms. The listed exception classes are *not* a pass of the 200 ms requirement. They are checked for conformance to their §5 values and the ≈0.85% exposure share is measured | p99 requirements |
| V5 | CRDT convergence and identity | Property tests and nightly fuzz: 10⁹ permuted interleavings including a virtual-clock 30-day offline, quarantine skip-sets and re-anchoring, schema versions N-12…N. LWW test: an offline client with 10⁵ ops versus one online edit of the same property, in plaintext and E2EE. Text test: a delete of a range vs a concurrent insert inside it. Move-tree tests: a skipped move that later becomes valid after an earlier-ordered move arrives, with the benchmark (10 µs/move assumption). **Clone test:** two offline clones with the same actorID and counter range both edit, then sync in either order, in plaintext and E2EE. E2EE causal test: out-of-order cross-device submissions with encrypted deps buffered | 0 divergent state hashes; 10⁶ randomized 30-day-offline merges bit-identical; the commit-position LWW winner is identical on every replica; the concurrent insert survives the range delete; move-tree results identical on all replicas and p99 ≤0.2 s at 20k moves; clone test: `ActorFork` detected, 0 lost edits, identical final state | Offline convergence |
| V6 | Residency | OPA policy-as-code on IaC; daily honeytoken docs searched in non-EU storage, logs, backups, heap dumps, and edge captures; BGP announcements for EU-pinned prefixes from EU PoPs only, confirmed by non-EU probes; egress allow-list audit; quarterly external audit; SR tenants checked for zero non-EU device sessions; strict-policy managed devices checked for empty caches after session end | 0 hits; 0 violations. Under IR the device-boundary exception is excluded from "hits" and tested only for policy-flag conformance; under SR any non-EU device session is a violation | EU residency |
| V7 | E2EE | Marker strings; crypto audit; recovery drill; offline-across-rotation test (client offline 30 days across ≥3 epoch rotations and one removal); S1 offline search; client snapshot upload with a mid-upload disconnect and recovery across all 3 voters; single-device doc through the 35-day delayed-trust path, including replay re-verification of a bad checkpoint | 0 plaintext hits server-side; 0 critical findings; 100% DEK recovery; 100% rule conformance; incomplete snapshot uploads ignored and collected within 24 h; single-device docs fold and WAL-truncate without blocking | E2EE |
| V8 | Compatibility | 13 builds (N-12…N) × servers N and N+1 on golden logs; live mixed-version docs where new op types are attempted; rolling-deploy test. **Bootstrap test:** an N-12 client performs the QUIC/TLS handshake, token refresh and signed-config refresh. **Snapshot test:** all 13 builds read the same snapshot and compute equal `stateHash`. A separate emergency-expiry test: the force-expired build is refused with an upgrade prompt, and all other builds are unaffected | Server log and segments round-trip byte-identical (hash compare). Client unknown fields preserved semantically (canonical decode equal), 100%. Old clients never crash and show fallbacks. Deploy error ≤0.001%. The emergency exception is verified as the *only* path that breaks the window | 12-month clients |
| V9 | Poison pill | 1,000 injected pills per class: large plaintext docs; small docs (≤3 sessions) with reproducible and non-reproducible pills; E2EE docs including single-device docs; **crash after apply during layout or render, after the local commit would have occurred, during snapshot load and during asset decode**; crashes from interactions of multiple ops; malformed assets; a reporter that crashes repeatedly before it can upload (the persisted report); partial-batch crash followed by safe-mode bisection; a malicious member submitting fabricated `LocalSkip` records against legitimate ops; a malicious author spraying pills across docs in a burst; 10⁴ pill/snapshot interleavings; 50% report loss and delay | Plaintext max ≤120 s (design p99 ≤15 s); small reproducible docs ≤60 s unattended; tier R: local skip ≤10 s (100%), skip committed ≤15 s and honored ≤2 s after receipt, final ≤120 s with ≥3 devices; a fabricated skip suppresses 0 ops on healthy devices and the skipper loses skip rights at the 2nd refutation; replica identical to its pre-batch state after any mid-pipeline crash; author suspended at the 2nd strike (≤2 docs plaintext, ≤8 E2EE under the burst-2 limiter); ≤2 crash-loop iterations; 0 untrusted snapshots used in recovery | Poison-pill analysis |
| V10 | Heavy tail and hot docs | Replay the 5,000-session doc and storms (us-east 750k, sin 600k, eu-fra 450k sessions), with scenario-specific criteria. (a) is diagnostic only (≈1.9 GB/s expected). (b) doc egress ≤90 MB/s, as a counterfactual. (c) doc egress ≤22 MB/s (176 Mbps), DSO egress ≤2.2 MB/s, relay CPU ≤25 raw vCPU for five docs. (d) E2EE variant (100 editors + 900 viewers) ≤50 MB/s. (e) hot doc: a synthetic 600 msg/s doc is capped at 500 msg/s and, at ≥250 msg/s for 60 s, isolated to its own group with 0 lost acked ops | In-viewport editor p99 <200 ms; off-viewport apply ≤300 ms after scroll; storm CPU ≤90% before shedding and ≤75% after (model: ≤82% / ≤68%); max first submit ≤120 s for the cohort; neighbors in the hot doc's group see no append-latency regression beyond the p99 budget | Heavy-tail model |
| V11 | Control plane, gray failure, revocation | Kill the global directory for 30 min while creating docs and opening them cross-jurisdiction, with and without invite hints; block DNS; drop one KMS region; inject 200 ms fsync stalls and 5% loss in one region; a slow burn at 0.5% errors; hold a follower beyond the lag bound. **Account-revocation drill:** revoke an account, then submit with the unexpired token, with the push working and with the push dropped, and during a jurisdiction partition | Join success ≥99.9% with the directory down; docs created during the outage routable at home immediately and reachable cross-jurisdiction via hints, with queued writes reconciled after recovery; 0 failed submits on existing sessions; drain ≤50 s (fast) or ≤5 min (slow burn); 0 acked-op loss; revoked-account writes rejected ≤30 s with push and ≤1 h worst case, fail closed in a partition, and writes in the window tagged `postRevocationWindow` | Failure analysis, §4A |
| V12 | Availability | Prober plus server-side SLI per the fixed definition: submit acked ≤5 s, join complete ≤30 s, expired ticket waits counted as failures. Includes deferred joins under 150% load, client timeouts, and monthly chaos (zone, region, partition), drills counted | ≥99.99% in every rolling 30-day window across 90 consecutive days; error-budget burn tracked per the §4A table. Presented as empirical SLO evidence, not a universal guarantee | 99.99% |
| V13 | Offline load | Replay 5% of DAU reconnecting at 100 KB up and 1 MB down, plus the long-offline cohort (20 MB up), key-epoch rotations, removals and owner-review inbox load | Egress and CPU within model +20% | Capacity model |
| V14 | Cost | Quarterly bill against the model, with sensitivity re-run. The inputs that gate the ceiling are measured first: per-op CPU, presence share, search CPU and queries, whether CRDT metadata is inside the 500 KB, EU byte share and per-region demand equivalence, E2EE size mix, Zipf tail, backup churn | Unit cost within ±15% of model and at or under the ceiling | Budget ceiling |
| V15 | Restore from WORM | Quarterly restore of a 1% sample of each jurisdiction from the WORM copy into an isolated cell, after simulated double-region loss | Restored docs hash-equal to the last flushed log; acked ops lost ≤ those newer than the last 60 s bundle; EU restore rate ≥5 GB/s measured (projects to ≤6.5 h at the 25% EU byte share) | Restore RPO, §3 item 3 |

**Traceability.** Each row of the requirement-status table maps to at least one V-row:
- Latency maps to V4 and V10.
- Offline maps to V5 and V13.
- Residency maps to V6.
- E2EE maps to V7.
- Availability, RPO and RTO map to V1, V2, V3, V11, V12 and V15.
- 12-month clients map to V8.
- The poison-pill analysis maps to V9.
- Cost maps to V14.
 

```


# TEST 

``` 
# High-Assurance Gateway Challenge v02

## Objective and time budget

Implement a small HTTP/1.1 API gateway whose security boundaries, rate limiting, and logging remain correct under adversarial input and concurrent traffic. The challenge is intentionally narrower than a production gateway: correctness and evidence matter more than feature count.

**Suggested time budget: 60 minutes.** Use Go and the standard library only. No external network access or dependency downloads are available during evaluation.

## Required submission

Submit exactly:

1. `main.go` — the complete implementation in one source file. Do not submit a multi-package project, generated code, vendored dependencies, or a custom test suite.
2. `DESIGN.txt` — at most 200 words explaining the key security invariant, the rate-limiting trade-off, and one known limitation.

The evaluator supplies backend fixtures, an ingress that injects trusted client identity, and the test harness. The evaluator does not award points for the sophistication of submitted tests or lengthy documentation.

## Runtime contract

The program must build offline with `go build -trimpath -o gateway main.go` and start with `./gateway`.

Configuration is supplied through environment variables:

- `GATEWAY_ADDR`: listen address; defaults to `127.0.0.1:8080`.
- `REPLICA_ID`: one of `0`, `1`, `2`, or `3`; required.
- `SEARCH_BACKEND_URL`: origin URL for Route A.
- `FILES_BACKEND_URL`: origin URL for Route B.
- `RESOLVE_BACKEND_URL`: origin URL for Route C.

Backend URLs are plain-HTTP origins with no path, query, or fragment. The evaluator launches four instances with distinct replica IDs. The gateway listens on a private test interface behind a trusted ingress. The ingress removes any client-supplied `X-Verified-Client-ID` and injects exactly one verified value. Use that value as the authenticated identity; do not trust other client-controlled identity headers. The identity must be 1–128 ASCII characters matching `^[A-Za-z0-9_-]+$`. If the verified identity is absent or invalid, fail closed with `401`.

Use only plain HTTP/1.1 for this exercise. TLS termination and HTTP/2 are out of scope.

## 1. HTTP behavior and limits

Implement these routes exactly:

| Route | Required behavior |
|---|---|
| `GET /v1/search?q=<term>` | Validate `q`, then proxy to the search backend. |
| `GET /v1/files/{file_id}` | Validate the one path segment, then proxy to the files backend. |
| `POST /v1/tools/resolve` | Validate the JSON contract, then proxy to the resolve backend. |
| `GET /healthz` | Return a minimal liveness response. |
| `GET /_internal/metrics` | Return bounded counters for logging reservation/delivery failures and current queue depth. |

Unknown routes return `404`; a known route with the wrong method returns `405`; invalid route input returns `400`; an oversized request returns `413`; a request without trusted identity returns `401`; an exhausted quota returns `429`; inability to reserve logging capacity returns `503`. Upstream timeouts return `504`; other upstream failures return `502`. Do not expose internal errors, stack traces, credentials, or backend diagnostics.

Enforce all of the following:

- At most 64 request header field occurrences (including repeated names) and 16 KiB of aggregate request headers.
- Request target no longer than 8 KiB.
- Request body no larger than 64 KiB.
- Upstream response body no larger than 1 MiB. A response that exceeds this limit must fail with `502` without buffering it unboundedly or sending a partial successful response.
- At most 512 requests may be in active application/upstream processing at once; reject excess work with `503` after reserving a log slot and without contacting a backend.
- Use `ReadHeaderTimeout` of 5 seconds, `ReadTimeout` of 10 seconds, `WriteTimeout` of 3 seconds, and `IdleTimeout` of 30 seconds. The total upstream request timeout is 2 seconds. Propagate client cancellation to upstream work.
- Strip hop-by-hop headers. Do not forward `Authorization`, `Cookie`, client-supplied identity headers, or other secrets. Preserve the method and the request semantics defined below. `Accept`, `Content-Type`, and `If-None-Match` are the only client request headers that must be forwarded.

For a successful upstream response within the size limit, preserve its status and body and forward end-to-end response headers after stripping hop-by-hop headers. Malformed HTTP streams rejected by `net/http` before a request object exists do not require an application log record. Do not add custom parser behavior that accepts malformed framing.

## 2. Security contracts

### Route A: Search

`GET /v1/search?q=<term>` accepts exactly one `q` parameter and no other query parameters. After exactly one query-decoding pass, the value must contain 1–64 ASCII characters and match:

`^[A-Za-z0-9 _-]+$`

Use standard form-query decoding (`+` decodes to a space). Reject malformed percent-encoding, duplicate `q` parameters, invalid characters, empty values, and values outside the length limit. Do not use a parser that silently ignores malformed query data. Forward a canonical, correctly escaped query containing exactly one `q` parameter.

### Route B: File retrieval

`GET /v1/files/{file_id}` accepts exactly one path segment. After exactly one path-decoding pass, the identifier must contain 1–64 characters and match:

`^[A-Za-z0-9_-]+$`

Reject dot segments, slash and backslash separators, malformed escapes, encoded separators, and double-encoding attempts. Validate the decoded identifier before forwarding it. Forward only a canonical single path segment; the backend must not receive a value whose meaning changes under another decode or normalization pass.

### Route C: Host resolution

`POST /v1/tools/resolve` requires `Content-Type: application/json` (optionally with `charset=utf-8`) and accepts a UTF-8 JSON object containing exactly one field named `host`. The decoded value must be a non-empty ASCII DNS-style hostname of at most 253 characters. Each dot-separated label must be 1–63 characters, contain only letters, digits, or hyphens, and not begin or end with a hyphen. A trailing dot is not accepted.

Reject invalid UTF-8, malformed JSON, non-standard constants, additional fields, duplicate object keys, and trailing non-whitespace JSON values. Duplicate detection must compare decoded member names, so `"host"` and `"h\u006fst"` are equivalent keys. Do not rely on `encoding/json`'s default duplicate-key behavior. Forward one canonical JSON object with one `host` field; lowercase the hostname when forwarding.

**Security invariant:** a request that fails its route contract must never be forwarded to the corresponding backend. The evaluator checks backend side effects, not merely the status code.

## 3. Four-replica rate limiter

Implement a cluster-wide per-client token-bucket envelope without a synchronous central service call per request. To keep this test focused, quota is statically partitioned across four replicas; dynamic quota transfer, replica restart recovery, and control-plane coordination are out of scope and must be acknowledged in `DESIGN.txt`.

Each replica owns one independent bucket for each verified client identity:

- Capacity: **5 tokens per client per replica**.
- Refill: **0.5 tokens per second per client per replica**.
- A newly observed identity starts with a full bucket on each replica. One route-valid request consumes one token before it is forwarded. Invalid route input is rejected and logged before quota is consumed.
- If no token is available, return `429`; include `Retry-After` when a reliable estimate is available.
- Use a monotonic elapsed-time source for refill calculations; wall-clock skew must not create extra quota.
- Concurrent requests must update a client's bucket atomically.

Across four replicas, the required cluster-wide envelope for any one client is:

`admitted(T) <= 20 + 2*T`

where `T` is the interval duration in seconds. The evaluator checks the envelope with controlled request timing and uneven as well as evenly distributed traffic. Under evenly distributed traffic, at least 95% of the quota made available by this fixed allocation must be usable before legitimate requests are unnecessarily throttled. Static sharding may strand quota under uneven traffic or replica failure; it must never create excess quota.

Bound per-replica client state to 10,000 identities. If the state table is full, only evict entries that are fully replenished; otherwise fail closed with `503` rather than resetting a depleted identity to a full bucket. Do not use unbounded maps, queues, goroutine creation, or attacker-controlled allocation sizes.

## 4. Structured logging with admission reservation

Write one JSON event to standard output for every successfully parsed application request for which log capacity was reserved, including requests rejected by route validation or rate limiting. `/healthz`, `/_internal/metrics`, and HTTP parser failures are exempt. Standard output must contain only JSON event lines; startup diagnostics go to standard error.

Each event must include a generated request ID, event ID, timestamp, method, normalized route template, status, duration, outcome/reason code, and a pseudonymous client identifier. Never log request bodies, raw query values, authorization data, cookies, secrets, or raw user-controlled values to either output stream. Escape and bound externally influenced values. Each event is at most 2 KiB.

- Maximum queue/reservation capacity: 8,192 events, including requests currently being processed whose event slot has been reserved.
- Reserve capacity before route processing or forwarding. If capacity cannot be reserved, return `503`, increment a bounded failure metric, and do not contact a backend.
- Once capacity is reserved, deliver exactly one event to the output stream. Do not silently drop a reserved event during normal operation.
- If the output stream blocks, queue growth and admitted work must remain bounded. Do not create a goroutine per event or request merely to hide backpressure. If a write fails, mark logging unhealthy, increment the delivery-failure metric, and fail closed for new requests rather than treating delivery as successful.
- On `SIGTERM`, stop accepting connections and dispatching new requests, including on existing keep-alive connections. Allow dispatched work to finish until 2.5 seconds after the signal, then cancel remaining work; attempt to drain reserved log events and exit within 3 seconds even if the output stream is stalled. Document best-effort behavior at the deadline.

## 5. Evaluation conditions

The evaluator, not the candidate, supplies and runs automated tests. Tests include:

- Valid requests, method/path behavior, query semantics, response forwarding, and upstream failures.
- Encoded and double-encoded path attacks, malformed percent escapes, duplicate query parameters, duplicate JSON keys including escaped-equivalent names, invalid UTF-8, and trailing JSON values.
- Inspection of backend side effects to confirm invalid requests were never forwarded.
- Concurrent requests across four replicas to verify the token-bucket envelope, identity isolation, and lack of races.
- Log-injection attempts, secret-leak checks, exact event counts, queue saturation, blocked output, bounded memory, and shutdown.
- A short open-loop load test at 2,000 offered requests/second for 30 seconds after a 5-second warm-up. The workload is 70% search, 20% file retrieval, and 10% host resolution, with a 0.25 ms fixture delay and 4,000 verified identities distributed evenly across the four replicas. The reference environment uses a one-core CPU quota and a 512 MiB container cap. The evaluator records offered/completed rate, p99 latency, unexpected errors, and peak RSS; the gateway must remain alive and at or below 256 MiB RSS.
- Five independent clean-environment runs of the functional and security suite. Any verified critical exploit is disqualifying.

The same versioned fixtures, harness, random seeds, CPU/memory limits, and software environment are used for all candidates. No candidate-generated benchmark claims are required.

## 6. What is intentionally out of scope

Do not implement TLS termination, HTTP/2, dynamic quota rebalancing, coordinator leases, replica restart recovery, 10,000-connection Slowloris defense, full fuzzing infrastructure, or a production deployment framework. The design note must not claim guarantees for these out-of-scope cases. 

``` 


--- 


# Evaluator Rubric 

``` 

# Private Evaluator Rubric — High-Assurance Gateway v02

**Confidential: do not provide this document to candidates.** Candidates receive `TEST-v02.md` only.

## Scoring overview

| Dimension | Weight |
|---|---:|
| Functional correctness and rate limiting | 25 |
| Security | 30 |
| Performance and resource efficiency | 15 |
| Reliability and logging | 20 |
| Safety and engineering judgment | 10 |
| **Total before any gate** | **100** |

Score only behavior reproduced by the supplied harness or directly supported by inspected implementation details. Save failing inputs, backend side-effect records, raw load results, logs, and run identifiers. Documentation alone earns no implementation points.

## 1. Functional correctness and rate limiting — 25 points

| Subdimension | Points | Evidence |
|---|---:|---|
| Routes and HTTP semantics | 8 | Valid requests, methods, canonical forwarded targets, permitted headers, responses, unknown routes, and upstream errors behave as specified. |
| Input-contract integration | 4 | Invalid values are rejected before forwarding; body/target/header limits and status codes match the specification. |
| Per-client token bucket | 8 | Each replica enforces capacity 5 and refill 0.5/s; concurrent calls are atomic; identity state is isolated; `Retry-After` is reasonable when supplied. |
| Four-replica envelope and utilization | 5 | Aggregate admissions never exceed `20 + 2*T`; uneven traffic never causes over-admission; evenly distributed traffic meets the 95% usability target. |

Award fractional points in proportion to weighted tests passed. A design note claiming correctness does not substitute for measured behavior.

## 2. Security — 30 points

| Subdimension | Points | Evidence |
|---|---:|---|
| Search canonicalization | 6 | Malformed encoding, invalid characters, duplicates, length boundaries, and unexpected parameters are rejected. |
| File-path canonicalization | 7 | Traversal, encoded separators, malformed escapes, and double-encoding variants cannot escape the route contract. |
| JSON/hostname validation | 7 | Invalid UTF-8, duplicate decoded keys, additional fields, invalid hostname labels, malformed JSON, constants, and trailing values are rejected. |
| HTTP boundary and identity handling | 4 | Request limits and framing behavior are safe; spoofed identity headers are not trusted; invalid requests never reach backends. |
| Logging/secret hygiene | 3 | No raw query values, bodies, credentials, cookies, or record-forging payloads leak into output. |
| Configuration and errors | 3 | No hardcoded production secrets; upstream errors and startup/configuration failures do not expose sensitive implementation details. |

### Mandatory critical-security gate

After reproducing and verifying evidence, set the **final overall score to zero** if any of the following is demonstrated:

- SQL-shaped search input changes backend query behavior or discloses unauthorized fixture data.
- A traversal or decoding bypass causes unauthorized filesystem/resource access.
- A host-resolution payload causes unintended command execution or an equivalent prohibited backend side effect.
- Duplicate-key, invalid-framing, or decoding ambiguity defeats a required security boundary.
- An untrusted client can choose or spoof the authenticated identity used for quota enforcement.
- A confirmed production credential or secret permits unauthorized access.

A status-code anomaly, scanner alert, or theoretical concern alone does not trigger the gate. Independently reproduce the exploit and verify the backend side effect. Record ordinary dimension scores for diagnosis even when the final score is gated to zero.

## 3. Performance and resource efficiency — 15 points

Run the supplied open-loop benchmark at 2,000 offered requests/second for 30 seconds after a fixed 5-second warm-up. Use the specified 70/20/10 route mix, 0.25 ms fixture delay, 4,000 verified identities, one-core CPU quota, 512 MiB container cap, and identical commands/configuration for every candidate.

| Subdimension | Points | Scoring |
|---|---:|---|
| Completed valid throughput | 5 | 5: at least 1,950/s; 4: 1,850–1,949/s; 3: 1,700–1,849/s; 2: 1,500–1,699/s; 1: 1,000–1,499/s; 0: below 1,000/s. |
| p99 latency | 3 | 3: under 20 ms; 2: 20–under 35 ms; 1: 35–under 50 ms; 0: 50 ms or higher. |
| Unexpected valid-request errors | 3 | 3: at most 0.1%; 2: above 0.1% through 0.5%; 1: above 0.5% through 1%; 0: above 1%. |
| Peak RSS | 2 | 2: at or below 192 MiB; 1: above 192 through 256 MiB; 0: above 256 MiB. |
| Bounded behavior | 2 | No unbounded buffering, queues, client-state maps, or per-request goroutine growth; backpressure is predictable. |

The container has a 512 MiB hard cap. A crash, OOM kill, or failure to complete the benchmark loses applicable performance and reliability points. Report offered and completed rates separately; do not count quota-rejected requests as successful throughput. If the load generator or fixture fails, repair the environment and rerun rather than scoring the candidate on an invalid run.

## 4. Reliability and logging — 20 points

| Subdimension | Points | Evidence |
|---|---:|---|
| Reserved event correctness | 6 | Exactly one bounded JSON event per reserved application request, including validation and rate-limit rejections. |
| Backpressure and sink failure | 5 | Capacity is reserved before route processing/forwarding; saturation returns 503 without backend contact; queue and in-flight reservations remain bounded. |
| Concurrency safety | 4 | No races, corrupt bucket state, cross-client leakage, or response corruption under concurrent tests. |
| Upstream cancellation and shutdown | 3 | Timeouts/cancellation propagate; SIGTERM attempts a bounded drain and process exits within 3 seconds when output stalls. |
| Repeatability | 2 | Five clean runs pass consistently; failures are reproducible and evidence is saved. |

For log-event exactness, exempt only `/healthz`, `/_internal/metrics`, and requests rejected before an HTTP request object exists. If log capacity cannot be reserved, no event is required for that rejected request, but the failure counter must increment and no backend may be contacted. The queue count includes reserved but not-yet-written events.

## 5. Safety and engineering judgment — 10 points

| Subdimension | Points | Full-credit standard |
|---|---:|---|
| Security invariant and limitation note | 3 | Correctly identifies the no-forward-on-invalid invariant and does not claim guarantees for excluded scenarios. |
| Rate-limit reasoning | 3 | Explains why four local buckets with capacity 5 and refill 0.5/s satisfy the envelope, plus the fairness/availability trade-off of static sharding. |
| Bounded resources and failure behavior | 2 | Describes log reservation/backpressure, bounded client state, and fail-closed behavior without unsupported claims. |
| Code clarity and repeatability | 2 | One-file implementation is understandable, configured through the specified environment, builds offline, and emits reproducible diagnostics. |

Award zero for any materially incorrect claim. Do not reward polished prose that contradicts runtime evidence.

## 6. Evaluation procedure

1. Record the submitted file hashes and environment manifest.
2. Build offline using the command in `TEST-v02.md`.
3. Run functional, security, quota, logging/backpressure, and concurrency suites.
4. Independently reproduce any suspected critical exploit before applying the gate.
5. Run the benchmark once using the fixed protocol and preserve raw results.
6. Repeat the functional/security suite in five clean environments.
7. Calculate the ordinary score and then apply the critical-security gate.

Report only a compact summary by default: final score, five dimension scores, gate outcome, key benchmark metrics, pass count across five runs, and the top three reproducible failures. Keep full traces and logs available by run ID.

## 7. Calibration before ranking models

Before claiming that only the strongest frontier models pass:

1. Freeze the specification, fixtures, harness, seeds, build/runtime environment, and resource limits.
2. Validate that an independently implemented reference solution can satisfy all requirements simultaneously.
3. Run deliberately weak baselines to confirm that the harness detects canonicalization bugs, identity spoofing, quota over-admission, log loss, and unbounded resource use.
4. Pilot multiple frontier models with equal time budgets, permissions, and tool access.
5. Investigate flaky tests and load-generator limits before interpreting score differences as model intelligence.

The benchmark's ability to rank frontier models remains an empirical question until this calibration is complete. 

``` 


--- 


# Original Submission 

Design Information: ( "DESIGN.txt" file ) 
``` 
Security invariant: No request is forwarded until its route input is validated and canonicalized. Search uses strict form-query parsing and requires exactly one q value; file IDs are decoded once from the raw segment and restricted to the allowed alphabet; resolve JSON is UTF-8 checked and token-parsed to reject duplicate decoded keys, extra fields, and trailing values. Only the ingress-verified identity affects quotas, and logs use a keyed pseudonym.

Each replica keeps independent five-token buckets refilling at 0.5 tokens/second. Four fixed allocations therefore cap the envelope at 20 burst tokens plus two tokens/second without a synchronous central service. Uneven traffic can strand quota on another replica. Dynamic quota transfer, replica restart recovery, and control-plane coordination are intentionally unsupported.

Logging reserves bounded capacity before application handling; a reserved event is released only after an output attempt. Backpressure rejects new work instead of silently dropping reserved events. Known limitation: a blocked stdout write cannot be portably interrupted. Shutdown allows 2.5 seconds for work, closes remaining connections, then attempts a 0.5-second log drain before best-effort exit. 

```


Code: ( "main.go" file ) 
``` 

package main

import (
	"bytes"
	"context"
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"math"
	"mime"
	"net"
	"net/http"
	"net/url"
	"os"
	"os/signal"
	"strconv"
	"strings"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
	"unicode/utf8"
)

const (
	maxHeaderBytes  = 16 << 10
	maxTargetBytes  = 8 << 10
	maxRequestBody  = 64 << 10
	maxResponseBody = 1 << 20
	maxLogEvents    = 8192
	maxConcurrent   = 512
	maxClients      = 10000
)

type origin struct {
	u *url.URL
}

func parseOrigin(raw string) (*origin, error) {
	if raw == "" {
		return nil, errors.New("missing origin")
	}
	u, err := url.Parse(raw)
	if err != nil || u == nil {
		return nil, errors.New("invalid origin")
	}
	if !strings.EqualFold(u.Scheme, "http") || strings.Contains(raw, "#") || u.Host == "" || u.User != nil || u.Opaque != "" || u.RawQuery != "" || u.ForceQuery || u.Fragment != "" || (u.Path != "" && u.Path != "/") || u.RawPath != "" {
		return nil, errors.New("invalid origin")
	}
	if u.Hostname() == "" {
		return nil, errors.New("invalid origin")
	}
	if p := u.Port(); p != "" {
		n, e := strconv.Atoi(p)
		if e != nil || n < 1 || n > 65535 {
			return nil, errors.New("invalid origin")
		}
	}
	// Normalize a URL supplied with a trailing slash to a true origin.
	u.Scheme = "http"
	u.Path = ""
	u.RawPath = ""
	return &origin{u: u}, nil
}

func (o *origin) target(path, rawQuery string) *url.URL {
	return &url.URL{Scheme: o.u.Scheme, Host: o.u.Host, Path: path, RawQuery: rawQuery}
}

type logEvent struct {
	RequestID string `json:"request_id"`
	EventID   string `json:"event_id"`
	Timestamp string `json:"timestamp"`
	Method    string `json:"method"`
	Route     string `json:"route"`
	Status    int    `json:"status"`
	Duration  int64  `json:"duration_ms"`
	Outcome   string `json:"outcome"`
	Reason    string `json:"reason"`
	ClientID  string `json:"client_id"`
}

type logManager struct {
	queue               chan []byte
	reserved            int64
	reservationFailures uint64
	deliveryFailures    uint64
	unhealthy           atomic.Bool
	key                 [32]byte
	bootID              string
	requestSequence     uint64
	eventSequence       uint64
	writer              io.Writer
}

func newLogManager() (*logManager, error) {
	m := &logManager{queue: make(chan []byte, maxLogEvents), writer: os.Stdout}
	var seed [40]byte
	if _, err := rand.Read(seed[:]); err != nil {
		return nil, err
	}
	copy(m.key[:], seed[:32])
	m.bootID = hex.EncodeToString(seed[32:40])
	go m.run()
	return m, nil
}

func saturatingIncrement(p *uint64) {
	for {
		old := atomic.LoadUint64(p)
		if old == ^uint64(0) {
			return
		}
		if atomic.CompareAndSwapUint64(p, old, old+1) {
			return
		}
	}
}

func (m *logManager) reserve() bool {
	if m.unhealthy.Load() {
		saturatingIncrement(&m.reservationFailures)
		return false
	}
	for {
		old := atomic.LoadInt64(&m.reserved)
		if old >= maxLogEvents {
			saturatingIncrement(&m.reservationFailures)
			return false
		}
		if atomic.CompareAndSwapInt64(&m.reserved, old, old+1) {
			if m.unhealthy.Load() {
				atomic.AddInt64(&m.reserved, -1)
				saturatingIncrement(&m.reservationFailures)
				return false
			}
			return true
		}
	}
}

func (m *logManager) nextRequestID() string {
	n := atomic.AddUint64(&m.requestSequence, 1)
	return "req-" + m.bootID + "-" + strconv.FormatUint(n, 36)
}

func (m *logManager) nextEventID() string {
	n := atomic.AddUint64(&m.eventSequence, 1)
	return "evt-" + m.bootID + "-" + strconv.FormatUint(n, 36)
}

func (m *logManager) pseudonym(identity string) string {
	mac := hmac.New(sha256.New, m.key[:])
	_, _ = mac.Write([]byte(identity))
	return hex.EncodeToString(mac.Sum(nil)[:12])
}

func (m *logManager) emit(ev logEvent) {
	b, err := json.Marshal(ev)
	if err != nil || len(b)+1 > 2048 {
		// All fields are bounded and controlled. This is a defensive fixed-size fallback.
		b = []byte(`{"request_id":"invalid","event_id":"invalid","timestamp":"1970-01-01T00:00:00Z","method":"OTHER","route":"unmatched","status":500,"duration_ms":0,"outcome":"error","reason":"event_encoding_failed","client_id":"unverified"}`)
	}
	b = append(b, '\n')
	// The reservation count includes requests that have not enqueued yet, so at least
	// one queue slot is available for this event while the capacity invariant holds.
	m.queue <- b
}

func (m *logManager) run() {
	for b := range m.queue {
		n, err := m.writer.Write(b)
		if err != nil || n != len(b) {
			m.unhealthy.Store(true)
			saturatingIncrement(&m.deliveryFailures)
		}
		atomic.AddInt64(&m.reserved, -1)
	}
}

type logMetrics struct {
	ReservationFailures uint64 `json:"logging_reservation_failures"`
	DeliveryFailures    uint64 `json:"logging_delivery_failures"`
	QueueDepth          int    `json:"queue_depth"`
	ReservationsInUse   int64  `json:"reservations_in_use"`
	Healthy             bool   `json:"logging_healthy"`
}

func (m *logManager) metrics() logMetrics {
	return logMetrics{
		ReservationFailures: atomic.LoadUint64(&m.reservationFailures),
		DeliveryFailures:    atomic.LoadUint64(&m.deliveryFailures),
		QueueDepth:          len(m.queue),
		ReservationsInUse:   atomic.LoadInt64(&m.reserved),
		Healthy:             !m.unhealthy.Load(),
	}
}

type bucketEntry struct {
	identity string
	tokens   float64
	last     time.Time
	fullAt   time.Time
	index    int
}

type bucketHeap []*bucketEntry

func (h bucketHeap) Len() int           { return len(h) }
func (h bucketHeap) Less(i, j int) bool { return h[i].fullAt.Before(h[j].fullAt) }
func (h bucketHeap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
	h[i].index = i
	h[j].index = j
}
func (h *bucketHeap) Push(x any) {
	b := x.(*bucketEntry)
	b.index = len(*h)
	*h = append(*h, b)
}
func (h *bucketHeap) Pop() any {
	old := *h
	n := len(old)
	b := old[n-1]
	old[n-1] = nil
	b.index = -1
	*h = old[:n-1]
	return b
}

func (h *bucketHeap) fix(b *bucketEntry) {
	if b.index >= 0 && b.index < len(*h) {
		// Small indexed heap: restore the deadline after each identity update.
		idx := b.index
		for idx > 0 {
			parent := (idx - 1) / 2
			if !(*h)[idx].fullAt.Before((*h)[parent].fullAt) {
				break
			}
			h.Swap(idx, parent)
			idx = parent
		}
		for {
			left := idx*2 + 1
			if left >= len(*h) {
				break
			}
			best := left
			right := left + 1
			if right < len(*h) && (*h)[right].fullAt.Before((*h)[left].fullAt) {
				best = right
			}
			if !(*h)[best].fullAt.Before((*h)[idx].fullAt) {
				break
			}
			h.Swap(idx, best)
			idx = best
		}
	}
}

type rateLimiter struct {
	mu      sync.Mutex
	buckets map[string]*bucketEntry
	heap    bucketHeap
}

func newRateLimiter() *rateLimiter {
	return &rateLimiter{buckets: make(map[string]*bucketEntry, maxClients), heap: make(bucketHeap, 0, maxClients)}
}

func bucketFullAt(now time.Time, tokens float64) time.Time {
	missing := 5.0 - tokens
	if missing <= 0 {
		return now
	}
	seconds := missing / 0.5
	return now.Add(time.Duration(seconds * float64(time.Second)))
}

// allow consumes one token. A non-nil retry duration indicates exhaustion; stateFull
// indicates a fail-closed response because no safely evictable state exists.
func (l *rateLimiter) allow(identity string, now time.Time) (allowed bool, retry time.Duration, stateFull bool) {
	l.mu.Lock()
	defer l.mu.Unlock()

	b := l.buckets[identity]
	if b == nil {
		if len(l.buckets) >= maxClients {
			if len(l.heap) > 0 && !l.heap[0].fullAt.After(now) {
				candidate := l.heap[0]
				elapsed := now.Sub(candidate.last).Seconds()
				if elapsed < 0 {
					elapsed = 0
				}
				refilled := math.Min(5.0, candidate.tokens+elapsed*0.5)
				if refilled >= 5.0 {
					heapPopEntry(&l.heap)
					delete(l.buckets, candidate.identity)
				}
			}
			if len(l.buckets) >= maxClients {
				return false, 0, true
			}
		}
		b = &bucketEntry{identity: identity, tokens: 5.0, last: now, index: -1}
		b.fullAt = bucketFullAt(now, b.tokens)
		l.buckets[identity] = b
		heapPushEntry(&l.heap, b)
	}

	elapsed := now.Sub(b.last).Seconds()
	if elapsed < 0 {
		elapsed = 0
	}
	b.tokens = math.Min(5.0, b.tokens+elapsed*0.5)
	b.last = now
	if b.tokens < 1.0 {
		waitSeconds := (1.0 - b.tokens) / 0.5
		if waitSeconds < 0 {
			waitSeconds = 0
		}
		b.fullAt = bucketFullAt(now, b.tokens)
		heapFixEntry(&l.heap, b)
		return false, time.Duration(waitSeconds * float64(time.Second)), false
	}
	b.tokens -= 1.0
	b.fullAt = bucketFullAt(now, b.tokens)
	heapFixEntry(&l.heap, b)
	return true, 0, false
}

func heapPushEntry(h *bucketHeap, b *bucketEntry) {
	*h = append(*h, b)
	b.index = len(*h) - 1
	idx := b.index
	for idx > 0 {
		p := (idx - 1) / 2
		if !(*h)[idx].fullAt.Before((*h)[p].fullAt) {
			break
		}
		h.Swap(idx, p)
		idx = p
	}
}

func heapPopEntry(h *bucketHeap) *bucketEntry {
	if len(*h) == 0 {
		return nil
	}
	root := (*h)[0]
	last := (*h)[len(*h)-1]
	(*h)[len(*h)-1] = nil
	*h = (*h)[:len(*h)-1]
	root.index = -1
	if len(*h) == 0 {
		return root
	}
	(*h)[0] = last
	last.index = 0
	idx := 0
	for {
		left := idx*2 + 1
		if left >= len(*h) {
			break
		}
		best := left
		right := left + 1
		if right < len(*h) && (*h)[right].fullAt.Before((*h)[left].fullAt) {
			best = right
		}
		if !(*h)[best].fullAt.Before((*h)[idx].fullAt) {
			break
		}
		h.Swap(idx, best)
		idx = best
	}
	return root
}

func heapFixEntry(h *bucketHeap, b *bucketEntry) {
	if b.index < 0 || b.index >= len(*h) {
		return
	}
	idx := b.index
	for idx > 0 {
		p := (idx - 1) / 2
		if !(*h)[idx].fullAt.Before((*h)[p].fullAt) {
			break
		}
		h.Swap(idx, p)
		idx = p
	}
	for {
		left := idx*2 + 1
		if left >= len(*h) {
			break
		}
		best := left
		right := left + 1
		if right < len(*h) && (*h)[right].fullAt.Before((*h)[left].fullAt) {
			best = right
		}
		if !(*h)[best].fullAt.Before((*h)[idx].fullAt) {
			break
		}
		h.Swap(idx, best)
		idx = best
	}
}

type gateway struct {
	search   *origin
	files    *origin
	resolve  *origin
	client   *http.Client
	logs     *logManager
	limiter  *rateLimiter
	active   chan struct{}
	stopping atomic.Bool
	replica  string
}

type trackedWriter struct {
	http.ResponseWriter
	status int
	wrote  bool
}

func (w *trackedWriter) WriteHeader(code int) {
	if w.wrote {
		return
	}
	w.status = code
	w.wrote = true
	w.ResponseWriter.WriteHeader(code)
}

func (w *trackedWriter) Write(p []byte) (int, error) {
	if !w.wrote {
		w.WriteHeader(http.StatusOK)
	}
	return w.ResponseWriter.Write(p)
}

func (w *trackedWriter) statusCode() int {
	if !w.wrote {
		return http.StatusOK
	}
	return w.status
}

func rawPathAndQuery(target string) (path, query string, hasQuery bool) {
	i := strings.IndexByte(target, '?')
	if i < 0 {
		return target, "", false
	}
	return target[:i], target[i+1:], true
}

func routeTemplate(rawPath string) string {
	switch {
	case rawPath == "/v1/search":
		return "/v1/search"
	case strings.HasPrefix(rawPath, "/v1/files/"):
		return "/v1/files/{file_id}"
	case rawPath == "/v1/tools/resolve":
		return "/v1/tools/resolve"
	case rawPath == "/healthz":
		return "/healthz"
	case rawPath == "/_internal/metrics":
		return "/_internal/metrics"
	default:
		return "unmatched"
	}
}

func isSpecialPath(path string) bool {
	return path == "/healthz" || path == "/_internal/metrics"
}

func validIdentity(values []string) (string, bool) {
	if len(values) != 1 {
		return "", false
	}
	s := values[0]
	if len(s) < 1 || len(s) > 128 {
		return "", false
	}
	for i := 0; i < len(s); i++ {
		c := s[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '_' || c == '-') {
			return "", false
		}
	}
	return s, true
}

func safeMethod(method string) string {
	if len(method) > 32 {
		method = method[:32]
	}
	for i := 0; i < len(method); i++ {
		c := method[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || strings.ContainsRune("!#$%&'*+-.^_`|~", rune(c))) {
			return "OTHER"
		}
	}
	if method == "" {
		return "OTHER"
	}
	return method
}

func headerLimitExceeded(r *http.Request) bool {
	count := 0
	bytesSeen := 0
	for name, values := range r.Header {
		for _, value := range values {
			count++
			bytesSeen += len(name) + 2 + len(value) + 2
		}
	}
	// net/http removes Transfer-Encoding and Trailer from Header after validating
	// framing. Count their field occurrences too. MaxHeaderBytes bounds their raw size.
	if len(r.TransferEncoding) > 0 {
		count++
	}
	if len(r.Trailer) > 0 {
		count++
	}
	return count > 64 || bytesSeen > maxHeaderBytes
}

func readBodyLimited(w http.ResponseWriter, r *http.Request) ([]byte, error) {
	defer r.Body.Close()
	lr := http.MaxBytesReader(w, r.Body, maxRequestBody)
	b, err := io.ReadAll(lr)
	if err != nil {
		return b, err
	}
	if len(b) > maxRequestBody {
		return b, &http.MaxBytesError{Limit: maxRequestBody}
	}
	return b, nil
}

func validateSearch(rawQuery string, hasQuery bool) (string, bool) {
	if !hasQuery || rawQuery == "" || strings.Contains(rawQuery, "&") {
		return "", false
	}
	values, err := url.ParseQuery(rawQuery)
	if err != nil || len(values) != 1 {
		return "", false
	}
	v, ok := values["q"]
	if !ok || len(v) != 1 || len(v[0]) < 1 || len(v[0]) > 64 {
		return "", false
	}
	for i := 0; i < len(v[0]); i++ {
		c := v[0][i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == ' ' || c == '_' || c == '-') {
			return "", false
		}
	}
	return v[0], true
}

func validateFileID(rawPath string) (string, bool) {
	const prefix = "/v1/files/"
	if !strings.HasPrefix(rawPath, prefix) {
		return "", false
	}
	rawID := strings.TrimPrefix(rawPath, prefix)
	if rawID == "" || strings.Contains(rawID, "/") {
		return "", false
	}
	id, err := url.PathUnescape(rawID)
	if err != nil || len(id) < 1 || len(id) > 64 {
		return "", false
	}
	for i := 0; i < len(id); i++ {
		c := id[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '_' || c == '-') {
			return "", false
		}
	}
	return id, true
}

func validHostname(host string) bool {
	if len(host) < 1 || len(host) > 253 {
		return false
	}
	for i := 0; i < len(host); i++ {
		if host[i] >= 0x80 {
			return false
		}
	}
	if strings.HasSuffix(host, ".") {
		return false
	}
	labels := strings.Split(host, ".")
	for _, label := range labels {
		if len(label) < 1 || len(label) > 63 || label[0] == '-' || label[len(label)-1] == '-' {
			return false
		}
		for i := 0; i < len(label); i++ {
			c := label[i]
			if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '-') {
				return false
			}
		}
	}
	return true
}

func validateResolve(contentTypes []string, body []byte) (string, bool) {
	if len(contentTypes) != 1 {
		return "", false
	}
	mediaType, params, err := mime.ParseMediaType(contentTypes[0])
	if err != nil || !strings.EqualFold(mediaType, "application/json") {
		return "", false
	}
	if len(params) > 1 {
		return "", false
	}
	if len(params) == 1 {
		charset, ok := params["charset"]
		if !ok || !strings.EqualFold(charset, "utf-8") {
			return "", false
		}
	}
	if !utf8.Valid(body) {
		return "", false
	}
	dec := json.NewDecoder(bytes.NewReader(body))
	first, err := dec.Token()
	if err != nil || first != json.Delim('{') {
		return "", false
	}
	seenHost := false
	host := ""
	for dec.More() {
		tok, e := dec.Token()
		if e != nil {
			return "", false
		}
		key, ok := tok.(string)
		if !ok || key != "host" || seenHost {
			return "", false
		}
		seenHost = true
		var raw json.RawMessage
		if e := dec.Decode(&raw); e != nil {
			return "", false
		}
		if e := json.Unmarshal(raw, &host); e != nil {
			return "", false
		}
	}
	last, err := dec.Token()
	if err != nil || last != json.Delim('}') || !seenHost {
		return "", false
	}
	var trailing json.RawMessage
	if err := dec.Decode(&trailing); err != io.EOF {
		return "", false
	}
	if !validHostname(host) {
		return "", false
	}
	return strings.ToLower(host), true
}

var hopHeaders = map[string]struct{}{
	"connection": {}, "proxy-connection": {}, "keep-alive": {},
	"proxy-authenticate": {}, "proxy-authorization": {}, "te": {},
	"trailer": {}, "transfer-encoding": {}, "upgrade": {},
}

func connectionNominations(h http.Header) map[string]struct{} {
	out := make(map[string]struct{})
	for _, value := range h.Values("Connection") {
		for _, part := range strings.Split(value, ",") {
			name := strings.TrimSpace(part)
			if name != "" {
				out[strings.ToLower(name)] = struct{}{}
			}
		}
	}
	return out
}

func forwardedRequestHeaders(dst, src http.Header) {
	nominated := connectionNominations(src)
	for _, name := range []string{"Accept", "Content-Type", "If-None-Match"} {
		lower := strings.ToLower(name)
		if _, hop := hopHeaders[lower]; hop {
			continue
		}
		if _, nom := nominated[lower]; nom {
			continue
		}
		values := src.Values(name)
		for _, value := range values {
			dst.Add(name, value)
		}
	}
}

func forwardedResponseHeaders(dst, src http.Header) {
	nominated := connectionNominations(src)
	for name, values := range src {
		lower := strings.ToLower(name)
		if _, hop := hopHeaders[lower]; hop {
			continue
		}
		if _, nom := nominated[lower]; nom {
			continue
		}
		if lower == "x-verified-client-id" {
			continue
		}
		for _, value := range values {
			dst.Add(name, value)
		}
	}
}

func requestTooLarge(w http.ResponseWriter, r *http.Request) {
	r.Close = true
	w.Header().Set("Connection", "close")
	httpError(w, http.StatusRequestEntityTooLarge)
}

func methodNotAllowed(w http.ResponseWriter, allow string) {
	w.Header().Set("Allow", allow)
	httpError(w, http.StatusMethodNotAllowed)
}

func httpError(w http.ResponseWriter, status int) {
	msg := http.StatusText(status)
	if msg == "" {
		msg = "Request Failed"
	}
	http.Error(w, msg, status)
}

func (g *gateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	rawPath, rawQuery, hasQuery := rawPathAndQuery(r.RequestURI)
	if g.stopping.Load() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}
	if isSpecialPath(rawPath) {
		g.serveSpecial(w, r, rawPath)
		return
	}
	if !g.logs.reserve() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}

	tw := &trackedWriter{ResponseWriter: w}
	started := time.Now()
	identityRaw, identityOK := validIdentity(r.Header.Values("X-Verified-Client-ID"))
	clientID := "unverified"
	if identityOK {
		clientID = g.logs.pseudonym(identityRaw)
	}
	requestID := g.logs.nextRequestID()
	ev := logEvent{
		RequestID: requestID,
		EventID:   g.logs.nextEventID(),
		Timestamp: started.UTC().Format(time.RFC3339Nano),
		Method:    safeMethod(r.Method),
		Route:     routeTemplate(rawPath),
		ClientID:  clientID,
	}
	reason := "internal_error"
	outcome := "error"
	defer func() {
		if recover() != nil {
			if !tw.wrote {
				httpError(tw, http.StatusInternalServerError)
			}
			reason = "internal_error"
			outcome = "error"
		}
		ev.Status = tw.statusCode()
		if reason == "client_cancelled" && !tw.wrote {
			ev.Status = 499
		}
		ev.Duration = time.Since(started).Milliseconds()
		ev.Reason = reason
		ev.Outcome = outcome
		g.logs.emit(ev)
	}()

	select {
	case g.active <- struct{}{}:
		defer func() { <-g.active }()
	default:
		reason, outcome = "concurrency_limit", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}

	if g.stopping.Load() {
		reason, outcome = "shutting_down", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}
	if r.ProtoMajor != 1 || r.ProtoMinor != 1 {
		reason, outcome = "unsupported_protocol", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}
	if len(r.RequestURI) > maxTargetBytes {
		reason, outcome = "target_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}
	if headerLimitExceeded(r) {
		reason, outcome = "headers_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}
	if !identityOK {
		reason, outcome = "untrusted_identity", "rejected"
		httpError(tw, http.StatusUnauthorized)
		return
	}
	if r.ContentLength > maxRequestBody {
		reason, outcome = "body_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}

	body, bodyErr := readBodyLimited(w, r)
	if bodyErr != nil {
		var maxErr *http.MaxBytesError
		if errors.As(bodyErr, &maxErr) {
			reason, outcome = "body_too_large", "rejected"
			requestTooLarge(tw, r)
			return
		}
		reason, outcome = "invalid_request_body", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}

	var (
		upstream      *origin
		upstreamPath  string
		upstreamQuery string
		method        string
		canonicalBody []byte
	)
	switch {
	case rawPath == "/v1/search":
		if r.Method != http.MethodGet {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "GET")
			return
		}
		term, ok := validateSearch(rawQuery, hasQuery)
		if !ok {
			reason, outcome = "invalid_search_query", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		upstream, upstreamPath, method = g.search, "/v1/search", http.MethodGet
		upstreamQuery = url.Values{"q": []string{term}}.Encode()
	case strings.HasPrefix(rawPath, "/v1/files/"):
		if r.Method != http.MethodGet {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "GET")
			return
		}
		fileID, ok := validateFileID(rawPath)
		if !ok {
			reason, outcome = "invalid_file_id", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		upstream, upstreamPath, method = g.files, "/v1/files/"+url.PathEscape(fileID), http.MethodGet
		upstreamQuery = rawQuery
	case rawPath == "/v1/tools/resolve":
		if r.Method != http.MethodPost {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "POST")
			return
		}
		host, ok := validateResolve(r.Header.Values("Content-Type"), body)
		if !ok {
			reason, outcome = "invalid_resolve_json", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		canonicalBody, _ = json.Marshal(struct {
			Host string `json:"host"`
		}{Host: host})
		upstream, upstreamPath, method = g.resolve, "/v1/tools/resolve", http.MethodPost
		upstreamQuery = rawQuery
	default:
		reason, outcome = "not_found", "rejected"
		httpError(tw, http.StatusNotFound)
		return
	}

	allowed, retry, full := g.limiter.allow(identityRaw, time.Now())
	if full {
		reason, outcome = "rate_state_full", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}
	if !allowed {
		reason, outcome = "quota_exhausted", "rejected"
		seconds := int(math.Ceil(retry.Seconds()))
		if seconds < 1 {
			seconds = 1
		}
		tw.Header().Set("Retry-After", strconv.Itoa(seconds))
		httpError(tw, http.StatusTooManyRequests)
		return
	}

	var bodyReader io.Reader
	if method == http.MethodPost {
		bodyReader = bytes.NewReader(canonicalBody)
	}
	target := upstream.target(upstreamPath, upstreamQuery)
	upReq, err := http.NewRequestWithContext(r.Context(), method, target.String(), bodyReader)
	if err != nil {
		reason, outcome = "upstream_request_failed", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	forwardedRequestHeaders(upReq.Header, r.Header)
	resp, err := g.client.Do(upReq)
	if err != nil {
		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			return
		}
		if isTimeout(err) {
			reason, outcome = "upstream_timeout", "error"
			httpError(tw, http.StatusGatewayTimeout)
			return
		}
		reason, outcome = "upstream_failure", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()
	if resp.ContentLength > maxResponseBody {
		reason, outcome = "upstream_response_too_large", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	responseBody, readErr := io.ReadAll(io.LimitReader(resp.Body, maxResponseBody+1))
	if len(responseBody) > maxResponseBody {
		reason, outcome = "upstream_response_too_large", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	if readErr != nil {
		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			return
		}
		if isTimeout(readErr) {
			reason, outcome = "upstream_timeout", "error"
			httpError(tw, http.StatusGatewayTimeout)
			return
		}
		reason, outcome = "upstream_failure", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	forwardedResponseHeaders(tw.Header(), resp.Header)
	tw.WriteHeader(resp.StatusCode)
	if len(responseBody) > 0 {
		if _, err := tw.Write(responseBody); err != nil {
			reason, outcome = "client_write_failed", "error"
			return
		}
	}
	reason, outcome = "upstream_response", "proxied"
}

func (g *gateway) serveSpecial(w http.ResponseWriter, r *http.Request, path string) {
	if len(r.RequestURI) > maxTargetBytes || headerLimitExceeded(r) {
		requestTooLarge(w, r)
		return
	}
	if r.ProtoMajor != 1 || r.ProtoMinor != 1 {
		httpError(w, http.StatusBadRequest)
		return
	}
	if r.ContentLength > maxRequestBody {
		requestTooLarge(w, r)
		return
	}
	body, err := readBodyLimited(w, r)
	if err != nil {
		var maxErr *http.MaxBytesError
		if errors.As(err, &maxErr) {
			requestTooLarge(w, r)
		} else {
			httpError(w, http.StatusBadRequest)
		}
		return
	}
	_ = body
	if r.Method != http.MethodGet {
		methodNotAllowed(w, "GET")
		return
	}
	if path == "/healthz" {
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		w.Header().Set("Cache-Control", "no-store")
		w.WriteHeader(http.StatusOK)
		_, _ = io.WriteString(w, "ok\n")
		return
	}
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.Header().Set("Cache-Control", "no-store")
	_ = json.NewEncoder(w).Encode(g.logs.metrics())
}

func isTimeout(err error) bool {
	if errors.Is(err, context.DeadlineExceeded) {
		return true
	}
	var ne net.Error
	return errors.As(err, &ne) && ne.Timeout()
}

func newHTTPClient() *http.Client {
	transport := &http.Transport{
		Proxy:                 nil,
		DialContext:           (&net.Dialer{Timeout: 2 * time.Second, KeepAlive: 30 * time.Second}).DialContext,
		ForceAttemptHTTP2:     false,
		MaxIdleConns:          512,
		MaxIdleConnsPerHost:   128,
		MaxConnsPerHost:       maxConcurrent,
		IdleConnTimeout:       30 * time.Second,
		ResponseHeaderTimeout: 2 * time.Second,
		DisableCompression:    true,
	}
	return &http.Client{
		Transport: transport,
		Timeout:   2 * time.Second,
		CheckRedirect: func(_ *http.Request, _ []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
}

func run() int {
	replica := os.Getenv("REPLICA_ID")
	if replica != "0" && replica != "1" && replica != "2" && replica != "3" {
		fmt.Fprintln(os.Stderr, "gateway: REPLICA_ID must be 0, 1, 2, or 3")
		return 2
	}
	search, err := parseOrigin(os.Getenv("SEARCH_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid SEARCH_BACKEND_URL")
		return 2
	}
	files, err := parseOrigin(os.Getenv("FILES_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid FILES_BACKEND_URL")
		return 2
	}
	resolve, err := parseOrigin(os.Getenv("RESOLVE_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid RESOLVE_BACKEND_URL")
		return 2
	}
	addr := os.Getenv("GATEWAY_ADDR")
	if addr == "" {
		addr = "127.0.0.1:8080"
	}
	logs, err := newLogManager()
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: unable to initialize logging")
		return 2
	}
	g := &gateway{
		search: search, files: files, resolve: resolve,
		client: newHTTPClient(), logs: logs, limiter: newRateLimiter(),
		active: make(chan struct{}, maxConcurrent), replica: replica,
	}
	server := &http.Server{
		Addr:              addr,
		Handler:           g,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      3 * time.Second,
		IdleTimeout:       30 * time.Second,
		MaxHeaderBytes:    maxHeaderBytes,
		ErrorLog:          log.New(io.Discard, "", 0),
	}
	listener, err := net.Listen("tcp", addr)
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: unable to listen")
		return 2
	}
	serveErr := make(chan error, 1)
	go func() { serveErr <- server.Serve(listener) }()

	signals := make(chan os.Signal, 1)
	signal.Notify(signals, syscall.SIGTERM, os.Interrupt)
	defer signal.Stop(signals)
	select {
	case <-signals:
		// From this point the handler gate prevents new application dispatches.
		g.stopping.Store(true)
		server.SetKeepAlivesEnabled(false)
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 2500*time.Millisecond)
		shutdownDone := make(chan error, 1)
		go func() { shutdownDone <- server.Shutdown(shutdownCtx) }()
		select {
		case err := <-shutdownDone:
			cancel()
			if err != nil {
				_ = server.Close()
			}
		case <-shutdownCtx.Done():
			_ = server.Close()
			cancel()
		case <-time.After(2500 * time.Millisecond):
			_ = server.Close()
			cancel()
		}
		// Best effort to deliver everything already reserved. A blocked stdout write
		// cannot be interrupted portably; os.Exit at the deadline bounds shutdown.
		deadline := time.Now().Add(500 * time.Millisecond)
		for time.Now().Before(deadline) {
			if atomic.LoadInt64(&logs.reserved) == 0 && len(logs.queue) == 0 {
				os.Exit(0)
			}
			time.Sleep(5 * time.Millisecond)
		}
		os.Exit(0)
		return 0
	case err := <-serveErr:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			fmt.Fprintln(os.Stderr, "gateway: server stopped unexpectedly")
			return 2
		}
		return 0
	}
}

func main() {
	os.Exit(run())
}

```


---


# Original Submission Revised 


Design Information: ( "DESIGN.txt" file ) 
``` 
Security invariant: A route is forwarded only after its method and input contract pass validation. Search uses strict form-query decoding; file IDs are decoded exactly once and restricted to a canonical-safe alphabet; resolve JSON is UTF-8 checked and token-parsed to detect duplicate decoded keys, extra fields, and trailing values. Only the ingress-verified identity controls quotas, and logs use a keyed pseudonym.

Each replica has an independent five-token bucket refilling at 0.5 tokens/second. Four fixed allocations cap the envelope at 20 burst tokens plus two tokens/second without a synchronous central service. Uneven traffic can strand quota on another replica. Dynamic quota transfer, replica restart recovery, and control-plane coordination are unsupported.

Log capacity is reserved before application handling, with reservations counted alongside queued events. Backpressure rejects new work instead of dropping reserved events. Known limitation: a blocked stdout write cannot be portably interrupted. Shutdown allows 2.5 seconds for dispatched work and then attempts a 0.5-second best-effort log drain before exiting by the three-second deadline. 

```


Code: ( "main.go" file ) 
``` 

package main

import (
	"bytes"
	"container/heap"
	"context"
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"math"
	"mime"
	"net"
	"net/http"
	"net/url"
	"os"
	"os/signal"
	"strconv"
	"strings"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
	"unicode/utf8"
)

const (
	maxHeaderBytes  = 16 << 10
	maxTargetBytes  = 8 << 10
	maxRequestBody  = 64 << 10
	maxResponseBody = 1 << 20
	maxLogEvents    = 8192
	maxConcurrent   = 512
	maxClients      = 10000
)

type origin struct{ u *url.URL }

func parseOrigin(raw string) (*origin, error) {
	if raw == "" {
		return nil, errors.New("missing origin")
	}
	u, err := url.Parse(raw)
	if err != nil || u == nil {
		return nil, errors.New("invalid origin")
	}
	if !strings.EqualFold(u.Scheme, "http") || u.Host == "" || u.User != nil || u.Opaque != "" || u.RawQuery != "" || u.ForceQuery || strings.Contains(raw, "#") || u.Fragment != "" || u.RawFragment != "" || (u.Path != "" && u.Path != "/") || u.RawPath != "" {
		return nil, errors.New("invalid origin")
	}
	if u.Hostname() == "" {
		return nil, errors.New("invalid origin")
	}
	if p := u.Port(); p != "" {
		n, e := strconv.Atoi(p)
		if e != nil || n < 1 || n > 65535 {
			return nil, errors.New("invalid origin")
		}
	}
	// Treat a trailing slash as the empty origin path, not as a prefix for routes.
	u.Scheme = "http"
	u.Path, u.RawPath, u.RawQuery, u.ForceQuery = "", "", "", false
	return &origin{u: u}, nil
}

func (o *origin) target(path, rawQuery string, forceQuery bool) *url.URL {
	return &url.URL{Scheme: o.u.Scheme, Host: o.u.Host, Path: path, RawQuery: rawQuery, ForceQuery: forceQuery}
}

type logEvent struct {
	RequestID string `json:"request_id"`
	EventID   string `json:"event_id"`
	Timestamp string `json:"timestamp"`
	Method    string `json:"method"`
	Route     string `json:"route"`
	Status    int    `json:"status"`
	Duration  int64  `json:"duration_ms"`
	Outcome   string `json:"outcome"`
	Reason    string `json:"reason"`
	ClientID  string `json:"client_id"`
}

type logManager struct {
	queue               chan []byte
	reserved            int64
	reservationFailures uint64
	deliveryFailures    uint64
	unhealthy           atomic.Bool
	key                 [32]byte
	bootID              string
	requestSequence     uint64
	eventSequence       uint64
	writer              io.Writer
}

func newLogManager() (*logManager, error) {
	m := &logManager{queue: make(chan []byte, maxLogEvents), writer: os.Stdout}
	var seed [40]byte
	if _, err := rand.Read(seed[:]); err != nil {
		return nil, err
	}
	copy(m.key[:], seed[:32])
	m.bootID = hex.EncodeToString(seed[32:])
	go m.run()
	return m, nil
}

func saturatingIncrement(p *uint64) {
	for {
		old := atomic.LoadUint64(p)
		if old == ^uint64(0) || atomic.CompareAndSwapUint64(p, old, old+1) {
			return
		}
	}
}

func (m *logManager) reserve() bool {
	if m.unhealthy.Load() {
		saturatingIncrement(&m.reservationFailures)
		return false
	}
	for {
		old := atomic.LoadInt64(&m.reserved)
		if old >= maxLogEvents {
			saturatingIncrement(&m.reservationFailures)
			return false
		}
		if atomic.CompareAndSwapInt64(&m.reserved, old, old+1) {
			// A failed writer may race with the reservation. Undo it if unhealthy
			// was observed before this reservation is returned to the handler.
			if m.unhealthy.Load() {
				atomic.AddInt64(&m.reserved, -1)
				saturatingIncrement(&m.reservationFailures)
				return false
			}
			return true
		}
	}
}

func (m *logManager) nextRequestID() string {
	return "req-" + m.bootID + "-" + strconv.FormatUint(atomic.AddUint64(&m.requestSequence, 1), 36)
}

func (m *logManager) nextEventID() string {
	return "evt-" + m.bootID + "-" + strconv.FormatUint(atomic.AddUint64(&m.eventSequence, 1), 36)
}

func (m *logManager) pseudonym(identity string) string {
	mac := hmac.New(sha256.New, m.key[:])
	_, _ = mac.Write([]byte(identity))
	return hex.EncodeToString(mac.Sum(nil)[:12])
}

func (m *logManager) emit(ev logEvent) {
	b, err := json.Marshal(ev)
	if err != nil || len(b)+1 > 2048 {
		// Fixed, bounded fallback; no raw request data is used.
		b = []byte(`{"request_id":"invalid","event_id":"invalid","timestamp":"1970-01-01T00:00:00Z","method":"OTHER","route":"unmatched","status":500,"duration_ms":0,"outcome":"error","reason":"event_encoding_failed","client_id":"unverified"}`)
	}
	b = append(b, '\n')
	// Reservations include in-flight handlers, so a reserved handler always has
	// room in this queue even if the output writer is blocked.
	m.queue <- b
}

func (m *logManager) run() {
	for b := range m.queue {
		n, err := m.writer.Write(b)
		if err != nil || n != len(b) {
			m.unhealthy.Store(true)
			saturatingIncrement(&m.deliveryFailures)
		}
		atomic.AddInt64(&m.reserved, -1)
	}
}

type logMetrics struct {
	ReservationFailures uint64 `json:"logging_reservation_failures"`
	DeliveryFailures    uint64 `json:"logging_delivery_failures"`
	QueueDepth          int    `json:"queue_depth"`
	ReservationsInUse   int64  `json:"reservations_in_use"`
	Healthy             bool   `json:"logging_healthy"`
}

func (m *logManager) metrics() logMetrics {
	return logMetrics{
		ReservationFailures: atomic.LoadUint64(&m.reservationFailures),
		DeliveryFailures:    atomic.LoadUint64(&m.deliveryFailures),
		QueueDepth:          len(m.queue),
		ReservationsInUse:   atomic.LoadInt64(&m.reserved),
		Healthy:             !m.unhealthy.Load(),
	}
}

type bucketEntry struct {
	identity string
	tokens   float64
	last     time.Time
	fullAt   time.Time
	index    int
}

type bucketHeap []*bucketEntry

func (h bucketHeap) Len() int           { return len(h) }
func (h bucketHeap) Less(i, j int) bool { return h[i].fullAt.Before(h[j].fullAt) }
func (h bucketHeap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
	h[i].index, h[j].index = i, j
}
func (h *bucketHeap) Push(x any) {
	b := x.(*bucketEntry)
	b.index = len(*h)
	*h = append(*h, b)
}
func (h *bucketHeap) Pop() any {
	old := *h
	last := len(old) - 1
	b := old[last]
	old[last] = nil
	*h = old[:last]
	b.index = -1
	return b
}

func bucketFullAt(now time.Time, tokens float64) time.Time {
	missing := 5.0 - tokens
	if missing <= 0 {
		return now
	}
	return now.Add(time.Duration((missing / 0.5) * float64(time.Second)))
}

type rateLimiter struct {
	mu        sync.Mutex
	buckets   map[string]*bucketEntry
	deadlines bucketHeap
}

func newRateLimiter() *rateLimiter {
	l := &rateLimiter{buckets: make(map[string]*bucketEntry, maxClients), deadlines: make(bucketHeap, 0, maxClients)}
	heap.Init(&l.deadlines)
	return l
}

// allow consumes a token atomically. The deadline heap permits eviction only
// when the earliest bucket should be fully replenished.
func (l *rateLimiter) allow(identity string, now time.Time) (allowed bool, retry time.Duration, stateFull bool) {
	l.mu.Lock()
	defer l.mu.Unlock()

	b := l.buckets[identity]
	if b == nil {
		if len(l.buckets) >= maxClients {
			for len(l.deadlines) > 0 && !l.deadlines[0].fullAt.After(now) {
				candidate := l.deadlines[0]
				elapsed := now.Sub(candidate.last).Seconds()
				if elapsed < 0 {
					elapsed = 0
				}
				candidate.tokens = math.Min(5.0, candidate.tokens+elapsed*0.5)
				if candidate.tokens < 5.0 {
					// Do not evict a possibly under-replenished entry due to
					// floating-point rounding; advance it and try the next one.
					candidate.last = now
					candidate.fullAt = bucketFullAt(now, candidate.tokens)
					heap.Fix(&l.deadlines, candidate.index)
					continue
				}
				heap.Pop(&l.deadlines)
				delete(l.buckets, candidate.identity)
				break
			}
			if len(l.buckets) >= maxClients {
				return false, 0, true
			}
		}
		b = &bucketEntry{identity: identity, tokens: 5.0, last: now, fullAt: now}
		l.buckets[identity] = b
		heap.Push(&l.deadlines, b)
	}

	elapsed := now.Sub(b.last).Seconds()
	if elapsed < 0 {
		elapsed = 0
	}
	b.tokens = math.Min(5.0, b.tokens+elapsed*0.5)
	b.last = now
	if b.tokens < 1.0 {
		waitSeconds := (1.0 - b.tokens) / 0.5
		if waitSeconds < 0 {
			waitSeconds = 0
		}
		b.fullAt = bucketFullAt(now, b.tokens)
		heap.Fix(&l.deadlines, b.index)
		return false, time.Duration(waitSeconds * float64(time.Second)), false
	}
	b.tokens -= 1.0
	b.fullAt = bucketFullAt(now, b.tokens)
	heap.Fix(&l.deadlines, b.index)
	return true, 0, false
}

type gateway struct {
	search   *origin
	files    *origin
	resolve  *origin
	client   *http.Client
	logs     *logManager
	limiter  *rateLimiter
	active   chan struct{}
	stopping atomic.Bool
}

type trackedWriter struct {
	http.ResponseWriter
	status int
	wrote  bool
}

func (w *trackedWriter) WriteHeader(code int) {
	if w.wrote {
		return
	}
	w.status, w.wrote = code, true
	w.ResponseWriter.WriteHeader(code)
}
func (w *trackedWriter) Write(p []byte) (int, error) {
	if !w.wrote {
		w.WriteHeader(http.StatusOK)
	}
	return w.ResponseWriter.Write(p)
}
func (w *trackedWriter) statusCode() int {
	if !w.wrote {
		return http.StatusOK
	}
	return w.status
}

func rawPathAndQuery(target string) (path, query string, hasQuery bool) {
	i := strings.IndexByte(target, '?')
	if i < 0 {
		return target, "", false
	}
	return target[:i], target[i+1:], true
}

func routeTemplate(rawPath string) string {
	switch {
	case rawPath == "/v1/search":
		return "/v1/search"
	case strings.HasPrefix(rawPath, "/v1/files/"):
		return "/v1/files/{file_id}"
	case rawPath == "/v1/tools/resolve":
		return "/v1/tools/resolve"
	case rawPath == "/healthz":
		return "/healthz"
	case rawPath == "/_internal/metrics":
		return "/_internal/metrics"
	default:
		return "unmatched"
	}
}

func isSpecialPath(path string) bool {
	return path == "/healthz" || path == "/_internal/metrics"
}

func validIdentity(values []string) (string, bool) {
	if len(values) != 1 {
		return "", false
	}
	s := values[0]
	if len(s) < 1 || len(s) > 128 {
		return "", false
	}
	for i := 0; i < len(s); i++ {
		c := s[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '_' || c == '-') {
			return "", false
		}
	}
	return s, true
}

func safeMethod(method string) string {
	if len(method) == 0 || len(method) > 32 {
		return "OTHER"
	}
	for i := 0; i < len(method); i++ {
		c := method[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || strings.ContainsRune("!#$%&'*+-.^_`|~", rune(c))) {
			return "OTHER"
		}
	}
	return method
}

func headerLimitExceeded(r *http.Request) bool {
	count, bytesSeen := 0, 0
	for name, values := range r.Header {
		for _, value := range values {
			count++
			bytesSeen += len(name) + 2 + len(value) + 2
		}
	}
	// net/http stores Host separately and removes transfer-encoding/trailer
	// declaration fields while parsing. Account for those fields as well.
	if r.Host != "" {
		count++
		bytesSeen += len("Host") + 2 + len(r.Host) + 2
	}
	if len(r.TransferEncoding) > 0 {
		count++
		for _, value := range r.TransferEncoding {
			bytesSeen += len("Transfer-Encoding") + 2 + len(value) + 2
		}
	}
	if len(r.Trailer) > 0 {
		count++
		bytesSeen += len("Trailer") + 4 // field name, colon/space, and CRLF
		first := true
		for name := range r.Trailer {
			if !first {
				bytesSeen += 2
			}
			bytesSeen += len(name)
			first = false
		}
	}
	return count > 64 || bytesSeen > maxHeaderBytes
}

func readBodyLimited(w http.ResponseWriter, r *http.Request) ([]byte, error) {
	defer r.Body.Close()
	lr := http.MaxBytesReader(w, r.Body, maxRequestBody)
	b, err := io.ReadAll(lr)
	if err != nil {
		return b, err
	}
	if len(b) > maxRequestBody {
		return b, &http.MaxBytesError{Limit: maxRequestBody}
	}
	return b, nil
}

func validateSearch(rawQuery string, hasQuery bool) (string, bool) {
	if !hasQuery || rawQuery == "" || strings.Contains(rawQuery, "&") {
		return "", false
	}
	values, err := url.ParseQuery(rawQuery)
	if err != nil || len(values) != 1 {
		return "", false
	}
	v, ok := values["q"]
	if !ok || len(v) != 1 || len(v[0]) < 1 || len(v[0]) > 64 {
		return "", false
	}
	for i := 0; i < len(v[0]); i++ {
		c := v[0][i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == ' ' || c == '_' || c == '-') {
			return "", false
		}
	}
	return v[0], true
}

func validateFileID(rawPath string) (string, bool) {
	const prefix = "/v1/files/"
	if !strings.HasPrefix(rawPath, prefix) {
		return "", false
	}
	rawID := strings.TrimPrefix(rawPath, prefix)
	if rawID == "" || strings.Contains(rawID, "/") {
		return "", false
	}
	id, err := url.PathUnescape(rawID)
	if err != nil || len(id) < 1 || len(id) > 64 {
		return "", false
	}
	for i := 0; i < len(id); i++ {
		c := id[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '_' || c == '-') {
			return "", false
		}
	}
	return id, true
}

// For routes whose query semantics are not constrained, preserve the raw query
// but reject malformed escapes and fragments that would change URL parsing.
func validRawQuery(q string) bool {
	if strings.Contains(q, "#") {
		return false
	}
	for i := 0; i < len(q); i++ {
		if q[i] == '%' {
			if i+2 >= len(q) || !isHex(q[i+1]) || !isHex(q[i+2]) {
				return false
			}
			i += 2
		}
	}
	return true
}

func isHex(c byte) bool {
	return (c >= '0' && c <= '9') || (c >= 'a' && c <= 'f') || (c >= 'A' && c <= 'F')
}

func validHostname(host string) bool {
	if len(host) < 1 || len(host) > 253 || strings.HasSuffix(host, ".") {
		return false
	}
	for i := 0; i < len(host); i++ {
		if host[i] >= 0x80 {
			return false
		}
	}
	for _, label := range strings.Split(host, ".") {
		if len(label) < 1 || len(label) > 63 || label[0] == '-' || label[len(label)-1] == '-' {
			return false
		}
		for i := 0; i < len(label); i++ {
			c := label[i]
			if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '-') {
				return false
			}
		}
	}
	return true
}

func validateResolve(contentTypes []string, body []byte) (string, bool) {
	if len(contentTypes) != 1 {
		return "", false
	}
	mediaType, params, err := mime.ParseMediaType(contentTypes[0])
	if err != nil || !strings.EqualFold(mediaType, "application/json") {
		return "", false
	}
	if len(params) > 1 {
		return "", false
	}
	if len(params) == 1 {
		charset, ok := params["charset"]
		if !ok || !strings.EqualFold(charset, "utf-8") {
			return "", false
		}
	}
	if !utf8.Valid(body) {
		return "", false
	}
	dec := json.NewDecoder(bytes.NewReader(body))
	first, err := dec.Token()
	if err != nil || first != json.Delim('{') {
		return "", false
	}
	seenHost, host := false, ""
	for dec.More() {
		tok, e := dec.Token()
		if e != nil {
			return "", false
		}
		key, ok := tok.(string)
		if !ok || key != "host" || seenHost {
			return "", false
		}
		seenHost = true
		var raw json.RawMessage
		if e := dec.Decode(&raw); e != nil {
			return "", false
		}
		if e := json.Unmarshal(raw, &host); e != nil {
			return "", false
		}
	}
	last, err := dec.Token()
	if err != nil || last != json.Delim('}') || !seenHost {
		return "", false
	}
	var trailing json.RawMessage
	if err := dec.Decode(&trailing); err != io.EOF {
		return "", false
	}
	if !validHostname(host) {
		return "", false
	}
	return strings.ToLower(host), true
}

var hopHeaders = map[string]struct{}{
	"connection": {}, "proxy-connection": {}, "keep-alive": {},
	"proxy-authenticate": {}, "proxy-authorization": {}, "te": {},
	"trailer": {}, "transfer-encoding": {}, "upgrade": {},
}

func connectionNominations(h http.Header) map[string]struct{} {
	out := make(map[string]struct{})
	for _, value := range h.Values("Connection") {
		for _, part := range strings.Split(value, ",") {
			name := strings.TrimSpace(part)
			if name != "" {
				out[strings.ToLower(name)] = struct{}{}
			}
		}
	}
	return out
}

func forwardedRequestHeaders(dst, src http.Header) {
	nominated := connectionNominations(src)
	for _, name := range []string{"Accept", "Content-Type", "If-None-Match"} {
		lower := strings.ToLower(name)
		if _, hop := hopHeaders[lower]; hop {
			continue
		}
		if _, nom := nominated[lower]; nom {
			continue
		}
		for _, value := range src.Values(name) {
			dst.Add(name, value)
		}
	}
}

func forwardedResponseHeaders(dst, src http.Header) {
	nominated := connectionNominations(src)
	for name, values := range src {
		lower := strings.ToLower(name)
		if _, hop := hopHeaders[lower]; hop {
			continue
		}
		if _, nom := nominated[lower]; nom {
			continue
		}
		for _, value := range values {
			dst.Add(name, value)
		}
	}
}

func requestTooLarge(w http.ResponseWriter, r *http.Request) {
	r.Close = true
	w.Header().Set("Connection", "close")
	httpError(w, http.StatusRequestEntityTooLarge)
}

func methodNotAllowed(w http.ResponseWriter, allow string) {
	w.Header().Set("Allow", allow)
	httpError(w, http.StatusMethodNotAllowed)
}

func httpError(w http.ResponseWriter, status int) {
	msg := http.StatusText(status)
	if msg == "" {
		msg = "Request Failed"
	}
	http.Error(w, msg, status)
}

func (g *gateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	rawPath, rawQuery, hasQuery := rawPathAndQuery(r.RequestURI)
	if g.stopping.Load() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}
	if isSpecialPath(rawPath) {
		g.serveSpecial(w, r, rawPath)
		return
	}
	if !g.logs.reserve() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}

	tw := &trackedWriter{ResponseWriter: w}
	started := time.Now()
	identityRaw, identityOK := validIdentity(r.Header.Values("X-Verified-Client-ID"))
	clientID := "unverified"
	if identityOK {
		clientID = g.logs.pseudonym(identityRaw)
	}
	ev := logEvent{
		RequestID: g.logs.nextRequestID(),
		EventID:   g.logs.nextEventID(),
		Timestamp: started.UTC().Format(time.RFC3339Nano),
		Method:    safeMethod(r.Method),
		Route:     routeTemplate(rawPath),
		ClientID:  clientID,
	}
	reason, outcome := "internal_error", "error"
	defer func() {
		if recover() != nil {
			if !tw.wrote {
				httpError(tw, http.StatusInternalServerError)
			}
			reason, outcome = "internal_error", "error"
		}
		ev.Status = tw.statusCode()
		if reason == "client_cancelled" && !tw.wrote {
			ev.Status = 499
		}
		ev.Duration = time.Since(started).Milliseconds()
		ev.Reason, ev.Outcome = reason, outcome
		g.logs.emit(ev)
	}()

	select {
	case g.active <- struct{}{}:
		defer func() { <-g.active }()
	default:
		reason, outcome = "concurrency_limit", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}
	if g.stopping.Load() {
		reason, outcome = "shutting_down", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}
	if r.ProtoMajor != 1 || r.ProtoMinor != 1 {
		reason, outcome = "unsupported_protocol", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}
	if len(r.RequestURI) > maxTargetBytes {
		reason, outcome = "target_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}
	if headerLimitExceeded(r) {
		reason, outcome = "headers_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}
	if !identityOK {
		reason, outcome = "untrusted_identity", "rejected"
		httpError(tw, http.StatusUnauthorized)
		return
	}
	if r.ContentLength > maxRequestBody {
		reason, outcome = "body_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}

	body, bodyErr := readBodyLimited(tw, r)
	if bodyErr != nil {
		var maxErr *http.MaxBytesError
		if errors.As(bodyErr, &maxErr) {
			reason, outcome = "body_too_large", "rejected"
			requestTooLarge(tw, r)
			return
		}
		reason, outcome = "invalid_request_body", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}

	var (
		upstream      *origin
		upstreamPath  string
		upstreamQuery string
		forceQuery    bool
		method        string
		canonicalBody []byte
	)
	switch {
	case rawPath == "/v1/search":
		if r.Method != http.MethodGet {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "GET")
			return
		}
		term, ok := validateSearch(rawQuery, hasQuery)
		if !ok {
			reason, outcome = "invalid_search_query", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		upstream, upstreamPath, method = g.search, "/v1/search", http.MethodGet
		upstreamQuery = url.Values{"q": []string{term}}.Encode()
	case strings.HasPrefix(rawPath, "/v1/files/"):
		if r.Method != http.MethodGet {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "GET")
			return
		}
		fileID, ok := validateFileID(rawPath)
		if !ok || (hasQuery && !validRawQuery(rawQuery)) {
			reason, outcome = "invalid_file_request", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		upstream, upstreamPath, method = g.files, "/v1/files/"+url.PathEscape(fileID), http.MethodGet
		upstreamQuery, forceQuery = rawQuery, hasQuery && rawQuery == ""
	case rawPath == "/v1/tools/resolve":
		if r.Method != http.MethodPost {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "POST")
			return
		}
		if hasQuery && !validRawQuery(rawQuery) {
			reason, outcome = "invalid_resolve_query", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		host, ok := validateResolve(r.Header.Values("Content-Type"), body)
		if !ok {
			reason, outcome = "invalid_resolve_json", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		canonicalBody, _ = json.Marshal(struct {
			Host string `json:"host"`
		}{Host: host})
		upstream, upstreamPath, method = g.resolve, "/v1/tools/resolve", http.MethodPost
		upstreamQuery, forceQuery = rawQuery, hasQuery && rawQuery == ""
	default:
		reason, outcome = "not_found", "rejected"
		httpError(tw, http.StatusNotFound)
		return
	}

	allowed, retry, full := g.limiter.allow(identityRaw, time.Now())
	if full {
		reason, outcome = "rate_state_full", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}
	if !allowed {
		reason, outcome = "quota_exhausted", "rejected"
		seconds := int(math.Ceil(retry.Seconds()))
		if seconds < 1 {
			seconds = 1
		}
		tw.Header().Set("Retry-After", strconv.Itoa(seconds))
		httpError(tw, http.StatusTooManyRequests)
		return
	}

	var bodyReader io.Reader
	if method == http.MethodPost {
		bodyReader = bytes.NewReader(canonicalBody)
	}
	target := upstream.target(upstreamPath, upstreamQuery, forceQuery)
	upReq, err := http.NewRequestWithContext(r.Context(), method, target.String(), bodyReader)
	if err != nil {
		// Route-specific raw-query validation above keeps malformed client data
		// from being misreported as an upstream failure.
		reason, outcome = "upstream_request_failed", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	forwardedRequestHeaders(upReq.Header, r.Header)
	resp, err := g.client.Do(upReq)
	if err != nil {
		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			return
		}
		if isTimeout(err) {
			reason, outcome = "upstream_timeout", "error"
			httpError(tw, http.StatusGatewayTimeout)
			return
		}
		reason, outcome = "upstream_failure", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()
	if resp.ContentLength > maxResponseBody {
		reason, outcome = "upstream_response_too_large", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	responseBody, readErr := io.ReadAll(io.LimitReader(resp.Body, maxResponseBody+1))
	if len(responseBody) > maxResponseBody {
		reason, outcome = "upstream_response_too_large", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	if readErr != nil {
		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			return
		}
		if isTimeout(readErr) {
			reason, outcome = "upstream_timeout", "error"
			httpError(tw, http.StatusGatewayTimeout)
			return
		}
		reason, outcome = "upstream_failure", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	forwardedResponseHeaders(tw.Header(), resp.Header)
	tw.WriteHeader(resp.StatusCode)
	if len(responseBody) > 0 {
		if n, err := tw.Write(responseBody); err != nil || n != len(responseBody) {
			reason, outcome = "client_write_failed", "error"
			return
		}
	}
	reason, outcome = "upstream_response", "proxied"
}

func (g *gateway) serveSpecial(w http.ResponseWriter, r *http.Request, path string) {
	if len(r.RequestURI) > maxTargetBytes || headerLimitExceeded(r) {
		requestTooLarge(w, r)
		return
	}
	if r.ProtoMajor != 1 || r.ProtoMinor != 1 {
		httpError(w, http.StatusBadRequest)
		return
	}
	if r.ContentLength > maxRequestBody {
		requestTooLarge(w, r)
		return
	}
	body, err := readBodyLimited(w, r)
	if err != nil {
		var maxErr *http.MaxBytesError
		if errors.As(err, &maxErr) {
			requestTooLarge(w, r)
		} else {
			httpError(w, http.StatusBadRequest)
		}
		return
	}
	_ = body
	if r.Method != http.MethodGet {
		methodNotAllowed(w, "GET")
		return
	}
	if path == "/healthz" {
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		w.Header().Set("Cache-Control", "no-store")
		w.WriteHeader(http.StatusOK)
		_, _ = io.WriteString(w, "ok\n")
		return
	}
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.Header().Set("Cache-Control", "no-store")
	_ = json.NewEncoder(w).Encode(g.logs.metrics())
}

func isTimeout(err error) bool {
	if errors.Is(err, context.DeadlineExceeded) {
		return true
	}
	var ne net.Error
	return errors.As(err, &ne) && ne.Timeout()
}

func newHTTPClient() *http.Client {
	transport := &http.Transport{
		Proxy:                 nil,
		DialContext:           (&net.Dialer{Timeout: 2 * time.Second, KeepAlive: 30 * time.Second}).DialContext,
		ForceAttemptHTTP2:     false,
		MaxIdleConns:          512,
		MaxIdleConnsPerHost:   128,
		MaxConnsPerHost:       maxConcurrent,
		IdleConnTimeout:       30 * time.Second,
		ResponseHeaderTimeout: 2 * time.Second,
		DisableCompression:    true,
	}
	return &http.Client{
		Transport: transport,
		Timeout:   2 * time.Second,
		CheckRedirect: func(_ *http.Request, _ []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
}

func run() int {
	replica := os.Getenv("REPLICA_ID")
	if replica != "0" && replica != "1" && replica != "2" && replica != "3" {
		fmt.Fprintln(os.Stderr, "gateway: REPLICA_ID must be 0, 1, 2, or 3")
		return 2
	}
	search, err := parseOrigin(os.Getenv("SEARCH_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid SEARCH_BACKEND_URL")
		return 2
	}
	files, err := parseOrigin(os.Getenv("FILES_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid FILES_BACKEND_URL")
		return 2
	}
	resolve, err := parseOrigin(os.Getenv("RESOLVE_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid RESOLVE_BACKEND_URL")
		return 2
	}
	addr := os.Getenv("GATEWAY_ADDR")
	if addr == "" {
		addr = "127.0.0.1:8080"
	}
	logs, err := newLogManager()
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: unable to initialize logging")
		return 2
	}
	g := &gateway{
		search: search, files: files, resolve: resolve,
		client: newHTTPClient(), logs: logs, limiter: newRateLimiter(),
		active: make(chan struct{}, maxConcurrent),
	}
	server := &http.Server{
		Addr:              addr,
		Handler:           g,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      3 * time.Second,
		IdleTimeout:       30 * time.Second,
		MaxHeaderBytes:    maxHeaderBytes,
		ErrorLog:          log.New(io.Discard, "", 0),
	}
	listener, err := net.Listen("tcp", addr)
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: unable to listen")
		return 2
	}
	serveErr := make(chan error, 1)
	go func() { serveErr <- server.Serve(listener) }()

	signals := make(chan os.Signal, 1)
	signal.Notify(signals, syscall.SIGTERM, os.Interrupt)
	defer signal.Stop(signals)
	select {
	case <-signals:
		g.stopping.Store(true)
		server.SetKeepAlivesEnabled(false)
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 2500*time.Millisecond)
		shutdownDone := make(chan error, 1)
		go func() { shutdownDone <- server.Shutdown(shutdownCtx) }()
		select {
		case err := <-shutdownDone:
			if err != nil {
				_ = server.Close()
			}
		case <-shutdownCtx.Done():
			_ = server.Close()
		}
		cancel()
		// Best effort: an OS stdout write may block indefinitely and cannot be
		// portably interrupted. The process still exits by the three-second cap.
		deadline := time.Now().Add(500 * time.Millisecond)
		for time.Now().Before(deadline) {
			if atomic.LoadInt64(&logs.reserved) == 0 && len(logs.queue) == 0 {
				os.Exit(0)
			}
			time.Sleep(5 * time.Millisecond)
		}
		os.Exit(0)
		return 0
	case err := <-serveErr:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			fmt.Fprintln(os.Stderr, "gateway: server stopped unexpectedly")
			return 2
		}
		return 0
	}
}

func main() { os.Exit(run()) }

```


--- 


# Validation Layer Submission (Iteration 1)


Design Information: ( "DESIGN.txt" file ) 
``` 
Security invariant: No request reaches a backend until its route contract has been validated and its forwarded path, query, and body canonicalized. Search accepts exactly one validated q parameter and forwards its canonical encoding. File retrieval rejects extra path segments and any query string, decodes the identifier once, and forwards only the validated segment. Resolve rejects query strings, validates UTF-8 JSON while detecting duplicate decoded keys and trailing values, then forwards a lowercase hostname in canonical JSON. Only the ingress-verified identity controls quotas; logs contain a pseudonymous identifier.

Each replica owns a five-token bucket per identity, refilling at 0.5 tokens/second. Four static allocations cap the envelope at 20 burst tokens plus two tokens/second without a synchronous coordinator. Uneven traffic can strand quota; dynamic transfer, restart recovery, and control-plane coordination are unsupported.

Logging reserves bounded capacity before normal route processing and fails closed on saturation or output failure. Upstream responses are limited to 1 MiB each and globally limited to 128 concurrent response buffers to preserve memory headroom. Shutdown uses a signal-anchored best-effort drain deadline. A blocked stdout write cannot be portably interrupted, so reserved events may remain undelivered at the hard exit deadline. 

```


Code: ( "main.go" file ) 
``` 

package main

import (
	"bytes"
	"context"
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"math"
	"mime"
	"net"
	"net/http"
	"net/url"
	"os"
	"os/signal"
	"strconv"
	"strings"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
	"unicode/utf8"
)

const (
	maxHeaderBytes       = 16 << 10
	parserHeaderBytes    = 128 << 10
	maxBufferedResponses = 128
	maxTargetBytes       = 8 << 10
	maxRequestBody       = 64 << 10
	maxResponseBody      = 1 << 20
	maxLogEvents         = 8192
	maxConcurrent        = 512
	maxClients           = 10000
)

type origin struct {
	u *url.URL
}

func parseOrigin(raw string) (*origin, error) {
	if raw == "" {
		return nil, errors.New("missing origin")
	}
	u, err := url.Parse(raw)
	if err != nil || u == nil {
		return nil, errors.New("invalid origin")
	}
	if !strings.EqualFold(u.Scheme, "http") || strings.Contains(raw, "#") || u.Host == "" || u.User != nil || u.Opaque != "" || u.RawQuery != "" || u.ForceQuery || u.Fragment != "" || u.Path != "" || u.RawPath != "" {
		return nil, errors.New("invalid origin")
	}
	if u.Hostname() == "" {
		return nil, errors.New("invalid origin")
	}
	if p := u.Port(); p != "" {
		n, e := strconv.Atoi(p)
		if e != nil || n < 1 || n > 65535 {
			return nil, errors.New("invalid origin")
		}
	}
	u.Scheme = "http"
	return &origin{u: u}, nil
}

func (o *origin) target(path, rawQuery string) *url.URL {
	return &url.URL{Scheme: o.u.Scheme, Host: o.u.Host, Path: path, RawQuery: rawQuery}
}

type logEvent struct {
	RequestID string `json:"request_id"`
	EventID   string `json:"event_id"`
	Timestamp string `json:"timestamp"`
	Method    string `json:"method"`
	Route     string `json:"route"`
	Status    int    `json:"status"`
	Duration  int64  `json:"duration_ms"`
	Outcome   string `json:"outcome"`
	Reason    string `json:"reason"`
	ClientID  string `json:"client_id"`
}

type logManager struct {
	queue               chan []byte
	reserved            int64
	reservationFailures uint64
	deliveryFailures    uint64
	unhealthy           atomic.Bool
	key                 [32]byte
	bootID              string
	requestSequence     uint64
	eventSequence       uint64
	writer              io.Writer
}

func newLogManager() (*logManager, error) {
	m := &logManager{queue: make(chan []byte, maxLogEvents), writer: os.Stdout}
	var seed [40]byte
	if _, err := rand.Read(seed[:]); err != nil {
		return nil, err
	}
	copy(m.key[:], seed[:32])
	m.bootID = hex.EncodeToString(seed[32:40])
	go m.run()
	return m, nil
}

func saturatingIncrement(p *uint64) {
	for {
		old := atomic.LoadUint64(p)
		if old == ^uint64(0) {
			return
		}
		if atomic.CompareAndSwapUint64(p, old, old+1) {
			return
		}
	}
}

func (m *logManager) reserve() bool {
	if m.unhealthy.Load() {
		saturatingIncrement(&m.reservationFailures)
		return false
	}
	for {
		old := atomic.LoadInt64(&m.reserved)
		if old >= maxLogEvents {
			saturatingIncrement(&m.reservationFailures)
			return false
		}
		if atomic.CompareAndSwapInt64(&m.reserved, old, old+1) {
			if m.unhealthy.Load() {
				atomic.AddInt64(&m.reserved, -1)
				saturatingIncrement(&m.reservationFailures)
				return false
			}
			return true
		}
	}
}

func (m *logManager) nextRequestID() string {
	n := atomic.AddUint64(&m.requestSequence, 1)
	return "req-" + m.bootID + "-" + strconv.FormatUint(n, 36)
}

func (m *logManager) nextEventID() string {
	n := atomic.AddUint64(&m.eventSequence, 1)
	return "evt-" + m.bootID + "-" + strconv.FormatUint(n, 36)
}

func (m *logManager) pseudonym(identity string) string {
	mac := hmac.New(sha256.New, m.key[:])
	_, _ = mac.Write([]byte(identity))
	return hex.EncodeToString(mac.Sum(nil)[:12])
}

func (m *logManager) emit(ev logEvent) {
	b, err := json.Marshal(ev)
	if err != nil || len(b)+1 > 2048 {
		// All fields are bounded and controlled. This is a defensive fixed-size fallback.
		b = []byte(`{"request_id":"invalid","event_id":"invalid","timestamp":"1970-01-01T00:00:00Z","method":"OTHER","route":"unmatched","status":500,"duration_ms":0,"outcome":"error","reason":"event_encoding_failed","client_id":"unverified"}`)
	}
	b = append(b, '\n')
	// The reservation count includes requests that have not enqueued yet, so at least
	// one queue slot is available for this event while the capacity invariant holds.
	m.queue <- b
}

func (m *logManager) run() {
	for b := range m.queue {
		n, err := m.writer.Write(b)
		if err != nil || n != len(b) {
			m.unhealthy.Store(true)
			saturatingIncrement(&m.deliveryFailures)
		}
		atomic.AddInt64(&m.reserved, -1)
	}
}

type logMetrics struct {
	ReservationFailures uint64 `json:"logging_reservation_failures"`
	DeliveryFailures    uint64 `json:"logging_delivery_failures"`
	QueueDepth          int    `json:"queue_depth"`
	ReservationsInUse   int64  `json:"reservations_in_use"`
	Healthy             bool   `json:"logging_healthy"`
}

func (m *logManager) metrics() logMetrics {
	return logMetrics{
		ReservationFailures: atomic.LoadUint64(&m.reservationFailures),
		DeliveryFailures:    atomic.LoadUint64(&m.deliveryFailures),
		QueueDepth:          len(m.queue),
		ReservationsInUse:   atomic.LoadInt64(&m.reserved),
		Healthy:             !m.unhealthy.Load(),
	}
}

type bucketEntry struct {
	identity string
	tokens   float64
	last     time.Time
	fullAt   time.Time
	index    int
}

type bucketHeap []*bucketEntry

func (h bucketHeap) Len() int           { return len(h) }
func (h bucketHeap) Less(i, j int) bool { return h[i].fullAt.Before(h[j].fullAt) }
func (h bucketHeap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
	h[i].index = i
	h[j].index = j
}
func (h *bucketHeap) Push(x any) {
	b := x.(*bucketEntry)
	b.index = len(*h)
	*h = append(*h, b)
}
func (h *bucketHeap) Pop() any {
	old := *h
	n := len(old)
	b := old[n-1]
	old[n-1] = nil
	b.index = -1
	*h = old[:n-1]
	return b
}

func (h *bucketHeap) fix(b *bucketEntry) {
	if b.index >= 0 && b.index < len(*h) {
		// Small indexed heap: restore the deadline after each identity update.
		idx := b.index
		for idx > 0 {
			parent := (idx - 1) / 2
			if !(*h)[idx].fullAt.Before((*h)[parent].fullAt) {
				break
			}
			h.Swap(idx, parent)
			idx = parent
		}
		for {
			left := idx*2 + 1
			if left >= len(*h) {
				break
			}
			best := left
			right := left + 1
			if right < len(*h) && (*h)[right].fullAt.Before((*h)[left].fullAt) {
				best = right
			}
			if !(*h)[best].fullAt.Before((*h)[idx].fullAt) {
				break
			}
			h.Swap(idx, best)
			idx = best
		}
	}
}

type rateLimiter struct {
	mu      sync.Mutex
	buckets map[string]*bucketEntry
	heap    bucketHeap
}

func newRateLimiter() *rateLimiter {
	return &rateLimiter{buckets: make(map[string]*bucketEntry, maxClients), heap: make(bucketHeap, 0, maxClients)}
}

func bucketFullAt(now time.Time, tokens float64) time.Time {
	missing := 5.0 - tokens
	if missing <= 0 {
		return now
	}
	seconds := missing / 0.5
	return now.Add(time.Duration(seconds * float64(time.Second)))
}

// allow consumes one token. A non-nil retry duration indicates exhaustion; stateFull
// indicates a fail-closed response because no safely evictable state exists.
func (l *rateLimiter) allow(identity string) (allowed bool, retry time.Duration, stateFull bool) {
	l.mu.Lock()
	defer l.mu.Unlock()
	now := time.Now()

	b := l.buckets[identity]
	if b == nil {
		if len(l.buckets) >= maxClients {
			if len(l.heap) > 0 && !l.heap[0].fullAt.After(now) {
				candidate := l.heap[0]
				elapsed := now.Sub(candidate.last).Seconds()
				if elapsed < 0 {
					elapsed = 0
				}
				refilled := math.Min(5.0, candidate.tokens+elapsed*0.5)
				if refilled >= 5.0 {
					heapPopEntry(&l.heap)
					delete(l.buckets, candidate.identity)
				}
			}
			if len(l.buckets) >= maxClients {
				return false, 0, true
			}
		}
		b = &bucketEntry{identity: identity, tokens: 5.0, last: now, index: -1}
		b.fullAt = bucketFullAt(now, b.tokens)
		l.buckets[identity] = b
		heapPushEntry(&l.heap, b)
	}

	elapsed := now.Sub(b.last).Seconds()
	if elapsed < 0 {
		elapsed = 0
	}
	b.tokens = math.Min(5.0, b.tokens+elapsed*0.5)
	b.last = now
	if b.tokens < 1.0 {
		waitSeconds := (1.0 - b.tokens) / 0.5
		if waitSeconds < 0 {
			waitSeconds = 0
		}
		b.fullAt = bucketFullAt(now, b.tokens)
		heapFixEntry(&l.heap, b)
		return false, time.Duration(waitSeconds * float64(time.Second)), false
	}
	b.tokens -= 1.0
	b.fullAt = bucketFullAt(now, b.tokens)
	heapFixEntry(&l.heap, b)
	return true, 0, false
}

func heapPushEntry(h *bucketHeap, b *bucketEntry) {
	*h = append(*h, b)
	b.index = len(*h) - 1
	idx := b.index
	for idx > 0 {
		p := (idx - 1) / 2
		if !(*h)[idx].fullAt.Before((*h)[p].fullAt) {
			break
		}
		h.Swap(idx, p)
		idx = p
	}
}

func heapPopEntry(h *bucketHeap) *bucketEntry {
	if len(*h) == 0 {
		return nil
	}
	root := (*h)[0]
	last := (*h)[len(*h)-1]
	(*h)[len(*h)-1] = nil
	*h = (*h)[:len(*h)-1]
	root.index = -1
	if len(*h) == 0 {
		return root
	}
	(*h)[0] = last
	last.index = 0
	idx := 0
	for {
		left := idx*2 + 1
		if left >= len(*h) {
			break
		}
		best := left
		right := left + 1
		if right < len(*h) && (*h)[right].fullAt.Before((*h)[left].fullAt) {
			best = right
		}
		if !(*h)[best].fullAt.Before((*h)[idx].fullAt) {
			break
		}
		h.Swap(idx, best)
		idx = best
	}
	return root
}

func heapFixEntry(h *bucketHeap, b *bucketEntry) {
	if b.index < 0 || b.index >= len(*h) {
		return
	}
	idx := b.index
	for idx > 0 {
		p := (idx - 1) / 2
		if !(*h)[idx].fullAt.Before((*h)[p].fullAt) {
			break
		}
		h.Swap(idx, p)
		idx = p
	}
	for {
		left := idx*2 + 1
		if left >= len(*h) {
			break
		}
		best := left
		right := left + 1
		if right < len(*h) && (*h)[right].fullAt.Before((*h)[left].fullAt) {
			best = right
		}
		if !(*h)[best].fullAt.Before((*h)[idx].fullAt) {
			break
		}
		h.Swap(idx, best)
		idx = best
	}
}

type gateway struct {
	search          *origin
	files           *origin
	resolve         *origin
	client          *http.Client
	logs            *logManager
	limiter         *rateLimiter
	active          chan struct{}
	responseBuffers chan struct{}
	stopping        atomic.Bool
	replica         string
}

type trackedWriter struct {
	http.ResponseWriter
	status int
	wrote  bool
}

func (w *trackedWriter) WriteHeader(code int) {
	if w.wrote {
		return
	}
	w.status = code
	w.wrote = true
	w.ResponseWriter.WriteHeader(code)
}

func (w *trackedWriter) Write(p []byte) (int, error) {
	if !w.wrote {
		w.WriteHeader(http.StatusOK)
	}
	return w.ResponseWriter.Write(p)
}

func (w *trackedWriter) statusCode() int {
	if !w.wrote {
		return http.StatusOK
	}
	return w.status
}

func rawPathAndQuery(target string) (path, query string, hasQuery bool) {
	i := strings.IndexByte(target, '?')
	if i < 0 {
		return target, "", false
	}
	return target[:i], target[i+1:], true
}

func routeTemplate(rawPath string) string {
	switch {
	case rawPath == "/v1/search":
		return "/v1/search"
	case strings.HasPrefix(rawPath, "/v1/files/") && !strings.Contains(strings.TrimPrefix(rawPath, "/v1/files/"), "/"):
		return "/v1/files/{file_id}"
	case rawPath == "/v1/tools/resolve":
		return "/v1/tools/resolve"
	case rawPath == "/healthz":
		return "/healthz"
	case rawPath == "/_internal/metrics":
		return "/_internal/metrics"
	default:
		return "unmatched"
	}
}

func isSpecialPath(path string) bool {
	return path == "/healthz" || path == "/_internal/metrics"
}

func validIdentity(values []string) (string, bool) {
	if len(values) != 1 {
		return "", false
	}
	s := values[0]
	if len(s) < 1 || len(s) > 128 {
		return "", false
	}
	for i := 0; i < len(s); i++ {
		c := s[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '_' || c == '-') {
			return "", false
		}
	}
	return s, true
}

func safeMethod(method string) string {
	if len(method) > 32 {
		method = method[:32]
	}
	for i := 0; i < len(method); i++ {
		c := method[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || strings.ContainsRune("!#$%&'*+-.^_`|~", rune(c))) {
			return "OTHER"
		}
	}
	if method == "" {
		return "OTHER"
	}
	return method
}

func headerLimitExceeded(r *http.Request) bool {
	count := 0
	bytesSeen := 0
	for name, values := range r.Header {
		for _, value := range values {
			count++
			bytesSeen += len(name) + 2 + len(value) + 2
		}
	}
	// net/http promotes Host and framing metadata out of Header. Count each
	// parsed occurrence conservatively; parser ceiling bounds the raw wire size.
	if r.Host != "" {
		count++
		bytesSeen += len("Host") + 2 + len(r.Host) + 2
	}
	if len(r.TransferEncoding) > 0 {
		count += len(r.TransferEncoding)
	}
	if len(r.Trailer) > 0 {
		for name, values := range r.Trailer {
			for _, value := range values {
				count++
				bytesSeen += len(name) + len(value) + 4
			}
		}
	}
	if r.ContentLength >= 0 { /* Content-Length remains in Header on supported Go versions. */
	}
	return count > 64 || bytesSeen > maxHeaderBytes
}

func readBodyLimited(w http.ResponseWriter, r *http.Request) ([]byte, error) {
	defer r.Body.Close()
	lr := http.MaxBytesReader(w, r.Body, maxRequestBody)
	b, err := io.ReadAll(lr)
	if err != nil {
		return b, err
	}
	if len(b) > maxRequestBody {
		return b, &http.MaxBytesError{Limit: maxRequestBody}
	}
	return b, nil
}

func validateSearch(rawQuery string, hasQuery bool) (string, bool) {
	if !hasQuery || rawQuery == "" || strings.Contains(rawQuery, "&") {
		return "", false
	}
	values, err := url.ParseQuery(rawQuery)
	if err != nil || len(values) != 1 {
		return "", false
	}
	v, ok := values["q"]
	if !ok || len(v) != 1 || len(v[0]) < 1 || len(v[0]) > 64 {
		return "", false
	}
	for i := 0; i < len(v[0]); i++ {
		c := v[0][i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == ' ' || c == '_' || c == '-') {
			return "", false
		}
	}
	return v[0], true
}

func validateFileID(rawPath string) (string, bool) {
	const prefix = "/v1/files/"
	if !strings.HasPrefix(rawPath, prefix) {
		return "", false
	}
	rawID := strings.TrimPrefix(rawPath, prefix)
	if rawID == "" || strings.Contains(rawID, "/") {
		return "", false
	}
	id, err := url.PathUnescape(rawID)
	if err != nil || len(id) < 1 || len(id) > 64 {
		return "", false
	}
	for i := 0; i < len(id); i++ {
		c := id[i]
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '_' || c == '-') {
			return "", false
		}
	}
	return id, true
}

func validHostname(host string) bool {
	if len(host) < 1 || len(host) > 253 {
		return false
	}
	for i := 0; i < len(host); i++ {
		if host[i] >= 0x80 {
			return false
		}
	}
	if strings.HasSuffix(host, ".") {
		return false
	}
	labels := strings.Split(host, ".")
	for _, label := range labels {
		if len(label) < 1 || len(label) > 63 || label[0] == '-' || label[len(label)-1] == '-' {
			return false
		}
		for i := 0; i < len(label); i++ {
			c := label[i]
			if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '-') {
				return false
			}
		}
	}
	return true
}

func validateResolve(contentTypes []string, body []byte) (string, bool) {
	if len(contentTypes) != 1 {
		return "", false
	}
	mediaType, params, err := mime.ParseMediaType(contentTypes[0])
	if err != nil || !strings.EqualFold(mediaType, "application/json") {
		return "", false
	}
	if len(params) > 1 {
		return "", false
	}
	if len(params) == 1 {
		charset, ok := params["charset"]
		if !ok || !strings.EqualFold(charset, "utf-8") {
			return "", false
		}
	}
	if !utf8.Valid(body) {
		return "", false
	}
	dec := json.NewDecoder(bytes.NewReader(body))
	first, err := dec.Token()
	if err != nil || first != json.Delim('{') {
		return "", false
	}
	seenHost := false
	host := ""
	for dec.More() {
		tok, e := dec.Token()
		if e != nil {
			return "", false
		}
		key, ok := tok.(string)
		if !ok || key != "host" || seenHost {
			return "", false
		}
		seenHost = true
		var raw json.RawMessage
		if e := dec.Decode(&raw); e != nil {
			return "", false
		}
		if e := json.Unmarshal(raw, &host); e != nil {
			return "", false
		}
	}
	last, err := dec.Token()
	if err != nil || last != json.Delim('}') || !seenHost {
		return "", false
	}
	var trailing json.RawMessage
	if err := dec.Decode(&trailing); err != io.EOF {
		return "", false
	}
	if !validHostname(host) {
		return "", false
	}
	return strings.ToLower(host), true
}

var hopHeaders = map[string]struct{}{
	"connection": {}, "proxy-connection": {}, "keep-alive": {},
	"proxy-authenticate": {}, "proxy-authorization": {}, "te": {},
	"trailer": {}, "transfer-encoding": {}, "upgrade": {},
}

func connectionNominations(h http.Header) map[string]struct{} {
	out := make(map[string]struct{})
	for _, value := range h.Values("Connection") {
		for _, part := range strings.Split(value, ",") {
			name := strings.TrimSpace(part)
			if name != "" {
				out[strings.ToLower(name)] = struct{}{}
			}
		}
	}
	return out
}

func forwardedRequestHeaders(dst, src http.Header) {
	nominated := connectionNominations(src)
	for _, name := range []string{"Accept", "Content-Type", "If-None-Match"} {
		lower := strings.ToLower(name)
		if _, hop := hopHeaders[lower]; hop {
			continue
		}
		if _, nom := nominated[lower]; nom {
			continue
		}
		values := src.Values(name)
		for _, value := range values {
			dst.Add(name, value)
		}
	}
}

func forwardedResponseHeaders(dst, src http.Header) {
	nominated := connectionNominations(src)
	for name, values := range src {
		lower := strings.ToLower(name)
		if _, hop := hopHeaders[lower]; hop {
			continue
		}
		if _, nom := nominated[lower]; nom {
			continue
		}
		if lower == "x-verified-client-id" {
			continue
		}
		for _, value := range values {
			dst.Add(name, value)
		}
	}
}

func requestTooLarge(w http.ResponseWriter, r *http.Request) {
	r.Close = true
	w.Header().Set("Connection", "close")
	httpError(w, http.StatusRequestEntityTooLarge)
}

func methodNotAllowed(w http.ResponseWriter, allow string) {
	w.Header().Set("Allow", allow)
	httpError(w, http.StatusMethodNotAllowed)
}

func httpError(w http.ResponseWriter, status int) {
	msg := http.StatusText(status)
	if msg == "" {
		msg = "Request Failed"
	}
	http.Error(w, msg, status)
}

func (g *gateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	rawPath, rawQuery, hasQuery := rawPathAndQuery(r.RequestURI)
	if g.stopping.Load() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}
	if isSpecialPath(rawPath) {
		if _, ok := validIdentity(r.Header.Values("X-Verified-Client-ID")); !ok {
			httpError(w, http.StatusUnauthorized)
			return
		}
		select {
		case g.active <- struct{}{}:
			defer func() { <-g.active }()
		default:
			httpError(w, http.StatusServiceUnavailable)
			return
		}
		if g.stopping.Load() {
			httpError(w, http.StatusServiceUnavailable)
			return
		}
		g.serveSpecial(w, r, rawPath)
		return
	}
	if !g.logs.reserve() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}

	tw := &trackedWriter{ResponseWriter: w}
	started := time.Now()
	identityRaw, identityOK := validIdentity(r.Header.Values("X-Verified-Client-ID"))
	clientID := "unverified"
	if identityOK {
		clientID = g.logs.pseudonym(identityRaw)
	}
	requestID := g.logs.nextRequestID()
	ev := logEvent{
		RequestID: requestID,
		EventID:   g.logs.nextEventID(),
		Timestamp: started.UTC().Format(time.RFC3339Nano),
		Method:    safeMethod(r.Method),
		Route:     routeTemplate(rawPath),
		ClientID:  clientID,
	}
	reason := "internal_error"
	outcome := "error"
	defer func() {
		if recover() != nil {
			if !tw.wrote {
				httpError(tw, http.StatusInternalServerError)
			}
			reason = "internal_error"
			outcome = "error"
		}
		ev.Status = tw.statusCode()
		if reason == "client_cancelled" && !tw.wrote {
			ev.Status = 499
		}
		ev.Duration = time.Since(started).Milliseconds()
		ev.Reason = reason
		ev.Outcome = outcome
		g.logs.emit(ev)
	}()

	select {
	case g.active <- struct{}{}:
		defer func() { <-g.active }()
	default:
		reason, outcome = "concurrency_limit", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}

	if g.stopping.Load() {
		reason, outcome = "shutting_down", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}
	if r.ProtoMajor != 1 || r.ProtoMinor != 1 {
		reason, outcome = "unsupported_protocol", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}
	if len(r.RequestURI) > maxTargetBytes {
		reason, outcome = "target_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}
	if headerLimitExceeded(r) {
		reason, outcome = "headers_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}
	if !identityOK {
		reason, outcome = "untrusted_identity", "rejected"
		httpError(tw, http.StatusUnauthorized)
		return
	}
	if r.ContentLength > maxRequestBody {
		reason, outcome = "body_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}

	body, bodyErr := readBodyLimited(w, r)
	if bodyErr != nil {
		var maxErr *http.MaxBytesError
		if errors.As(bodyErr, &maxErr) {
			reason, outcome = "body_too_large", "rejected"
			requestTooLarge(tw, r)
			return
		}
		reason, outcome = "invalid_request_body", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}

	var (
		upstream      *origin
		upstreamPath  string
		upstreamQuery string
		method        string
		canonicalBody []byte
	)
	switch {
	case rawPath == "/v1/search":
		if r.Method != http.MethodGet {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "GET")
			return
		}
		term, ok := validateSearch(rawQuery, hasQuery)
		if !ok {
			reason, outcome = "invalid_search_query", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		upstream, upstreamPath, method = g.search, "/v1/search", http.MethodGet
		upstreamQuery = url.Values{"q": []string{term}}.Encode()
	case strings.HasPrefix(rawPath, "/v1/files/") && !strings.Contains(strings.TrimPrefix(rawPath, "/v1/files/"), "/"):
		if r.Method != http.MethodGet {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "GET")
			return
		}
		fileID, ok := validateFileID(rawPath)
		if !ok || hasQuery {
			reason, outcome = "invalid_file_id", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		upstream, upstreamPath, method = g.files, "/v1/files/"+url.PathEscape(fileID), http.MethodGet
		upstreamQuery = ""
	case rawPath == "/v1/tools/resolve":
		if r.Method != http.MethodPost {
			reason, outcome = "method_not_allowed", "rejected"
			methodNotAllowed(tw, "POST")
			return
		}
		host, ok := validateResolve(r.Header.Values("Content-Type"), body)
		if !ok || hasQuery {
			reason, outcome = "invalid_resolve_json", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}
		canonicalBody, _ = json.Marshal(struct {
			Host string `json:"host"`
		}{Host: host})
		upstream, upstreamPath, method = g.resolve, "/v1/tools/resolve", http.MethodPost
		upstreamQuery = ""
	default:
		reason, outcome = "not_found", "rejected"
		httpError(tw, http.StatusNotFound)
		return
	}

	allowed, retry, full := g.limiter.allow(identityRaw)
	if full {
		reason, outcome = "rate_state_full", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}
	if !allowed {
		reason, outcome = "quota_exhausted", "rejected"
		seconds := int(math.Ceil(retry.Seconds()))
		if seconds < 1 {
			seconds = 1
		}
		tw.Header().Set("Retry-After", strconv.Itoa(seconds))
		httpError(tw, http.StatusTooManyRequests)
		return
	}

	var bodyReader io.Reader
	if method == http.MethodPost {
		bodyReader = bytes.NewReader(canonicalBody)
	}
	select {
	case g.responseBuffers <- struct{}{}:
		defer func() { <-g.responseBuffers }()
	default:
		reason, outcome = "response_memory_budget", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}
	target := upstream.target(upstreamPath, upstreamQuery)
	upReq, err := http.NewRequestWithContext(r.Context(), method, target.String(), bodyReader)
	if err != nil {
		reason, outcome = "upstream_request_failed", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	forwardedRequestHeaders(upReq.Header, r.Header)
	if method == http.MethodPost {
		upReq.Header.Set("Content-Type", "application/json")
	}
	upReq.Header.Set("User-Agent", "")
	resp, err := g.client.Do(upReq)
	if err != nil {
		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			return
		}
		if isTimeout(err) {
			reason, outcome = "upstream_timeout", "error"
			httpError(tw, http.StatusGatewayTimeout)
			return
		}
		reason, outcome = "upstream_failure", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()
	if resp.ContentLength > maxResponseBody {
		reason, outcome = "upstream_response_too_large", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	responseBody, readErr := io.ReadAll(io.LimitReader(resp.Body, maxResponseBody+1))
	if len(responseBody) > maxResponseBody {
		reason, outcome = "upstream_response_too_large", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	if readErr != nil {
		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			return
		}
		if isTimeout(readErr) {
			reason, outcome = "upstream_timeout", "error"
			httpError(tw, http.StatusGatewayTimeout)
			return
		}
		reason, outcome = "upstream_failure", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	forwardedResponseHeaders(tw.Header(), resp.Header)
	tw.WriteHeader(resp.StatusCode)
	if len(responseBody) > 0 {
		if _, err := tw.Write(responseBody); err != nil {
			reason, outcome = "client_write_failed", "error"
			return
		}
	}
	reason, outcome = "upstream_response", "proxied"
}

func (g *gateway) serveSpecial(w http.ResponseWriter, r *http.Request, path string) {
	if len(r.RequestURI) > maxTargetBytes || headerLimitExceeded(r) {
		requestTooLarge(w, r)
		return
	}
	if r.ProtoMajor != 1 || r.ProtoMinor != 1 {
		httpError(w, http.StatusBadRequest)
		return
	}
	if r.Method != http.MethodGet {
		methodNotAllowed(w, "GET")
		return
	}
	if r.ContentLength > maxRequestBody {
		requestTooLarge(w, r)
		return
	}
	body, err := readBodyLimited(w, r)
	if err != nil {
		var maxErr *http.MaxBytesError
		if errors.As(err, &maxErr) {
			requestTooLarge(w, r)
		} else {
			httpError(w, http.StatusBadRequest)
		}
		return
	}
	_ = body
	if path == "/healthz" {
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		w.Header().Set("Cache-Control", "no-store")
		w.WriteHeader(http.StatusOK)
		_, _ = io.WriteString(w, "ok\n")
		return
	}
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.Header().Set("Cache-Control", "no-store")
	_ = json.NewEncoder(w).Encode(g.logs.metrics())
}

func isTimeout(err error) bool {
	if errors.Is(err, context.DeadlineExceeded) {
		return true
	}
	var ne net.Error
	return errors.As(err, &ne) && ne.Timeout()
}

func newHTTPClient() *http.Client {
	transport := &http.Transport{
		Proxy:                  nil,
		DialContext:            (&net.Dialer{Timeout: 2 * time.Second, KeepAlive: 30 * time.Second}).DialContext,
		ForceAttemptHTTP2:      false,
		MaxIdleConns:           512,
		MaxIdleConnsPerHost:    128,
		MaxConnsPerHost:        maxConcurrent,
		MaxResponseHeaderBytes: 16 << 10,
		IdleConnTimeout:        30 * time.Second,
		ResponseHeaderTimeout:  2 * time.Second,
		DisableCompression:     true,
	}
	return &http.Client{
		Transport: transport,
		Timeout:   2 * time.Second,
		CheckRedirect: func(_ *http.Request, _ []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
}

func run() int {
	signal.Ignore(syscall.SIGPIPE)
	replica := os.Getenv("REPLICA_ID")
	if replica != "0" && replica != "1" && replica != "2" && replica != "3" {
		fmt.Fprintln(os.Stderr, "gateway: REPLICA_ID must be 0, 1, 2, or 3")
		return 2
	}
	search, err := parseOrigin(os.Getenv("SEARCH_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid SEARCH_BACKEND_URL")
		return 2
	}
	files, err := parseOrigin(os.Getenv("FILES_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid FILES_BACKEND_URL")
		return 2
	}
	resolve, err := parseOrigin(os.Getenv("RESOLVE_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid RESOLVE_BACKEND_URL")
		return 2
	}
	addr := os.Getenv("GATEWAY_ADDR")
	if addr == "" {
		addr = "127.0.0.1:8080"
	}
	logs, err := newLogManager()
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: unable to initialize logging")
		return 2
	}
	g := &gateway{
		search: search, files: files, resolve: resolve,
		client: newHTTPClient(), logs: logs, limiter: newRateLimiter(),
		active: make(chan struct{}, maxConcurrent), responseBuffers: make(chan struct{}, maxBufferedResponses), replica: replica,
	}
	server := &http.Server{
		Addr:              addr,
		Handler:           g,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      3 * time.Second,
		IdleTimeout:       30 * time.Second,
		MaxHeaderBytes:    parserHeaderBytes,
		ErrorLog:          log.New(io.Discard, "", 0),
	}
	listener, err := net.Listen("tcp", addr)
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: unable to listen")
		return 2
	}
	serveErr := make(chan error, 1)
	go func() { serveErr <- server.Serve(listener) }()

	signals := make(chan os.Signal, 1)
	signal.Notify(signals, syscall.SIGTERM, os.Interrupt)
	defer signal.Stop(signals)
	select {
	case <-signals:
		signalAt := time.Now()
		// From this point the handler gate prevents new application dispatches.
		g.stopping.Store(true)
		server.SetKeepAlivesEnabled(false)
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 2500*time.Millisecond)
		shutdownDone := make(chan error, 1)
		go func() { shutdownDone <- server.Shutdown(shutdownCtx) }()
		select {
		case err := <-shutdownDone:
			cancel()
			if err != nil {
				_ = server.Close()
			}
		case <-shutdownCtx.Done():
			_ = server.Close()
			cancel()
		case <-time.After(2500 * time.Millisecond):
			_ = server.Close()
			cancel()
		}
		// Best effort to deliver everything already reserved. A blocked stdout write
		// cannot be interrupted portably; os.Exit at the deadline bounds shutdown.
		deadline := signalAt.Add(2850 * time.Millisecond)
		for time.Now().Before(deadline) {
			if atomic.LoadInt64(&logs.reserved) == 0 && len(logs.queue) == 0 {
				os.Exit(0)
			}
			time.Sleep(5 * time.Millisecond)
		}
		os.Exit(0)
		return 0
	case err := <-serveErr:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			fmt.Fprintln(os.Stderr, "gateway: server stopped unexpectedly")
			return 2
		}
		return 0
	}
}

func main() {
	os.Exit(run())
}

``` 


--- 


# Validation Layer Submission (Iteration 2)


Design Information: ( "DESIGN.txt" file ) 
``` 
Security invariant: No backend is contacted until the request method and route contract pass validation. Search input is decoded once and re-encoded canonically; file IDs are decoded once and forwarded as one allowed segment; resolve requires one UTF-8 JSON `host` key, detects duplicate decoded keys, and forwards lowercase canonical JSON. Files and resolve reject query strings. Only the ingress-verified identity controls quotas; logs use HMAC pseudonyms and never include raw input or secrets.

Each replica independently enforces a five-token bucket refilling at 0.5 tokens/second per identity. Four static allocations cap the aggregate envelope at 20 burst tokens plus 2 tokens/second without a synchronous coordinator. Uneven traffic and failed replicas can strand quota; dynamic transfer, restart recovery, and control-plane coordination are unsupported.

Logging capacity is reserved before ordinary route processing. Response buffering is limited to 128 concurrent buffers, each no larger than 1 MiB. On SIGTERM, active work may finish for up to 2.5 seconds before cancellation and a bounded log drain. A stalled stdout write cannot be portably interrupted, so events outstanding at the hard exit deadline may be lost; draining is best effort. 

```


Code: ( "main.go" file ) 
``` 

package main

import (
	"bytes"
	"container/heap"
	"context"
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"math"
	"mime"
	"net"
	"net/http"
	"net/url"
	"os"
	"os/signal"
	"strconv"
	"strings"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
	"unicode/utf8"
)

const (
	maxHeaderBytes       = 16 << 10
	parserHeaderBytes    = 128 << 10
	maxTargetBytes       = 8 << 10
	maxRequestBody       = 64 << 10
	maxResponseBody      = 1 << 20
	maxLogEvents         = 8192
	maxConcurrent        = 512
	maxClients           = 10000
	maxResponseBuffers   = 128
	maxHeaderScanBytes   = parserHeaderBytes + (8 << 10)
	maxHeaderMetaPending = 1024
)

type origin struct {
	u *url.URL
}

func parseOrigin(raw string) (*origin, error) {
	if raw == "" {
		return nil, errors.New("missing origin")
	}

	u, err := url.Parse(raw)
	if err != nil || u == nil {
		return nil, errors.New("invalid origin")
	}

	if !strings.EqualFold(u.Scheme, "http") ||
		u.Host == "" ||
		u.User != nil ||
		u.Opaque != "" ||
		u.Path != "" ||
		u.RawPath != "" ||
		u.RawQuery != "" ||
		u.ForceQuery ||
		u.Fragment != "" ||
		strings.Contains(raw, "#") {
		return nil, errors.New("invalid origin")
	}

	if u.Hostname() == "" {
		return nil, errors.New("invalid origin")
	}

	if p := u.Port(); p != "" {
		n, e := strconv.Atoi(p)
		if e != nil || n < 1 || n > 65535 {
			return nil, errors.New("invalid origin")
		}
	}

	u.Scheme = "http"
	return &origin{u: u}, nil
}

func (o *origin) target(path, rawQuery string) *url.URL {
	return &url.URL{
		Scheme:   o.u.Scheme,
		Host:     o.u.Host,
		Path:     path,
		RawQuery: rawQuery,
	}
}

type headerMeta struct {
	targetBytes int
	headerBytes int
	fields      int
	valid       bool
}

const (
	scanHeader = iota
	scanFixedBody
	scanChunkSize
	scanChunkData
	scanChunkCRLF
	scanTrailers
)

// accountingConn observes request bytes without changing them or replacing
// net/http's parser. Its framing scanner associates raw header counts with
// parsed requests; net/http remains responsible for HTTP framing validation.
type accountingConn struct {
	net.Conn

	mu          sync.Mutex
	state       int
	headerBuf   []byte
	headerTail  []byte
	headerDrop  bool
	lineBuf     []byte
	bodyRemain  int64
	chunkCRLF   int
	pending     []headerMeta
	pendingHead int
}

func newAccountingConn(c net.Conn) *accountingConn {
	return &accountingConn{
		Conn:       c,
		state:      scanHeader,
		headerBuf:  make([]byte, 0, 1024),
		headerTail: make([]byte, 0, 4),
		lineBuf:    make([]byte, 0, 128),
		pending:    make([]headerMeta, 0, 8),
	}
}

func (c *accountingConn) Read(p []byte) (int, error) {
	n, err := c.Conn.Read(p)
	if n > 0 {
		c.observe(p[:n])
	}
	return n, err
}

func (c *accountingConn) pushMeta(m headerMeta) {
	if len(c.pending)-c.pendingHead >= maxHeaderMetaPending {
		return
	}

	if c.pendingHead > 0 && c.pendingHead >= len(c.pending)/2 {
		copy(c.pending, c.pending[c.pendingHead:])
		c.pending = c.pending[:len(c.pending)-c.pendingHead]
		c.pendingHead = 0
	}

	c.pending = append(c.pending, m)
}

func (c *accountingConn) nextMeta() (headerMeta, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()

	if c.pendingHead >= len(c.pending) {
		return headerMeta{}, false
	}

	m := c.pending[c.pendingHead]
	c.pendingHead++

	if c.pendingHead == len(c.pending) {
		c.pending = c.pending[:0]
		c.pendingHead = 0
	}

	return m, true
}

func parseRawHeaderBlock(block []byte) (headerMeta, int, int64) {
	m := headerMeta{valid: true}

	first := bytes.Index(block, []byte("\r\n"))
	if first < 0 {
		return headerMeta{}, scanHeader, 0
	}

	requestLine := strings.Fields(string(block[:first]))
	if len(requestLine) >= 2 {
		m.targetBytes = len(requestLine[1])
	}

	// Count raw header bytes independently of the request line.
	m.headerBytes = len(block) - (first + 2)
	if m.headerBytes < 0 {
		m.headerBytes = 0
	}

	var contentLength int64
	chunked := false

	// Field lines retain their CRLF; omit only the final CRLF of the block.
	sectionEnd := len(block) - 2
	if sectionEnd > first+2 {
		section := string(block[first+2 : sectionEnd])

		for _, line := range strings.Split(section, "\r\n") {
			if line == "" || line[0] == ' ' || line[0] == '\t' {
				continue
			}

			colon := strings.IndexByte(line, ':')
			if colon <= 0 {
				continue
			}

			m.fields++

			name := strings.TrimSpace(line[:colon])
			value := strings.TrimSpace(line[colon+1:])

			switch {
			case strings.EqualFold(name, "Transfer-Encoding") &&
				strings.Contains(strings.ToLower(value), "chunked"):
				chunked = true

			case strings.EqualFold(name, "Content-Length"):
				if n, err := strconv.ParseInt(value, 10, 64); err == nil && n > 0 {
					contentLength = n
				}
			}
		}
	}

	if chunked {
		return m, scanChunkSize, 0
	}
	if contentLength > 0 {
		return m, scanFixedBody, contentLength
	}

	return m, scanHeader, 0
}

func (c *accountingConn) finishHeader() {
	m, state, remain := parseRawHeaderBlock(c.headerBuf)
	c.pushMeta(m)

	c.headerBuf = c.headerBuf[:0]
	c.headerTail = c.headerTail[:0]
	c.headerDrop = false
	c.lineBuf = c.lineBuf[:0]
	c.bodyRemain = remain

	switch state {
	case scanFixedBody:
		if remain > 0 {
			c.state = scanFixedBody
		} else {
			c.state = scanHeader
		}
	case scanChunkSize:
		c.state = scanChunkSize
	default:
		c.state = scanHeader
	}
}

func (c *accountingConn) observe(p []byte) {
	c.mu.Lock()
	defer c.mu.Unlock()

	for _, b := range p {
		switch c.state {
		case scanHeader:
			if !c.headerDrop {
				if len(c.headerBuf) < maxHeaderScanBytes {
					c.headerBuf = append(c.headerBuf, b)
				} else {
					c.headerDrop = true
					c.headerBuf = c.headerBuf[:0]
				}
			}

			c.headerTail = append(c.headerTail, b)
			if len(c.headerTail) > 4 {
				copy(c.headerTail, c.headerTail[len(c.headerTail)-4:])
				c.headerTail = c.headerTail[:4]
			}

			if len(c.headerTail) == 4 &&
				bytes.Equal(c.headerTail, []byte("\r\n\r\n")) {
				if !c.headerDrop {
					c.finishHeader()
				} else {
					c.headerBuf = c.headerBuf[:0]
					c.headerTail = c.headerTail[:0]
					c.headerDrop = false
					c.state = scanHeader
				}
			}

		case scanFixedBody:
			c.bodyRemain--
			if c.bodyRemain <= 0 {
				c.bodyRemain = 0
				c.state = scanHeader
				c.headerBuf = c.headerBuf[:0]
				c.headerTail = c.headerTail[:0]
			}

		case scanChunkSize:
			c.lineBuf = append(c.lineBuf, b)

			if len(c.lineBuf) >= 2 &&
				c.lineBuf[len(c.lineBuf)-2] == '\r' &&
				c.lineBuf[len(c.lineBuf)-1] == '\n' {

				line := string(c.lineBuf[:len(c.lineBuf)-2])
				c.lineBuf = c.lineBuf[:0]

				if semicolon := strings.IndexByte(line, ';'); semicolon >= 0 {
					line = line[:semicolon]
				}

				n, err := strconv.ParseUint(strings.TrimSpace(line), 16, 63)
				if err != nil {
					// Observation only; net/http handles malformed framing.
					c.state = scanHeader
				} else if n == 0 {
					c.state = scanTrailers
				} else {
					c.bodyRemain = int64(n)
					c.state = scanChunkData
				}
			} else if len(c.lineBuf) > 8192 {
				c.lineBuf = c.lineBuf[:0]
				c.state = scanHeader
			}

		case scanChunkData:
			c.bodyRemain--
			if c.bodyRemain <= 0 {
				c.chunkCRLF = 2
				c.state = scanChunkCRLF
			}

		case scanChunkCRLF:
			c.chunkCRLF--
			if c.chunkCRLF <= 0 {
				c.state = scanChunkSize
			}

		case scanTrailers:
			c.lineBuf = append(c.lineBuf, b)

			if len(c.lineBuf) >= 2 &&
				c.lineBuf[len(c.lineBuf)-2] == '\r' &&
				c.lineBuf[len(c.lineBuf)-1] == '\n' {

				if len(c.lineBuf) == 2 {
					c.state = scanHeader
					c.headerBuf = c.headerBuf[:0]
					c.headerTail = c.headerTail[:0]
				}

				c.lineBuf = c.lineBuf[:0]
			} else if len(c.lineBuf) > 8192 {
				c.lineBuf = c.lineBuf[:0]
				c.state = scanHeader
			}
		}
	}
}

type accountingListener struct {
	net.Listener
}

func (l accountingListener) Accept() (net.Conn, error) {
	c, err := l.Listener.Accept()
	if err != nil {
		return nil, err
	}
	return newAccountingConn(c), nil
}

type connContextKey struct{}

func rawHeaderMeta(r *http.Request) (headerMeta, bool) {
	if r == nil {
		return headerMeta{}, false
	}

	if ac, ok := r.Context().Value(connContextKey{}).(*accountingConn); ok {
		return ac.nextMeta()
	}

	return headerMeta{}, false
}

type logEvent struct {
	RequestID string `json:"request_id"`
	EventID   string `json:"event_id"`
	Timestamp string `json:"timestamp"`
	Method    string `json:"method"`
	Route     string `json:"route"`
	Status    int    `json:"status"`
	Duration  int64  `json:"duration_ms"`
	Outcome   string `json:"outcome"`
	Reason    string `json:"reason"`
	ClientID  string `json:"client_id"`
}

type logManager struct {
	queue               chan []byte
	writer              io.Writer
	key                 [32]byte
	bootID              string
	requestSequence     atomic.Uint64
	eventSequence       atomic.Uint64
	admitMu             sync.Mutex
	reserved            int
	unhealthy           bool
	reservationFailures atomic.Uint64
	deliveryFailures    atomic.Uint64
}

func newLogManager() (*logManager, error) {
	m := &logManager{
		queue:  make(chan []byte, maxLogEvents),
		writer: os.Stdout,
	}

	var seed [40]byte
	if _, err := rand.Read(seed[:]); err != nil {
		return nil, err
	}

	copy(m.key[:], seed[:32])
	m.bootID = hex.EncodeToString(seed[32:])

	go m.run()
	return m, nil
}

func saturatingIncrement(p *atomic.Uint64) {
	for {
		old := p.Load()
		if old == ^uint64(0) || p.CompareAndSwap(old, old+1) {
			return
		}
	}
}

func (m *logManager) reserve() bool {
	m.admitMu.Lock()
	defer m.admitMu.Unlock()

	if m.unhealthy || m.reserved >= maxLogEvents {
		saturatingIncrement(&m.reservationFailures)
		return false
	}

	m.reserved++
	return true
}

func (m *logManager) releaseReservation() {
	m.admitMu.Lock()
	if m.reserved > 0 {
		m.reserved--
	}
	m.admitMu.Unlock()
}

func (m *logManager) nextRequestID() string {
	n := m.requestSequence.Add(1)
	return "req-" + m.bootID + "-" + strconv.FormatUint(n, 36)
}

func (m *logManager) nextEventID() string {
	n := m.eventSequence.Add(1)
	return "evt-" + m.bootID + "-" + strconv.FormatUint(n, 36)
}

func (m *logManager) pseudonym(identity string) string {
	mac := hmac.New(sha256.New, m.key[:])
	_, _ = mac.Write([]byte(identity))
	return hex.EncodeToString(mac.Sum(nil)[:12])
}

func (m *logManager) emit(ev logEvent) {
	b, err := json.Marshal(ev)
	if err != nil || len(b)+1 > 2048 {
		b = []byte(`{"request_id":"invalid","event_id":"invalid","timestamp":"1970-01-01T00:00:00Z","method":"OTHER","route":"unmatched","status":500,"duration_ms":0,"outcome":"error","reason":"event_encoding_failed","client_id":"unverified"}`)
	}

	b = append(b, '\n')

	// Queued and not-yet-emitted events each own a reservation. The total
	// reservation limit therefore prevents this bounded send from overflowing.
	m.queue <- b
}

func (m *logManager) run() {
	for b := range m.queue {
		n, err := m.writer.Write(b)
		if err != nil || n != len(b) {
			m.admitMu.Lock()
			m.unhealthy = true
			m.admitMu.Unlock()
			saturatingIncrement(&m.deliveryFailures)
		}

		m.releaseReservation()
	}
}

type logMetrics struct {
	ReservationFailures uint64 `json:"logging_reservation_failures"`
	DeliveryFailures    uint64 `json:"logging_delivery_failures"`
	QueueDepth          int    `json:"queue_depth"`
	ReservationsInUse   int    `json:"reservations_in_use"`
	Healthy             bool   `json:"logging_healthy"`
}

func (m *logManager) metrics() logMetrics {
	m.admitMu.Lock()
	reserved, healthy := m.reserved, !m.unhealthy
	m.admitMu.Unlock()

	return logMetrics{
		ReservationFailures: m.reservationFailures.Load(),
		DeliveryFailures:    m.deliveryFailures.Load(),
		QueueDepth:          len(m.queue),
		ReservationsInUse:   reserved,
		Healthy:             healthy,
	}
}

func (m *logManager) drained() bool {
	m.admitMu.Lock()
	reserved := m.reserved
	m.admitMu.Unlock()

	return reserved == 0 && len(m.queue) == 0
}

type bucketEntry struct {
	identity string
	tokens   float64
	last     time.Time
	fullAt   time.Time
	index    int
}

type bucketHeap []*bucketEntry

func (h bucketHeap) Len() int {
	return len(h)
}

func (h bucketHeap) Less(i, j int) bool {
	return h[i].fullAt.Before(h[j].fullAt)
}

func (h bucketHeap) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
	h[i].index = i
	h[j].index = j
}

func (h *bucketHeap) Push(x any) {
	b := x.(*bucketEntry)
	b.index = len(*h)
	*h = append(*h, b)
}

func (h *bucketHeap) Pop() any {
	old := *h
	n := len(old)
	b := old[n-1]
	old[n-1] = nil
	b.index = -1
	*h = old[:n-1]
	return b
}

func refillDuration(tokens float64) time.Duration {
	missing := 5.0 - tokens
	if missing <= 0 {
		return 0
	}

	return time.Duration(math.Ceil((missing / 0.5) * float64(time.Second)))
}

func refreshFullAt(b *bucketEntry) {
	b.fullAt = b.last.Add(refillDuration(b.tokens))
}

type rateLimiter struct {
	mu      sync.Mutex
	buckets map[string]*bucketEntry
	heap    bucketHeap
}

func newRateLimiter() *rateLimiter {
	return &rateLimiter{
		buckets: make(map[string]*bucketEntry, maxClients),
		heap:    make(bucketHeap, 0, maxClients),
	}
}

// allow samples time after taking the mutex, preventing goroutines from
// applying timestamps out of order and accidentally crediting quota twice.
func (l *rateLimiter) allow(identity string) (allowed bool, retry time.Duration, stateFull bool) {
	l.mu.Lock()
	defer l.mu.Unlock()

	now := time.Now()
	b := l.buckets[identity]

	if b == nil {
		if len(l.buckets) >= maxClients {
			for len(l.heap) > 0 && !l.heap[0].fullAt.After(now) {
				candidate := l.heap[0]
				needed := refillDuration(candidate.tokens)

				if now.Sub(candidate.last) < needed {
					candidate.fullAt = candidate.last.Add(needed)
					heap.Fix(&l.heap, candidate.index)
					break
				}

				heap.Pop(&l.heap)
				delete(l.buckets, candidate.identity)
				break
			}

			if len(l.buckets) >= maxClients {
				return false, 0, true
			}
		}

		b = &bucketEntry{
			identity: identity,
			tokens:   5.0,
			last:     now,
			index:    -1,
		}
		refreshFullAt(b)
		l.buckets[identity] = b
		heap.Push(&l.heap, b)
	}

	elapsed := now.Sub(b.last).Seconds()
	if elapsed > 0 {
		b.tokens = math.Min(5.0, b.tokens+elapsed*0.5)
		b.last = now
	}

	if b.tokens < 1.0 {
		waitSeconds := (1.0 - b.tokens) / 0.5
		if waitSeconds < 0 {
			waitSeconds = 0
		}

		refreshFullAt(b)
		heap.Fix(&l.heap, b.index)
		return false, time.Duration(math.Ceil(waitSeconds * float64(time.Second))), false
	}

	b.tokens--
	refreshFullAt(b)
	heap.Fix(&l.heap, b.index)
	return true, 0, false
}

type gateway struct {
	search          *origin
	files           *origin
	resolve         *origin
	client          *http.Client
	logs            *logManager
	limiter         *rateLimiter
	active          chan struct{}
	responseBuffers chan struct{}
	stopping        atomic.Bool
	rootCtx         context.Context
	cancelWork      context.CancelFunc
}

type trackedWriter struct {
	http.ResponseWriter
	status int
	wrote  bool
}

func (w *trackedWriter) WriteHeader(code int) {
	if w.wrote {
		return
	}
	w.status = code
	w.wrote = true
	w.ResponseWriter.WriteHeader(code)
}

func (w *trackedWriter) Write(p []byte) (int, error) {
	if !w.wrote {
		w.WriteHeader(http.StatusOK)
	}
	return w.ResponseWriter.Write(p)
}

func (w *trackedWriter) statusCode() int {
	if !w.wrote {
		return http.StatusOK
	}
	return w.status
}

func rawPathAndQuery(target string) (path, query string, hasQuery bool) {
	i := strings.IndexByte(target, '?')
	if i < 0 {
		return target, "", false
	}
	return target[:i], target[i+1:], true
}

type routeKind int

const (
	routeUnknown routeKind = iota
	routeSearch
	routeFiles
	routeResolve
	routeHealth
	routeMetrics
)

func classifyRoute(rawPath string) routeKind {
	switch {
	case rawPath == "/v1/search":
		return routeSearch

	case strings.HasPrefix(rawPath, "/v1/files/") &&
		!strings.Contains(strings.TrimPrefix(rawPath, "/v1/files/"), "/"):
		return routeFiles

	case rawPath == "/v1/tools/resolve":
		return routeResolve

	case rawPath == "/healthz":
		return routeHealth

	case rawPath == "/_internal/metrics":
		return routeMetrics

	default:
		return routeUnknown
	}
}

func routeTemplate(rawPath string) string {
	switch classifyRoute(rawPath) {
	case routeSearch:
		return "/v1/search"
	case routeFiles:
		return "/v1/files/{file_id}"
	case routeResolve:
		return "/v1/tools/resolve"
	case routeHealth:
		return "/healthz"
	case routeMetrics:
		return "/_internal/metrics"
	default:
		return "unmatched"
	}
}

func validIdentity(values []string) (string, bool) {
	if len(values) != 1 {
		return "", false
	}

	s := values[0]
	if len(s) < 1 || len(s) > 128 {
		return "", false
	}

	for i := 0; i < len(s); i++ {
		c := s[i]
		if !((c >= 'a' && c <= 'z') ||
			(c >= 'A' && c <= 'Z') ||
			(c >= '0' && c <= '9') ||
			c == '_' || c == '-') {
			return "", false
		}
	}

	return s, true
}

func safeMethod(method string) string {
	if method == "" || len(method) > 32 {
		return "OTHER"
	}

	for i := 0; i < len(method); i++ {
		c := method[i]
		if !((c >= 'a' && c <= 'z') ||
			(c >= 'A' && c <= 'Z') ||
			(c >= '0' && c <= '9') ||
			strings.ContainsRune("!#$%&'*+-.^_`|~", rune(c))) {
			return "OTHER"
		}
	}

	return method
}

func requestTargetTooLarge(r *http.Request, meta headerMeta, haveMeta bool) bool {
	if haveMeta && meta.valid {
		return meta.targetBytes > maxTargetBytes
	}
	return len(r.RequestURI) > maxTargetBytes
}

func headerLimitExceeded(r *http.Request, meta headerMeta, haveMeta bool) bool {
	if haveMeta && meta.valid {
		return meta.fields > 64 || meta.headerBytes > maxHeaderBytes
	}

	// Fallback for direct handler use without the raw-connection accounting.
	count, bytesSeen := 0, 0
	seenTE := false

	for name, values := range r.Header {
		if strings.EqualFold(name, "Transfer-Encoding") {
			seenTE = true
		}
		for _, value := range values {
			count++
			bytesSeen += len(name) + 2 + len(value) + 2
		}
	}

	if r.Host != "" {
		count++
		bytesSeen += len("Host") + 2 + len(r.Host) + 2
	}

	if len(r.TransferEncoding) > 0 && !seenTE {
		for _, value := range r.TransferEncoding {
			count++
			bytesSeen += len("Transfer-Encoding") + 2 + len(value) + 2
		}
	}

	return count > 64 || bytesSeen > maxHeaderBytes
}

func readBodyLimited(w http.ResponseWriter, r *http.Request) ([]byte, error) {
	defer r.Body.Close()

	lr := http.MaxBytesReader(w, r.Body, maxRequestBody)
	b, err := io.ReadAll(lr)
	if err != nil {
		return b, err
	}

	if len(b) > maxRequestBody {
		return b, &http.MaxBytesError{Limit: maxRequestBody}
	}

	return b, nil
}

func validateSearch(rawQuery string, hasQuery bool) (string, bool) {
	if !hasQuery || rawQuery == "" || strings.Contains(rawQuery, "&") {
		return "", false
	}

	values, err := url.ParseQuery(rawQuery)
	if err != nil || len(values) != 1 {
		return "", false
	}

	v, ok := values["q"]
	if !ok || len(v) != 1 || len(v[0]) < 1 || len(v[0]) > 64 {
		return "", false
	}

	for i := 0; i < len(v[0]); i++ {
		c := v[0][i]
		if !((c >= 'a' && c <= 'z') ||
			(c >= 'A' && c <= 'Z') ||
			(c >= '0' && c <= '9') ||
			c == ' ' || c == '_' || c == '-') {
			return "", false
		}
	}

	return v[0], true
}

func validateFileID(rawPath string) (string, bool) {
	const prefix = "/v1/files/"
	if !strings.HasPrefix(rawPath, prefix) {
		return "", false
	}

	rawID := strings.TrimPrefix(rawPath, prefix)
	if rawID == "" || strings.Contains(rawID, "/") {
		return "", false
	}

	id, err := url.PathUnescape(rawID)
	if err != nil || len(id) < 1 || len(id) > 64 {
		return "", false
	}

	for i := 0; i < len(id); i++ {
		c := id[i]
		if !((c >= 'a' && c <= 'z') ||
			(c >= 'A' && c <= 'Z') ||
			(c >= '0' && c <= '9') ||
			c == '_' || c == '-') {
			return "", false
		}
	}

	return id, true
}

func validHostname(host string) bool {
	if len(host) < 1 || len(host) > 253 {
		return false
	}

	for i := 0; i < len(host); i++ {
		if host[i] >= 0x80 {
			return false
		}
	}

	if strings.HasSuffix(host, ".") {
		return false
	}

	for _, label := range strings.Split(host, ".") {
		if len(label) < 1 || len(label) > 63 ||
			label[0] == '-' || label[len(label)-1] == '-' {
			return false
		}

		for i := 0; i < len(label); i++ {
			c := label[i]
			if !((c >= 'a' && c <= 'z') ||
				(c >= 'A' && c <= 'Z') ||
				(c >= '0' && c <= '9') || c == '-') {
				return false
			}
		}
	}

	return true
}

func validateResolve(contentTypes []string, body []byte) (string, bool) {
	if len(contentTypes) != 1 {
		return "", false
	}

	mediaType, params, err := mime.ParseMediaType(contentTypes[0])
	if err != nil ||
		!strings.EqualFold(mediaType, "application/json") ||
		len(params) > 1 {
		return "", false
	}

	if len(params) == 1 {
		charset, ok := params["charset"]
		if !ok || !strings.EqualFold(charset, "utf-8") {
			return "", false
		}
	}

	if !utf8.Valid(body) {
		return "", false
	}

	dec := json.NewDecoder(bytes.NewReader(body))
	first, err := dec.Token()
	if err != nil || first != json.Delim('{') {
		return "", false
	}

	seenHost := false
	host := ""

	for dec.More() {
		tok, e := dec.Token()
		if e != nil {
			return "", false
		}

		key, ok := tok.(string)
		if !ok || key != "host" || seenHost {
			return "", false
		}
		seenHost = true

		var raw json.RawMessage
		if e := dec.Decode(&raw); e != nil {
			return "", false
		}
		if e := json.Unmarshal(raw, &host); e != nil {
			return "", false
		}
	}

	last, err := dec.Token()
	if err != nil || last != json.Delim('}') || !seenHost {
		return "", false
	}

	var trailing json.RawMessage
	if err := dec.Decode(&trailing); err != io.EOF {
		return "", false
	}

	if !validHostname(host) {
		return "", false
	}

	return strings.ToLower(host), true
}

var hopHeaders = map[string]struct{}{
	"connection":          {},
	"proxy-connection":    {},
	"keep-alive":          {},
	"proxy-authenticate":  {},
	"proxy-authorization": {},
	"te":                  {},
	"trailer":             {},
	"transfer-encoding":   {},
	"upgrade":              {},
}

func connectionNominations(h http.Header) map[string]struct{} {
	out := make(map[string]struct{})

	for _, headerName := range []string{"Connection", "Proxy-Connection"} {
		for _, value := range h.Values(headerName) {
			for _, part := range strings.Split(value, ",") {
				name := strings.ToLower(strings.TrimSpace(part))
				if name != "" {
					out[name] = struct{}{}
				}
			}
		}
	}

	return out
}

func nominatedRequiredHeader(h http.Header) bool {
	nominated := connectionNominations(h)

	for _, name := range []string{"Accept", "Content-Type", "If-None-Match"} {
		if _, ok := nominated[strings.ToLower(name)]; ok {
			return true
		}
	}

	return false
}

func forwardedRequestHeaders(dst, src http.Header) {
	nominated := connectionNominations(src)

	for _, name := range []string{"Accept", "Content-Type", "If-None-Match"} {
		lower := strings.ToLower(name)

		if _, hop := hopHeaders[lower]; hop {
			continue
		}
		if _, nom := nominated[lower]; nom {
			continue
		}

		for _, value := range src.Values(name) {
			dst.Add(name, value)
		}
	}

	// Suppress net/http's implicit User-Agent.
	dst["User-Agent"] = []string{}
}

func forwardedResponseHeaders(dst, src http.Header) {
	nominated := connectionNominations(src)

	for name, values := range src {
		lower := strings.ToLower(name)

		if _, hop := hopHeaders[lower]; hop {
			continue
		}
		if _, nom := nominated[lower]; nom {
			continue
		}

		for _, value := range values {
			dst.Add(name, value)
		}
	}
}

func requestTooLarge(w http.ResponseWriter, r *http.Request) {
	r.Close = true
	w.Header().Set("Connection", "close")
	httpError(w, http.StatusRequestEntityTooLarge)
}

func methodNotAllowed(w http.ResponseWriter, allow string) {
	w.Header().Set("Allow", allow)
	httpError(w, http.StatusMethodNotAllowed)
}

func httpError(w http.ResponseWriter, status int) {
	msg := http.StatusText(status)
	if msg == "" {
		msg = "Request Failed"
	}
	http.Error(w, msg, status)
}

func (g *gateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	meta, haveMeta := rawHeaderMeta(r)
	rawPath, rawQuery, hasQuery := rawPathAndQuery(r.RequestURI)
	kind := classifyRoute(rawPath)

	if g.stopping.Load() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}

	if kind == routeHealth || kind == routeMetrics {
		g.serveSpecial(w, r, kind, meta, haveMeta)
		return
	}

	if !g.logs.reserve() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}

	tw := &trackedWriter{ResponseWriter: w}
	started := time.Now()

	identityRaw, identityOK := validIdentity(r.Header.Values("X-Verified-Client-ID"))
	clientID := "unverified"
	if identityOK {
		clientID = g.logs.pseudonym(identityRaw)
	}

	ev := logEvent{
		RequestID: g.logs.nextRequestID(),
		EventID:   g.logs.nextEventID(),
		Timestamp: started.UTC().Format(time.RFC3339Nano),
		Method:    safeMethod(r.Method),
		Route:     routeTemplate(rawPath),
		ClientID:  clientID,
	}

	reason, outcome := "internal_error", "error"

	defer func() {
		if recover() != nil {
			if !tw.wrote {
				httpError(tw, http.StatusInternalServerError)
			}
			reason, outcome = "internal_error", "error"
		}

		ev.Status = tw.statusCode()
		if !tw.wrote && (r.Context().Err() != nil || g.rootCtx.Err() != nil) {
			ev.Status = 499
		}
		ev.Duration = time.Since(started).Milliseconds()
		ev.Reason = reason
		ev.Outcome = outcome

		g.logs.emit(ev)
	}()

	select {
	case g.active <- struct{}{}:
		defer func() { <-g.active }()
	default:
		reason, outcome = "concurrency_limit", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}

	if g.stopping.Load() {
		reason, outcome = "shutting_down", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}

	if r.ProtoMajor != 1 || r.ProtoMinor != 1 {
		reason, outcome = "unsupported_protocol", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}

	if requestTargetTooLarge(r, meta, haveMeta) ||
		headerLimitExceeded(r, meta, haveMeta) {
		reason, outcome = "request_headers_or_target_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}

	if !identityOK {
		reason, outcome = "untrusted_identity", "rejected"
		httpError(tw, http.StatusUnauthorized)
		return
	}

	if kind == routeUnknown {
		reason, outcome = "not_found", "rejected"
		httpError(tw, http.StatusNotFound)
		return
	}

	var expectedMethod string
	switch kind {
	case routeSearch, routeFiles:
		expectedMethod = http.MethodGet
	case routeResolve:
		expectedMethod = http.MethodPost
	}

	if r.Method != expectedMethod {
		reason, outcome = "method_not_allowed", "rejected"
		methodNotAllowed(tw, expectedMethod)
		return
	}

	if r.ContentLength > maxRequestBody {
		reason, outcome = "body_too_large", "rejected"
		requestTooLarge(tw, r)
		return
	}

	if nominatedRequiredHeader(r.Header) {
		reason, outcome = "conflicting_connection_header", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}

	body, bodyErr := readBodyLimited(tw, r)
	if bodyErr != nil {
		var maxErr *http.MaxBytesError

		if errors.As(bodyErr, &maxErr) {
			reason, outcome = "body_too_large", "rejected"
			requestTooLarge(tw, r)
			return
		}

		if g.rootCtx.Err() != nil {
			reason, outcome = "shutting_down", "error"
			httpError(tw, http.StatusServiceUnavailable)
			return
		}

		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			tw.WriteHeader(499)
			return
		}

		reason, outcome = "invalid_request_body", "rejected"
		httpError(tw, http.StatusBadRequest)
		return
	}

	var (
		upstream      *origin
		upstreamPath  string
		upstreamQuery string
		method        string
		canonicalBody []byte
	)

	switch kind {
	case routeSearch:
		term, ok := validateSearch(rawQuery, hasQuery)
		if !ok {
			reason, outcome = "invalid_search_query", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}

		upstream = g.search
		upstreamPath = "/v1/search"
		method = http.MethodGet
		upstreamQuery = url.Values{"q": []string{term}}.Encode()

	case routeFiles:
		fileID, ok := validateFileID(rawPath)
		if !ok || hasQuery {
			reason, outcome = "invalid_file_id_or_query", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}

		upstream = g.files
		upstreamPath = "/v1/files/" + fileID
		method = http.MethodGet
		upstreamQuery = ""

	case routeResolve:
		if hasQuery {
			reason, outcome = "invalid_resolve_query", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}

		host, ok := validateResolve(r.Header.Values("Content-Type"), body)
		if !ok {
			reason, outcome = "invalid_resolve_json", "rejected"
			httpError(tw, http.StatusBadRequest)
			return
		}

		canonicalBody, _ = json.Marshal(struct {
			Host string `json:"host"`
		}{Host: host})

		upstream = g.resolve
		upstreamPath = "/v1/tools/resolve"
		method = http.MethodPost
		upstreamQuery = ""

	default:
		reason, outcome = "not_found", "rejected"
		httpError(tw, http.StatusNotFound)
		return
	}

	// Reserve the aggregate response-buffer budget before consuming quota
	// or contacting an upstream backend.
	select {
	case g.responseBuffers <- struct{}{}:
		defer func() { <-g.responseBuffers }()
	default:
		reason, outcome = "response_memory_budget", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}

	allowed, retry, full := g.limiter.allow(identityRaw)
	if full {
		reason, outcome = "rate_state_full", "rejected"
		httpError(tw, http.StatusServiceUnavailable)
		return
	}

	if !allowed {
		reason, outcome = "quota_exhausted", "rejected"
		seconds := int(math.Ceil(retry.Seconds()))
		if seconds < 1 {
			seconds = 1
		}
		tw.Header().Set("Retry-After", strconv.Itoa(seconds))
		httpError(tw, http.StatusTooManyRequests)
		return
	}

	var bodyReader io.Reader
	if method == http.MethodPost {
		bodyReader = bytes.NewReader(canonicalBody)
	}

	target := upstream.target(upstreamPath, upstreamQuery)
	upReq, err := http.NewRequestWithContext(
		r.Context(), method, target.String(), bodyReader,
	)
	if err != nil {
		reason, outcome = "upstream_request_failed", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}

	forwardedRequestHeaders(upReq.Header, r.Header)

	if method == http.MethodPost {
		// The validated body is canonical JSON, independently of client headers.
		upReq.Header.Set("Content-Type", "application/json")
	}

	resp, err := g.client.Do(upReq)
	if err != nil {
		if g.rootCtx.Err() != nil {
			reason, outcome = "shutting_down", "error"
			httpError(tw, http.StatusServiceUnavailable)
			return
		}

		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			tw.WriteHeader(499)
			return
		}

		if isTimeout(err) {
			reason, outcome = "upstream_timeout", "error"
			httpError(tw, http.StatusGatewayTimeout)
			return
		}

		reason, outcome = "upstream_failure", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()

	if resp.StatusCode < 200 ||
		resp.StatusCode == http.StatusSwitchingProtocols {
		reason, outcome = "unsupported_upstream_status", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}

	responseBody, readErr, tooLarge := readResponseBounded(resp)
	if tooLarge {
		reason, outcome = "upstream_response_too_large", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}

	if readErr != nil {
		if g.rootCtx.Err() != nil {
			reason, outcome = "shutting_down", "error"
			httpError(tw, http.StatusServiceUnavailable)
			return
		}

		if r.Context().Err() != nil {
			reason, outcome = "client_cancelled", "error"
			tw.WriteHeader(499)
			return
		}

		if isTimeout(readErr) {
			reason, outcome = "upstream_timeout", "error"
			httpError(tw, http.StatusGatewayTimeout)
			return
		}

		reason, outcome = "upstream_failure", "error"
		httpError(tw, http.StatusBadGateway)
		return
	}

	forwardedResponseHeaders(tw.Header(), resp.Header)

	nominatedResponseHeaders := connectionNominations(resp.Header)
	_, contentLengthNominated := nominatedResponseHeaders["content-length"]

	if resp.ContentLength >= 0 &&
		int64(len(responseBody)) == resp.ContentLength &&
		resp.StatusCode != http.StatusNoContent &&
		resp.StatusCode != http.StatusNotModified &&
		!contentLengthNominated {
		tw.Header().Set("Content-Length", strconv.Itoa(len(responseBody)))
	}

	tw.WriteHeader(resp.StatusCode)

	if len(responseBody) > 0 {
		if _, err := tw.Write(responseBody); err != nil {
			reason, outcome = "client_write_failed", "error"
			return
		}
	}

	reason, outcome = "upstream_response", "proxied"
}

func readResponseBounded(resp *http.Response) ([]byte, error, bool) {
	if resp.StatusCode == http.StatusNoContent ||
		resp.StatusCode == http.StatusNotModified {
		return []byte{}, nil, false
	}

	if resp.ContentLength > maxResponseBody {
		return nil, nil, true
	}

	if resp.ContentLength >= 0 {
		n := int(resp.ContentLength)
		buf := make([]byte, n)

		if n == 0 {
			return buf, nil, false
		}

		got, err := io.ReadFull(resp.Body, buf)
		if err != nil {
			return nil, err, false
		}
		if got != n {
			return nil, io.ErrUnexpectedEOF, false
		}

		return buf, nil, false
	}

	// Unknown-length responses use a fixed-size allocation. At most
	// maxResponseBuffers such buffers can exist concurrently.
	buf := make([]byte, maxResponseBody+1)
	n, err := io.ReadFull(resp.Body, buf)

	if n > maxResponseBody {
		return nil, nil, true
	}
	if err == nil {
		return nil, nil, true
	}
	if err == io.EOF || err == io.ErrUnexpectedEOF {
		return buf[:n], nil, false
	}

	return nil, err, false
}

func (g *gateway) serveSpecial(
	w http.ResponseWriter,
	r *http.Request,
	kind routeKind,
	meta headerMeta,
	haveMeta bool,
) {
	select {
	case g.active <- struct{}{}:
		defer func() { <-g.active }()
	default:
		httpError(w, http.StatusServiceUnavailable)
		return
	}

	if !validIdentityOnly(r) {
		httpError(w, http.StatusUnauthorized)
		return
	}

	if !g.logsHealthy() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}

	if g.stopping.Load() {
		httpError(w, http.StatusServiceUnavailable)
		return
	}

	if r.ProtoMajor != 1 || r.ProtoMinor != 1 {
		httpError(w, http.StatusBadRequest)
		return
	}

	if requestTargetTooLarge(r, meta, haveMeta) ||
		headerLimitExceeded(r, meta, haveMeta) {
		requestTooLarge(w, r)
		return
	}

	if r.Method != http.MethodGet {
		methodNotAllowed(w, "GET")
		return
	}

	if r.ContentLength > maxRequestBody {
		requestTooLarge(w, r)
		return
	}

	if _, err := readBodyLimited(w, r); err != nil {
		var maxErr *http.MaxBytesError
		if errors.As(err, &maxErr) {
			requestTooLarge(w, r)
		} else {
			httpError(w, http.StatusBadRequest)
		}
		return
	}

	if kind == routeHealth {
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		w.Header().Set("Cache-Control", "no-store")
		w.WriteHeader(http.StatusOK)
		_, _ = io.WriteString(w, "ok\n")
		return
	}

	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.Header().Set("Cache-Control", "no-store")
	_ = json.NewEncoder(w).Encode(g.logs.metrics())
}

func validIdentityOnly(r *http.Request) bool {
	_, ok := validIdentity(r.Header.Values("X-Verified-Client-ID"))
	return ok
}

func (g *gateway) logsHealthy() bool {
	g.logs.admitMu.Lock()
	defer g.logs.admitMu.Unlock()
	return !g.logs.unhealthy
}

func isTimeout(err error) bool {
	if errors.Is(err, context.DeadlineExceeded) {
		return true
	}

	var ne net.Error
	return errors.As(err, &ne) && ne.Timeout()
}

func newHTTPClient() *http.Client {
	transport := &http.Transport{
		Proxy:                  nil,
		DialContext:            (&net.Dialer{Timeout: 2 * time.Second, KeepAlive: 30 * time.Second}).DialContext,
		ForceAttemptHTTP2:      false,
		MaxIdleConns:           128,
		MaxIdleConnsPerHost:    64,
		MaxConnsPerHost:        maxResponseBuffers,
		MaxResponseHeaderBytes: 16 << 10,
		IdleConnTimeout:        30 * time.Second,
		ResponseHeaderTimeout:  2 * time.Second,
		DisableCompression:     true,
	}

	return &http.Client{
		Transport: transport,
		Timeout:   2 * time.Second,
		CheckRedirect: func(_ *http.Request, _ []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
}

func run() int {
	signal.Ignore(syscall.SIGPIPE)

	replica := os.Getenv("REPLICA_ID")
	if replica != "0" && replica != "1" &&
		replica != "2" && replica != "3" {
		fmt.Fprintln(os.Stderr, "gateway: REPLICA_ID must be 0, 1, 2, or 3")
		return 2
	}

	search, err := parseOrigin(os.Getenv("SEARCH_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid SEARCH_BACKEND_URL")
		return 2
	}

	files, err := parseOrigin(os.Getenv("FILES_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid FILES_BACKEND_URL")
		return 2
	}

	resolve, err := parseOrigin(os.Getenv("RESOLVE_BACKEND_URL"))
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: invalid RESOLVE_BACKEND_URL")
		return 2
	}

	addr := os.Getenv("GATEWAY_ADDR")
	if addr == "" {
		addr = "127.0.0.1:8080"
	}

	logs, err := newLogManager()
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: unable to initialize logging")
		return 2
	}

	rootCtx, cancelWork := context.WithCancel(context.Background())

	g := &gateway{
		search:          search,
		files:           files,
		resolve:         resolve,
		client:          newHTTPClient(),
		logs:            logs,
		limiter:         newRateLimiter(),
		active:          make(chan struct{}, maxConcurrent),
		responseBuffers: make(chan struct{}, maxResponseBuffers),
		rootCtx:         rootCtx,
		cancelWork:      cancelWork,
	}

	server := &http.Server{
		Addr:              addr,
		Handler:           g,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      3 * time.Second,
		IdleTimeout:       30 * time.Second,
		MaxHeaderBytes:    parserHeaderBytes,
		ErrorLog:          log.New(io.Discard, "", 0),
		BaseContext:       func(net.Listener) context.Context { return rootCtx },
		ConnContext: func(ctx context.Context, c net.Conn) context.Context {
			if ac, ok := c.(*accountingConn); ok {
				return context.WithValue(ctx, connContextKey{}, ac)
			}
			return ctx
		},
	}

	listener, err := net.Listen("tcp", addr)
	if err != nil {
		fmt.Fprintln(os.Stderr, "gateway: unable to listen")
		return 2
	}

	serveErr := make(chan error, 1)
	go func() {
		serveErr <- server.Serve(accountingListener{Listener: listener})
	}()

	signals := make(chan os.Signal, 1)
	signal.Notify(signals, syscall.SIGTERM, os.Interrupt)
	defer signal.Stop(signals)

	select {
	case <-signals:
		signalAt := time.Now()

		g.stopping.Store(true)
		server.SetKeepAlivesEnabled(false)

		shutdownCtx, cancel := context.WithDeadline(
			context.Background(),
			signalAt.Add(2500*time.Millisecond),
		)
		_ = server.Shutdown(shutdownCtx)
		cancel()

		// Cancel any work still running at the signal-anchored work deadline.
		g.cancelWork()
		_ = server.Close()

		drainDeadline := signalAt.Add(2850 * time.Millisecond)
		for time.Now().Before(drainDeadline) {
			if logs.drained() {
				os.Exit(0)
			}
			time.Sleep(2 * time.Millisecond)
		}

		// A permanently blocked stdout write cannot be interrupted portably.
		// Events remaining at this deadline may be lost; draining is best effort.
		os.Exit(0)
		return 0

	case err := <-serveErr:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			fmt.Fprintln(os.Stderr, "gateway: server stopped unexpectedly")
			cancelWork()
			return 2
		}

		cancelWork()
		return 0
	}
}

func main() {
	os.Exit(run())
}

``` 

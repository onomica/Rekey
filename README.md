# Rekey

A standalone network service that protects unsafe `POST` operations (creating an order, charging money, sending an email) from running twice on retry — and, unlike a plain middleware, recovers operations that got stuck halfway.

Clients from **any language** talk to it over REST or gRPC. The service stores the state of each operation by `Idempotency-Key` and decides whether to run it, replay the saved response, wait, or reject a conflict.

## Why this exists

On a network failure the client retries. Without protection the backend creates a **second** order or performs a **second** charge.

```
POST /orders + Idempotency-Key: abc-123

1st request               -> operation runs, response is saved
retry with same key       -> the same saved response is returned
retry with different body -> conflict (key reused for something else)
```

Plenty of libraries already do this much. The reason to run a *service* is two things they don't:

1. **Cross-language.** A Go package only imports into Go. A network service is callable from Python, Node, Java, C#, Go — anything that speaks HTTP or gRPC.
2. **Reconciliation.** If the external service created the order but crashed before confirming, the key would otherwise hang in `processing` forever. This service detects that and recovers the real outcome. Most middlewares just ignore this case.

## How it works

One PostgreSQL table with `UNIQUE (tenant, scope, key)` is the single source of truth. Atomicity comes from `INSERT ... ON CONFLICT DO NOTHING`, not application locks.

Three calls, four decisions:

| decision     | when                          | what the caller should do              |
|--------------|-------------------------------|----------------------------------------|
| `new`        | key seen for the first time   | run the operation, then call `Complete`|
| `replay`     | operation already completed   | return the saved response to the client|
| `processing` | operation still running       | return `409` / ask to retry later      |
| `conflict`   | same key, different body       | return `409` / `422`                   |

Key lifecycle:

```
Reserve --new--> processing --Complete--> completed  (replay only from here)
                            \--Fail-----> failed
                            \--timeout--> reconcile_required --> completed / failed / uncertain
```

## Concepts

- **Scope** — the operation type: `orders:create`, `payments:capture`. Separates different business actions.
- **Tenant** — the client/project using the service. Keys of different tenants never collide.
- **Request hash** — a hash of the request body. Detects reuse of a key with a different payload.
- **Reservation token** — a temporary token returned by `Reserve`, required for `Complete` / `Fail`.

## Integration: how other services talk to it

The service is deployed on its own; every language talks to it over the network. This is the only approach that gives real cross-language support. The cost: one network hop per `Reserve`, and you have to operate the service.

There are two transports, exposing the same domain logic:

### REST — simplest to integrate

Any language, no code generation, easy to debug with `curl`. The cost is latency and no static types. Best for external clients, dashboards, and getting started.

```
POST /v1/idempotency/reserve
Authorization: Bearer <api_key>

{ "scope": "orders:create", "key": "abc-123", "request_hash": "sha256:6abf...",
  "ttl_seconds": 86400, "processing_timeout_seconds": 120 }

-> { "decision": "new", "status": "processing", "reservation_token": "resv_123" }
-> { "decision": "replay", "status": "completed",
     "response_status": 201, "response_body": {"order_id": "ord_777"} }
```

```
POST /v1/idempotency/complete   { "reservation_token": "resv_123",
                                  "response_status": 201,
                                  "response_body": {"order_id": "ord_777"} }

POST /v1/idempotency/fail       { "reservation_token": "resv_123",
                                  "error_message": "provider timeout", "retryable": true }
```

### gRPC — for internal, high-traffic callers

From your `.proto` you generate clients for Python, Node, Java, C#, Go, etc. with one command. Faster than REST, strict types. The cost is heavier infrastructure (HTTP/2, proxies) and harder debugging. Worth it for a service called on **every** payment request.

```protobuf
service IdempotencyService {
  rpc Reserve(ReserveRequest) returns (ReserveResponse);
  rpc Complete(CompleteRequest) returns (CompleteResponse);
  rpc Fail(FailRequest) returns (FailResponse);
  rpc GetKey(GetKeyRequest) returns (GetKeyResponse);
}

message ReserveRequest {
  string scope = 1;
  string key = 2;
  string request_hash = 3;
  int64 ttl_seconds = 4;
  int64 processing_timeout_seconds = 5;
}

message ReserveResponse {
  string decision = 1;          // new | replay | processing | conflict | uncertain
  string status = 2;
  string reservation_token = 3;
  int32  response_status = 4;
  string response_body_json = 5;
}
```

**Rule of thumb:** start with REST for cross-language reach and simplicity; add gRPC only when latency on the hot path forces it.

### Caller pseudocode (any language)

```
res = idem.reserve(scope="orders:create",
                   key=header("Idempotency-Key"),
                   request_hash=sha256(body))

if res.decision == "replay":     return res.response          # already done
if res.decision == "processing": return 409                   # in flight
if res.decision == "conflict":   return 422                   # key reused

order = create_order(body)                                    # decision == new

idem.complete(token=res.reservation_token,
              response_status=201,
              response_body={"order_id": order.id})
return 201, {"order_id": order.id}
```

## Table schema

```sql
CREATE TABLE idempotency_keys (
    id             UUID PRIMARY KEY,
    tenant_id      UUID NOT NULL,
    scope          TEXT NOT NULL,
    key            TEXT NOT NULL,
    request_hash   TEXT NOT NULL,
    status         TEXT NOT NULL,          -- processing | completed | failed | reconcile_required | ...
    response_status INT,
    response_body  JSONB,
    error_message  TEXT,
    reservation_token TEXT,
    locked_until   TIMESTAMPTZ,            -- processing timeout
    expires_at     TIMESTAMPTZ NOT NULL,   -- key TTL
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, scope, key)
);
```

## Reconciliation — the core feature

This is the reason to build a service instead of copying a middleware. If the external code created the order but crashed before `Complete`, the key would stay in `processing` forever.

1. A background **scanner** finds `processing` keys whose `locked_until` expired and marks them `reconcile_required`.
2. A **worker** asks the external service "what happened to this key?" and writes the final status.

External contract (signed with HMAC + timestamp):

```
POST /reconcile  { scope, key, request_hash }
->  { "status": "completed", "response_status": 201, "response_body": {...} }
->  { "status": "not_found" }
->  { "status": "unknown" }     -> key becomes "uncertain", retried later
```

## Roadmap

| Phase | What | Result |
|-------|------|--------|
| 1 | REST `reserve` / `complete` / `fail`, `UNIQUE` constraint, conflict by hash | idempotency core, cross-language via REST |
| 2 | Scanner + reconciliation worker, mock external service | recovery of stuck operations (the differentiator) |
| 3 | Multi-tenant: tenants, API keys, scopes | client isolation |
| 4 | gRPC API sharing the same domain logic | internal high-throughput callers |
| 5 | Prometheus metrics (`reserve_total{decision}`, `reconciliation_total`) | observability |

Redis (cache / rate limit) and Kafka (events / audit) are optional deployment plumbing, not correctness. Correctness rests on Postgres.

## Extensions — how Rekey stands apart

Four directions that turn "another idempotency library" into a distinct product. They compose into one story: *a gateway that guarantees a dangerous action — triggered by a human or an AI agent — runs exactly once, and shows you what happened.*

### 1. Proxy mode — kill the `/reconcile` burden

The weakest point of the coordination model is that every protected service must implement `/reconcile`. Proxy mode removes it: instead of *coordinating* the operation, Rekey *executes* it.

```
client -> Rekey (proxy) -> downstream service

1. Client sends POST /orders with Idempotency-Key to Rekey, not to the backend.
2. Rekey reserves the key internally (same table, same UNIQUE constraint).
3. decision=new     -> Rekey forwards the request downstream itself,
                       saves the response, marks the key completed, returns it.
   decision=replay  -> Rekey returns the saved response; downstream never called.
4. If Rekey crashes mid-call, it retries the downstream call itself on restart --
   it made the call, so it owns the outcome. No reconcile endpoint needed.
```

Integration cost drops to zero: point traffic at Rekey (standalone reverse proxy, sidecar, or Envoy/Kong plugin) and everything behind it is protected. The extra network hop disappears in the sidecar/plugin form. The original API mode stays available for callers that need in-process control.

### 2. Side-effect firewall for AI agents

In practice the biggest source of duplicated actions is no longer network retries — it is agents: an LLM repeats a tool call, retries after a timeout, or two parallel agents perform the same action. The core engine already solves this; this extension repackages it for that caller.

```
agent -> MCP tool call "send_email" -> Rekey firewall -> actual tool

1. Rekey derives the key: hash(agent_id, tool, canonicalized args).
2. decision=new     -> execute the tool, record the result.
   decision=replay  -> return the recorded result; the email is NOT sent twice.
3. Policy layer (optional): operations matching a rule (amount > X,
   destructive scope) are held in `pending_approval` until a human confirms.
4. Every call lands in an audit journal: which agent did what, when, with what args.
```

Deliverable: an MCP-compatible gateway + audit log + approval gate. Same table, same state machine — `pending_approval` is just one more status before `processing`.

### 3. Inbound webhook deduplication — the adoption wedge

The mirror image of the same problem: providers (Stripe, GitHub, Telegram) deliver webhooks *at-least-once*, and every team hand-writes the same dedup. Rekey terminates webhooks instead:

```
provider -> Rekey /webhooks/{tenant}/{source} -> your handler

1. Rekey verifies the provider signature.
2. Reserves key = (tenant, source, event_id) -- the same Reserve call.
3. decision=new    -> forward the event to your handler exactly once;
                      retry with backoff until the handler returns 2xx.
   decision=replay -> drop the duplicate silently.
```

Small, well-understood pain; near-zero integration cost. It gets Rekey into the stack, after which the core product is already deployed.

### 4. Observability as a product, not a metrics endpoint

Rekey sits on data no one else has: every prevented duplicate is money not charged twice. Surface it instead of only exporting counters:

```
- prevented duplicates per scope/tenant, with the monetary amount when the
  request body carries one ("prevented 214 double charges / $18,340 this month")
- stuck-operation heatmap: which downstream service leaves keys in
  reconcile_required, how often, how long recovery takes
- per-tenant retry patterns: which client retries pathologically
```

Logic: everything is derived from the existing `idempotency_keys` table plus a small events table written on each state transition — no new infrastructure. A dashboard that states its value in currency justifies the service to the business by itself.

**Non-goal:** sagas / multi-step workflow chains. That is Temporal/Restate territory; entering it sacrifices the simplicity that is Rekey's advantage over them.

## Correctness check

The key test: 100 concurrent `Reserve` calls with the same key must yield exactly **one** `decision=new`; the rest must be `processing` or `replay`.

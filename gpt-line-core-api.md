# GPT-Line Core Platform API Repository Specification
**Suggested GitHub repository name:** `gpt-line-core-api`  
**Service owner:** Backend / business logic developer  
**Primary runtime:** Node.js 22 + TypeScript + PostgreSQL 16 + Redis 7  
**Primary role:** Own all business truth for caller accounts, balance in seconds, call authorization, call finalization, credits, debits, packages, and admin operations.

---

## 1. Mission

Build the production Core Platform API for GPT-Line. This service is the single source of truth for the product’s business state.

The finished service must:

- treat the caller’s phone number as the only account identifier
- auto-create accounts on first call
- store and expose remaining balance in **seconds**
- authorize or deny GPT calls
- prevent more than one active paid GPT call per phone number
- compute live-call cutoff timestamps
- finalize call billing exactly and safely
- accept payment credits from the Payment Service
- expose package data to Telephony
- expose balance phrasing to Telephony
- accept bridge lifecycle events from the Realtime Bridge
- expose admin APIs for the Admin Dashboard
- write an append-only ledger for all balance changes
- guarantee idempotency where needed
- keep its own contracts stable

This repository must be sufficient for a developer to implement the full backend without opening any other repository.

---

## 2. Hard decisions already locked

The following are final:

- Runtime: **Node.js 22**
- Language: **TypeScript**
- Framework: **NestJS**
- Database: **PostgreSQL 16**
- Cache/coordination: **Redis 7**
- Account identifier: `phone_e164` only
- No numeric user ID anywhere
- Materialized current balance is stored on the account row in seconds
- Every balance change must also be written to an append-only ledger
- One active GPT call per `phone_e164` at a time
- Final call debit is the exact whole-second duration rounded up, capped by the caller’s available preflight balance
- Package catalog is authoritative in this service
- Admin dashboard reads and writes through this service
- Internal auth: Bearer internal token
- Timestamps stored in UTC

---

## 3. Canonical phone-number contract

This service never accepts ambiguous phone-number formats from callers. Upstream services are expected to send canonical E.164.

### 3.1 Canonical format
Example:
`+972501234567`

Validation rules:
- starts with `+`
- all remaining characters are digits
- reject anything else with `400 Bad Request`

---

## 4. Repository deliverables

This repository must include:

- NestJS application source
- database migrations
- schema definitions / ORM mappings
- Redis locking utilities
- request validation
- idempotency protections
- internal telephony APIs
- internal bridge-event APIs
- internal payment-credit APIs
- admin APIs
- unit tests
- integration tests against PostgreSQL and Redis
- OpenAPI / Swagger generation for internal endpoints
- Dockerfile
- docker-compose for local dev
- `.env.example`
- README and runbook

Do not leave core endpoints as stubs or pseudocode.

---

## 5. Data model

Implement the following PostgreSQL tables exactly unless a minor technical improvement is required. Column names and semantics must remain the same.

### 5.1 accounts
```sql
CREATE TABLE accounts (
  phone_e164 TEXT PRIMARY KEY,
  status TEXT NOT NULL CHECK (status IN ('active','blocked','fraud_review')),
  remaining_seconds INTEGER NOT NULL DEFAULT 0 CHECK (remaining_seconds >= 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.2 packages
```sql
CREATE TABLE packages (
  package_code TEXT PRIMARY KEY,
  keypad_digit SMALLINT NOT NULL UNIQUE,
  name_he TEXT NOT NULL,
  price_agorot INTEGER NOT NULL CHECK (price_agorot > 0),
  granted_seconds INTEGER NOT NULL CHECK (granted_seconds > 0),
  active BOOLEAN NOT NULL DEFAULT true,
  display_order SMALLINT NOT NULL
);
```

Seed data must create exactly these packages:

| package_code | keypad_digit | name_he       | price_agorot | granted_seconds |
|--------------|--------------|---------------|--------------|-----------------|
| P05          | 1            | חמש דקות      | 3000         | 300             |
| P10          | 2            | עשר דקות      | 5000         | 600             |
| P20          | 3            | עשרים דקות    | 9000         | 1200            |
| P40          | 4            | ארבעים דקות   | 16000        | 2400            |

### 5.3 call_sessions
```sql
CREATE TABLE call_sessions (
  call_session_id TEXT PRIMARY KEY,
  phone_e164 TEXT NOT NULL REFERENCES accounts(phone_e164),
  provider_call_id TEXT NOT NULL,
  asterisk_uniqueid TEXT NOT NULL,
  state TEXT NOT NULL CHECK (state IN ('preflighted','connected','warning_sent','ended')),
  started_at TIMESTAMPTZ NOT NULL,
  connected_at TIMESTAMPTZ,
  ended_at TIMESTAMPTZ,
  absolute_cutoff_at TIMESTAMPTZ NOT NULL,
  warning_at_seconds INTEGER NOT NULL DEFAULT 60,
  ended_reason TEXT CHECK (ended_reason IN (
    'star_exit','caller_hangup','time_expired','system_error',
    'backend_revoke','openai_error','bridge_error'
  )),
  billed_seconds INTEGER CHECK (billed_seconds >= 0),
  preflight_remaining_seconds INTEGER NOT NULL CHECK (preflight_remaining_seconds >= 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.4 balance_ledger
```sql
CREATE TABLE balance_ledger (
  ledger_id BIGSERIAL PRIMARY KEY,
  phone_e164 TEXT NOT NULL REFERENCES accounts(phone_e164),
  entry_type TEXT NOT NULL CHECK (entry_type IN (
    'purchase_credit','call_debit','admin_credit','admin_debit','refund_debit'
  )),
  delta_seconds INTEGER NOT NULL,
  reference_type TEXT NOT NULL,
  reference_id TEXT NOT NULL,
  metadata_json JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.5 purchase_credits
```sql
CREATE TABLE purchase_credits (
  payment_txn_id TEXT PRIMARY KEY,
  phone_e164 TEXT NOT NULL REFERENCES accounts(phone_e164),
  package_code TEXT NOT NULL REFERENCES packages(package_code),
  amount_agorot INTEGER NOT NULL,
  granted_seconds INTEGER NOT NULL,
  provider_name TEXT NOT NULL,
  provider_status TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.6 admin_audit_log
```sql
CREATE TABLE admin_audit_log (
  audit_id BIGSERIAL PRIMARY KEY,
  admin_identity TEXT NOT NULL,
  action_type TEXT NOT NULL,
  target_phone_e164 TEXT,
  before_json JSONB,
  after_json JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 6. Core business rules

These rules are mandatory.

### Rule 1: Account creation
If `phone_e164` does not exist, create an account automatically with:
- `status = active`
- `remaining_seconds = 0`

### Rule 2: Allowed statuses
Only `active` accounts may start AI calls and receive payment credits normally.  
`blocked` accounts:
- cannot start AI calls
- cannot buy more minutes until unblocked

`fraud_review` accounts:
- deny AI calls
- deny new credits unless specifically allowed by future policy

### Rule 3: One active AI call per phone number
There may be at most one active GPT call for a given `phone_e164` at any time.

Use Redis lock:
`active_call:{phone_e164}`

### Rule 4: Preflight balance check
A call is allowed only if:
- account status is `active`
- `remaining_seconds >= 1`
- no active call lock exists

### Rule 5: Cutoff calculation
At preflight:
`absolute_cutoff_at = now() + remaining_seconds seconds`

### Rule 6: Warning threshold
Default `warning_at_seconds = 60`.  
If the caller has less than or equal to 60 seconds at preflight, still allow the call, but do not rely on the warning being meaningful. The bridge may fire it immediately if threshold is met; Telephony should still handle it gracefully.

### Rule 7: Final debit
At call end:
`billed_seconds = ceil(ended_at - connected_at)` in whole seconds

If `connected_at` is null, then `billed_seconds = 0`.

Cap billed seconds by the session’s `preflight_remaining_seconds`.

### Rule 8: Balance update transactionality
Whenever balance changes:
- update `accounts.remaining_seconds`
- insert corresponding `balance_ledger` row

These must happen in the same DB transaction.

### Rule 9: Idempotent payment credit
A payment credit is unique by `payment_txn_id`.  
If the same transaction arrives again, the credit must not be applied twice.

### Rule 10: End-call idempotency
Calling `POST /internal/telephony/calls/end` multiple times for the same `call_session_id` must not double-debit.

---

## 7. Internal APIs for Telephony

These endpoints are authoritative.

### 7.1 Ensure caller exists
**Endpoint**  
`POST /internal/telephony/caller/ensure`

**Request**
```json
{
  "phone_e164": "+972501234567",
  "source": "telephony",
  "provider_call_id": "PJSIP-abc-00001234"
}
```

**Behavior**
- validate `phone_e164`
- create account if missing
- return current status

**Response**
```json
{
  "phone_e164": "+972501234567",
  "status": "active"
}
```

### 7.2 Balance lookup
**Endpoint**  
`GET /internal/telephony/balance/:phone_e164`

**Response**
```json
{
  "phone_e164": "+972501234567",
  "remaining_seconds": 287,
  "speakable_hebrew_text": "נותרו לך 4 דקות ו-47 שניות"
}
```

The service must generate `speakable_hebrew_text` itself so Telephony does not need Hebrew grammar logic.

### 7.3 Call preflight
**Endpoint**  
`POST /internal/telephony/calls/preflight`

**Request**
```json
{
  "phone_e164": "+972501234567",
  "provider_call_id": "PJSIP-abc-00001234",
  "asterisk_uniqueid": "1742111111.152",
  "started_at": "2026-03-16T09:42:11.120Z"
}
```

**Behavior**
1. Validate request.
2. Lock account row `FOR UPDATE`.
3. Ensure account exists or create it.
4. Check account status.
5. Acquire Redis lock `active_call:{phone_e164}`.
6. If balance is zero or account not active, deny and release lock if acquired.
7. Create `call_session_id`.
8. Insert `call_sessions` row with state `preflighted`.
9. Return timing information.

**Allowed response**
```json
{
  "allowed": true,
  "remaining_seconds": 287,
  "warning_at_seconds": 60,
  "absolute_cutoff_epoch_ms": 1773654468120,
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1"
}
```

**Denied response**
```json
{
  "allowed": false,
  "deny_prompt": "no_minutes"
}
```

`deny_prompt` allowed values:
- `no_minutes`
- `system_error`

### 7.4 End call
**Endpoint**  
`POST /internal/telephony/calls/end`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "ended_reason": "star_exit",
  "ended_at": "2026-03-16T09:46:21.011Z"
}
```

**Behavior**
1. Find `call_sessions` row.
2. If already ended, return `ok=true` without further debit.
3. Compute `billed_seconds`.
4. In a single transaction:
   - mark call ended
   - set `ended_reason`
   - set `ended_at`
   - set `billed_seconds`
   - decrement account balance by billed seconds
   - insert `balance_ledger` row of type `call_debit`
5. Release Redis active call lock.

**Response**
```json
{
  "ok": true,
  "billed_seconds": 248,
  "remaining_seconds": 39
}
```

---

## 8. Internal APIs for Bridge events

### 8.1 Bridge connected
**Endpoint**  
`POST /internal/events/bridge-connected`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "connected_at": "2026-03-16T09:42:13.010Z"
}
```

**Behavior**
- set `connected_at`
- set state `connected`

**Response**
```json
{
  "ok": true
}
```

### 8.2 Warning due
**Endpoint**  
`POST /internal/events/bridge-warning-due`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "remaining_seconds": 60
}
```

**Behavior**
- if call not yet ended, set state `warning_sent`
- response should remain idempotent

**Response**
```json
{
  "ok": true
}
```

### 8.3 Cutoff due
**Endpoint**  
`POST /internal/events/bridge-cutoff-due`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567"
}
```

**Behavior**
- mark internally that the call should be considered cutoff-driven if not already ended
- this service does not itself play prompts; telephony handles audio to the caller

**Response**
```json
{
  "ok": true
}
```

### 8.4 Bridge ended
**Endpoint**  
`POST /internal/events/bridge-ended`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "ended_at": "2026-03-16T09:46:21.011Z",
  "reason": "star_exit"
}
```

**Behavior**
- if the call is not ended yet, this event may be stored/logged but the official account debit still occurs through `telephony/calls/end`
- do not double-debit here

**Response**
```json
{
  "ok": true
}
```

---

## 9. Internal APIs for Payment Service

### 9.1 Apply payment credit
**Endpoint**  
`POST /internal/payments/credit`

**Request**
```json
{
  "payment_txn_id": "txn_20260316_123",
  "phone_e164": "+972501234567",
  "package_code": "P10",
  "amount_agorot": 5000,
  "granted_seconds": 600,
  "provider_name": "cardcom",
  "provider_status": "approved"
}
```

**Behavior**
1. Validate package exists and is active.
2. Ensure `amount_agorot` and `granted_seconds` match the catalog for that package.
3. If `payment_txn_id` already exists in `purchase_credits`, return idempotent success.
4. In a single transaction:
   - insert purchase_credits row
   - increment `accounts.remaining_seconds`
   - insert `balance_ledger` row of type `purchase_credit`

**Response**
```json
{
  "ok": true,
  "phone_e164": "+972501234567",
  "remaining_seconds": 887
}
```

### 9.2 Package list for Telephony and Payment
**Endpoint**  
`GET /internal/catalog/packages`

**Response**
```json
{
  "packages": [
    { "package_code": "P05", "keypad_digit": 1, "name_he": "חמש דקות", "price_agorot": 3000, "granted_seconds": 300, "active": true, "display_order": 1 },
    { "package_code": "P10", "keypad_digit": 2, "name_he": "עשר דקות", "price_agorot": 5000, "granted_seconds": 600, "active": true, "display_order": 2 },
    { "package_code": "P20", "keypad_digit": 3, "name_he": "עשרים דקות", "price_agorot": 9000, "granted_seconds": 1200, "active": true, "display_order": 3 },
    { "package_code": "P40", "keypad_digit": 4, "name_he": "ארבעים דקות", "price_agorot": 16000, "granted_seconds": 2400, "active": true, "display_order": 4 }
  ]
}
```

Telephony may consume a narrower payment-oriented view, but this service is the catalog authority.

---

## 10. Admin API requirements

These endpoints are consumed by the Admin Dashboard.

### 10.1 List accounts
`GET /admin/accounts?search=...&status=...&page=...`

Response items must include:
- `phone_e164`
- `status`
- `remaining_seconds`
- `last_call_at`
- `lifetime_purchased_seconds`
- `lifetime_consumed_seconds`

### 10.2 Get account detail
`GET /admin/accounts/:phone_e164`

Must include:
- account summary
- recent call sessions
- recent ledger entries
- recent purchases

### 10.3 Block account
`POST /admin/accounts/:phone_e164/block`

### 10.4 Unblock account
`POST /admin/accounts/:phone_e164/unblock`

### 10.5 Admin credit
`POST /admin/accounts/:phone_e164/credit`

Request:
```json
{
  "seconds": 300,
  "reason": "support adjustment",
  "admin_identity": "ops@example.com"
}
```

### 10.6 Admin debit
`POST /admin/accounts/:phone_e164/debit`

Request:
```json
{
  "seconds": 120,
  "reason": "manual correction",
  "admin_identity": "ops@example.com"
}
```

### 10.7 List calls
`GET /admin/calls?page=...&phone=...`

### 10.8 Get single call
`GET /admin/calls/:call_session_id`

### 10.9 Terminate active call
`POST /admin/calls/:call_session_id/terminate`

This endpoint must mark the session for backend revocation and is expected to be consumed by an operator workflow. The exact telephony-side enforcement can be implemented via future signaling, but the endpoint must exist and log the request.

### 10.10 Audit logging
Every admin mutating endpoint must write to `admin_audit_log`.

---

## 11. Hebrew balance phrasing rules

This service must return `speakable_hebrew_text` for balance inquiries.

Required examples:
- `0` → `לא נותרו לך דקות לשיחה`
- `60` → `נותרה לך דקה אחת`
- `120` → `נותרו לך 2 דקות`
- `287` → `נותרו לך 4 דקות ו-47 שניות`
- `59` → `נותרו לך 59 שניות`

Implement a deterministic Hebrew formatter so Telephony does not need any linguistic logic.

---

## 12. Redis requirements

Use Redis for:
- active call lock: `active_call:{phone_e164}`
- optional short-lived idempotency / coordination helpers

Redis is not business truth. If Redis state and PostgreSQL disagree, PostgreSQL and reconciliation logic win.

### 12.1 Active call lock behavior
- Set with `NX`
- TTL: 6 hours
- release on successful call finalization
- if lock exists during preflight, deny call
- if a stale lock is suspected, validate against `call_sessions`

---

## 13. Security requirements

- validate all request bodies strictly
- mask phone numbers in ordinary logs when feasible
- never log raw bearer tokens
- least-privilege DB credentials
- all mutating admin actions require authenticated upstream identity
- avoid overexposing internals in 500 responses
- keep OpenAPI docs internal only

---

## 14. Configuration and environment variables

Provide `.env.example` including at least:

```env
PORT=3000
NODE_ENV=development

DATABASE_URL=postgres://...
REDIS_URL=redis://...

INTERNAL_SERVICE_TOKEN=replace_me
ADMIN_API_TOKEN=replace_me

LOG_LEVEL=info
```

If using separate tokens per upstream service, include them explicitly and document them.

---

## 15. Required tests

### 15.1 Unit tests
- phone validation
- Hebrew balance formatter
- call preflight allow path
- call preflight deny on zero balance
- call preflight deny on blocked account
- final billed seconds calculation
- cap billed seconds by preflight balance
- idempotent payment credit
- idempotent call end
- admin credit/debit ledger writes

### 15.2 Integration tests
Against real PostgreSQL and Redis containers:
- auto-create account on ensure caller
- successful call preflight acquires lock
- duplicate simultaneous preflight denied
- bridge connected updates state
- warning event idempotent
- payment credit increments balance and ledger
- repeated payment callback does not double-credit
- end call decrements balance and releases lock
- admin block prevents future preflight

---

## 16. Definition of done

This repository is complete only when:

1. The Telephony service can ensure callers, check balances, preflight calls, and end calls successfully.
2. The Payment Service can apply credits exactly once.
3. The Bridge can report lifecycle events without causing double-debits.
4. The Admin Dashboard can list accounts, calls, purchases, and perform adjustments.
5. Balances are always correct in seconds.
6. Ledger history fully explains every balance change.
7. One active call per phone number is enforced.
8. The repo contains all schema, code, tests, and docs needed to run the service end to end.
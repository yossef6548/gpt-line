# GPT-Line Payment Service Repository Specification
**Suggested GitHub repository name:** `gpt-line-payments`  
**Service owner:** Payment / PCI-flow developer  
**Primary runtime:** Node.js 22 + TypeScript  
**Primary role:** Manage package purchase sessions, orchestrate PCI-isolated card-entry during the phone call, execute CardCom settlement flow, and credit caller minutes through the Core API exactly once.

---

## 1. Mission

Build the production payment service for GPT-Line. This repository must completely implement the “press 3 to buy minutes” flow so that a developer can work from this document alone.

The finished service must:

- expose the package list for telephony purchase menus
- create payment sessions keyed by caller phone number and selected package
- orchestrate a PCI-isolated card-entry subflow during the live phone call
- settle the transaction with CardCom
- handle provider callbacks/webhooks or provider result polling as needed
- apply credits to the Core API exactly once on approved transactions
- let Telephony poll purchase result status
- ensure raw card details never enter logs, DBs, caches, or ordinary application memory outside the PCI-isolated capture boundary
- provide admin-facing payment inspection endpoints

This service owns payment orchestration and settlement, but not account balance truth.

---

## 2. Hard decisions already locked

Do not change these decisions.

- Runtime: **Node.js 22**
- Language: **TypeScript**
- Framework: **Fastify**
- Settlement provider: **CardCom**
- Caller must enter credit-card details during the phone call
- Raw PAN/CVV must never enter Asterisk, Core API, Redis, ordinary logs, or persistent DB columns in this service
- Account identifier: `phone_e164`
- Purchase-session identifier: `payment_session_id`
- Provider-settlement uniqueness identifier: `provider_transaction_id`
- Package catalog authority: **Core API**
- This service may cache package catalog data but must validate against Core API
- Credits are applied only after approved provider result
- Credit application must be idempotent

---

## 3. Critical compliance boundary

This repo must be designed around the following non-negotiable rule:

### 3.1 Prohibited outcomes
The implementation must never allow any of the following:
- full card number logged anywhere
- CVV logged anywhere
- raw DTMF card-entry digits stored in ordinary app logs
- raw PAN or CVV stored in PostgreSQL
- raw PAN or CVV written into Redis
- support staff being able to retrieve raw card data later
- card details being routed through the Core API or Telephony service

### 3.2 Required architecture
The service must orchestrate a **PCI-isolated card-entry component**.  
That component may be:
- a vendor-hosted secure IVR capture service,
- a PCI-compliant separate capture microcomponent/network segment,
- or another secure telephony payment capture mechanism,

but in all cases:
- GPT-Line’s standard app stack must receive only a tokenized/payment-result outcome, not raw PAN/CVV.

This repo must still provide the complete orchestration code, state handling, and result processing needed for the product to work end to end.

---

## 4. Product behavior this service must support

Caller flow:

1. Caller presses `3` in the main telephony menu.
2. Telephony asks this service for the package list.
3. Caller selects a package digit.
4. Telephony asks this service to create a payment session.
5. This service returns a `payment_session_id` and a `transfer_target` for the PCI card-entry leg.
6. Telephony transfers the caller there.
7. Caller enters card details during the call.
8. The PCI leg captures the card data securely and drives settlement via CardCom.
9. This service receives the payment result.
10. If approved, it applies the purchased seconds through the Core API.
11. Telephony polls for the outcome and plays the proper result prompt.
12. Caller returns to the main menu.

---

## 5. Package catalog

The package catalog is fixed initially and must be mirrored exactly.

| Digit | package_code | Hebrew name     | Price | price_agorot | granted_seconds |
|------:|--------------|-----------------|------:|-------------:|----------------:|
| 1     | P05          | חמש דקות        | 30 ₪  | 3000         | 300             |
| 2     | P10          | עשר דקות        | 50 ₪  | 5000         | 600             |
| 3     | P20          | עשרים דקות      | 90 ₪  | 9000         | 1200            |
| 4     | P40          | ארבעים דקות     | 160 ₪ | 16000        | 2400            |

The source of truth is the Core API endpoint:
`GET /internal/catalog/packages`

This service must keep its internal view aligned with Core API and must reject inconsistent package data.

---

## 6. Repository deliverables

The finished repository must include:

- application source code
- payment-session state machine
- package-catalog client
- card-entry orchestration logic
- CardCom adapter
- callback/webhook endpoints
- result-polling endpoint for Telephony
- Core API crediting client
- admin endpoints
- tests
- Dockerfile
- docker-compose
- `.env.example`
- README
- deployment runbook
- security/logging redaction rules

Do not leave core paths as TODOs.

---

## 7. Data model

### 7.1 payment_sessions
```sql
CREATE TABLE payment_sessions (
  payment_session_id TEXT PRIMARY KEY,
  phone_e164 TEXT NOT NULL,
  package_code TEXT NOT NULL,
  package_price_agorot INTEGER NOT NULL,
  package_seconds INTEGER NOT NULL,
  status TEXT NOT NULL CHECK (status IN (
    'created','ivr_in_progress','submitted','approved','declined','cancelled','failed','credited'
  )),
  provider_name TEXT NOT NULL DEFAULT 'cardcom',
  provider_transaction_id TEXT,
  provider_result_code TEXT,
  provider_result_message TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 7.2 payment_attempt_events
```sql
CREATE TABLE payment_attempt_events (
  event_id BIGSERIAL PRIMARY KEY,
  payment_session_id TEXT NOT NULL REFERENCES payment_sessions(payment_session_id),
  event_type TEXT NOT NULL,
  payload_json JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 7.3 Optional constraint
Add a unique index on `provider_transaction_id` where not null.

---

## 8. Payment session state machine

### 8.1 States
- `created`
- `ivr_in_progress`
- `submitted`
- `approved`
- `declined`
- `cancelled`
- `failed`
- `credited`

### 8.2 Allowed transitions
- `created -> ivr_in_progress`
- `ivr_in_progress -> submitted`
- `submitted -> approved`
- `submitted -> declined`
- `submitted -> failed`
- `ivr_in_progress -> cancelled`
- `approved -> credited`

Duplicate provider callbacks must not create illegal transitions or duplicate credits.

---

## 9. Internal APIs for Telephony

### 9.1 Package list
**Endpoint**  
`GET /internal/telephony/packages`

**Behavior**
Return the current package list in a telephony-friendly shape.

**Response**
```json
{
  "packages": [
    { "digit": 1, "package_code": "P05", "name_he": "חמש דקות", "price_agorot": 3000 },
    { "digit": 2, "package_code": "P10", "name_he": "עשר דקות", "price_agorot": 5000 },
    { "digit": 3, "package_code": "P20", "name_he": "עשרים דקות", "price_agorot": 9000 },
    { "digit": 4, "package_code": "P40", "name_he": "ארבעים דקות", "price_agorot": 16000 }
  ]
}
```

### 9.2 Start payment session
**Endpoint**  
`POST /internal/telephony/payment/session/start`

**Request**
```json
{
  "phone_e164": "+972501234567",
  "package_code": "P10",
  "provider_call_id": "PJSIP-abc-00001234"
}
```

**Behavior**
1. Validate `phone_e164`.
2. Validate package exists and is active.
3. Create `payment_session_id`.
4. Store session row with status `created`.
5. Move status to `ivr_in_progress`.
6. Return a transfer target representing the secure card-entry leg.

**Response**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "flow_type": "ivr_card_entry",
  "transfer_target": "pci_capture_leg_4021"
}
```

The exact internal meaning of `transfer_target` is implementation-specific, but it must be stable and documented. Telephony will use it as the route to the secure card-entry flow.

### 9.3 Poll payment result
**Endpoint**  
`GET /internal/telephony/payment/session/:payment_session_id`

**Response examples**

Approved and credited:
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "credited",
  "result_prompt": "payment_success"
}
```

Declined:
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "declined",
  "result_prompt": "payment_failed"
}
```

Cancelled:
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "cancelled",
  "result_prompt": "payment_cancelled"
}
```

Unavailable/failure:
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "failed",
  "result_prompt": "payment_unavailable"
}
```

---

## 10. Core API contracts this service must consume

### 10.1 Fetch package catalog
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

### 10.2 Apply approved credit
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

**Response**
```json
{
  "ok": true,
  "phone_e164": "+972501234567",
  "remaining_seconds": 887
}
```

This endpoint is idempotent by `payment_txn_id`.  
This service must rely on that idempotency but must also protect itself from duplicate local processing.

---

## 11. Card-entry orchestration requirements

This repository must define and implement the orchestration of a secure IVR card-entry leg.

### 11.1 Inputs collected during the secure capture leg
The secure card-entry leg must collect:
- card number
- expiry month/year
- CVV
- if acquirer requires it, Israeli ID number

### 11.2 Rules
- Raw digits must not pass through Telephony’s ordinary logs or the Core API.
- Ordinary application logs in this repo must not include raw card fields.
- Any vendor tokens or opaque references returned from the secure capture component may be stored if they do not expose PAN/CVV.

### 11.3 Result outcomes
The card-entry leg must return one of:
- approved with `provider_transaction_id`
- declined with result code/message
- cancelled by caller
- failed technically

### 11.4 Timeout behavior
If the caller does not complete card entry in the secure leg within the configured timeout:
- mark session `cancelled` or `failed` according to the reason
- make the result visible to Telephony via the poll endpoint

---

## 12. CardCom settlement adapter requirements

Implement a dedicated adapter layer for CardCom.

### 12.1 Responsibilities
- submit or finalize a charge based on the secure captured card/token result
- parse approval/decline responses
- verify callbacks where applicable
- normalize provider result into GPT-Line internal statuses

### 12.2 Provider result normalization
Map provider result to internal statuses:
- approved -> `approved`
- issuer/card decline -> `declined`
- user cancel -> `cancelled`
- provider/system/network error -> `failed`

### 12.3 Storage
Store:
- `provider_transaction_id`
- `provider_result_code`
- sanitized `provider_result_message`

Do not store:
- PAN
- CVV
- full expiry if it would violate policy

---

## 13. Callback/webhook endpoint

Expose a public callback endpoint for provider settlement completion if required by the chosen CardCom flow.

### 13.1 Endpoint
`POST /provider/cardcom/callback`

### 13.2 Required behavior
1. Verify authenticity of the callback as supported by the chosen provider flow.
2. Find the `payment_session_id` using provider metadata or a secure correlation token.
3. Upsert the provider result.
4. Transition session state safely.
5. If approved and not yet credited:
   - call Core API credit endpoint
   - on success, set state `credited`
6. Return HTTP 200 quickly once the callback is accepted.

### 13.3 Idempotency
If the same callback is received more than once:
- do not create duplicate credits
- do not create duplicate state transitions that break the state machine

---

## 14. Failure handling

### 14.1 Core API unavailable after approved payment
If the card charge is approved but Core API crediting fails temporarily:
- keep the payment session in `approved`, not `credited`
- retry credit application with backoff
- never re-charge the card
- never credit twice

### 14.2 Provider unavailable before settlement
- mark payment session `failed`
- expose `payment_unavailable` to Telephony

### 14.3 Caller cancels during secure card entry
- mark `cancelled`
- expose `payment_cancelled`

### 14.4 Declined payment
- mark `declined`
- expose `payment_failed`

---

## 15. Admin APIs

These are consumed by the Admin Dashboard.

### 15.1 List payments
`GET /admin/payments?page=...&phone=...&status=...`

Return fields:
- `payment_session_id`
- `phone_e164`
- `package_code`
- `package_price_agorot`
- `package_seconds`
- `status`
- `provider_name`
- `provider_transaction_id`
- `created_at`
- `updated_at`

### 15.2 Get payment detail
`GET /admin/payments/:payment_session_id`

Must include:
- payment session fields
- event history from `payment_attempt_events`

### 15.3 Reconcile payment
`POST /admin/payments/:payment_session_id/reconcile`

Behavior:
- re-check whether an approved-but-not-credited payment should trigger credit retry
- log the admin identity and result

### 15.4 Refund marker
If full refund execution is not implemented in v1, provide an admin action endpoint that marks a payment for refund workflow and records it for operations.

---

## 16. Security and logging rules

Allowed logs:
- `payment_session_id`
- masked phone number
- package code
- high-level state transitions
- provider transaction ID if not considered sensitive
- sanitized provider result code

Forbidden logs:
- PAN
- CVV
- raw secure-entry DTMF
- raw card-entry request payloads
- full Authorization secrets

All request/response logging middleware must support field redaction.

---

## 17. Configuration and environment variables

Provide `.env.example` including at least:

```env
PORT=4000
NODE_ENV=development

DATABASE_URL=postgres://...
CORE_API_BASE_URL=https://core.internal
CORE_API_TOKEN=replace_me

CARDCOM_API_BASE_URL=replace_me
CARDCOM_TERMINAL_ID=replace_me
CARDCOM_USERNAME=replace_me
CARDCOM_PASSWORD=replace_me

LOG_LEVEL=info
```

Also include any secure-capture-leg configuration values needed, explicitly documented.

---

## 18. Required tests

### 18.1 Unit tests
- package validation against catalog
- payment-session state transitions
- result-prompt mapping
- idempotent callback handling
- duplicate provider transaction rejection
- safe redaction of logs

### 18.2 Integration tests
With mocked Core API and mocked CardCom:
- create payment session
- approved callback credits exactly once
- duplicate callback does not double-credit
- declined callback never credits
- cancelled flow returns correct prompt
- Core API temporary failure after approval leaves session retryable

### 18.3 Security tests
Include tests ensuring:
- forbidden card fields are redacted from logs
- webhook handler does not echo sensitive request data back in errors

---

## 19. Definition of done

This repository is complete only when:

1. Telephony can fetch packages and start payment sessions.
2. Caller card entry can be handed off into a secure call-based payment flow.
3. Approved payments are credited exactly once through the Core API.
4. Telephony can poll a stable result prompt afterward.
5. Duplicate provider callbacks do not double-credit.
6. No raw card details are stored or logged anywhere in the ordinary app stack.
7. The repo contains all source, tests, docs, and deployment instructions needed to run the service end to end.
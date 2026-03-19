# GPT-Line Admin Dashboard Repository Specification
**Suggested GitHub repository name:** `gpt-line-admin`  
**Service owner:** Full-stack / internal tools developer  
**Primary runtime:** Next.js 15  
**Primary role:** Provide the internal operations dashboard for support, monitoring, account management, live-call visibility, purchase inspection, reconciliation, and auditability.

---

## 1. Mission

Build the production internal Admin Dashboard for GPT-Line. This repository must be sufficient for a developer to implement the entire admin surface without opening any other repository.

The finished dashboard must:

- authenticate internal operators
- provide a Hebrew RTL interface
- show dashboard summary cards
- show live calls
- show account balances and statuses
- allow blocking and unblocking accounts
- allow manual credit and debit adjustments
- show call history
- show purchase history
- allow payment reconciliation actions
- allow active-call termination requests
- keep a clear operator-friendly audit trail
- consume only the Core API and Payments service
- be ready for production deployment

---

## 2. Hard decisions already locked

Do not change these decisions.

- Framework: **Next.js 15**
- Language: **TypeScript**
- UI language: **Hebrew**
- Layout direction: **RTL**
- Auth method: **Firebase Authentication using Google Sign-In with explicit operator email allowlist**
- Data source for accounts/calls/summary: **Core API**
- Data source for payment views: **Payments Service**
- Browser must never talk directly to PostgreSQL or Redis
- All mutating actions must go through server-side route handlers or server actions
- Every operator action must be auditable

---

## 3. Users of the dashboard

The dashboard is for internal operators, not end users.

Typical tasks:

- inspect caller account by phone number
- see remaining balance
- see account status
- watch live calls
- manually credit or debit seconds
- inspect payment results
- retry payment reconciliation
- terminate an active call
- investigate why a caller could not connect

---

## 4. Required repository deliverables

The repository must include:

- Next.js app
- Firebase authentication setup
- page routing
- server actions or route handlers for mutations
- API clients for Core and Payments
- UI components
- tables and filters
- live data refresh/polling strategy
- tests
- Dockerfile
- `.env.example`
- README
- deployment docs

---

## 5. UX requirements

### 5.1 Language and direction

- all interface text in Hebrew
- full RTL layout
- phone numbers and numeric values must render clearly in mixed RTL/LTR contexts

### 5.2 Design principles

- clean operator-oriented layout
- fast search by phone number
- newest items first where sensible
- accessible contrast and readable tables

### 5.3 Refresh behavior

- live calls page auto-refreshes every 5–10 seconds
- detail pages may use manual refresh and/or polling

---

## 6. Required pages and features

### 6.1 Login / access control

Implement sign-in using Firebase Authentication with Google Sign-In.

Requirements:

- unauthenticated users are redirected to sign-in
- only allowlisted operator emails are authorized
- operator identity email must be available server-side for audit logging
- authentication must work with personal Gmail accounts
- authorization must be enforced server-side

Rules:

- Use Firebase Auth as identity provider
- Browser authenticates with Firebase client SDK
- Browser sends ID token to Next.js server route
- Server verifies token using Firebase Admin SDK
- Server checks normalized lowercase email against `ALLOWED_ADMIN_EMAILS`
- If allowed, server creates application session cookie
- If not allowed, deny access and destroy session
- Mutating actions must derive `admin_identity` from server session, never from client fields

### 6.2 Dashboard home

Show:

- current active call count
- total active accounts
- blocked account count
- recent purchases count
- recent failed purchases count
- quick links

Data source:
- Core `GET /admin/summary`
- optionally Payments for more detailed breakdowns if desired, but the required counts are available from Core

### 6.3 Live calls page

Show a table of active or recently active calls.

Required columns:

- `call_session_id`
- `phone_e164`
- `state`
- `started_at`
- `connected_at`
- `estimated_duration_seconds`
- `estimated_remaining_seconds`
- `ended_reason`
- terminate action

Data source:
- Core `GET /admin/calls`

Terminate action:
- server-side request to `POST /admin/calls/:call_session_id/terminate`
- confirmation modal required

### 6.4 Accounts page

Required columns:

- `phone_e164`
- `status`
- `remaining_seconds`
- `last_call_at`
- `lifetime_purchased_seconds`
- `lifetime_consumed_seconds`

Filters:

- search by phone number
- status
- pagination

Actions:

- details
- block
- unblock
- credit
- debit

### 6.5 Account detail page

Show:

- account summary
- remaining seconds
- status
- recent calls
- recent purchases
- recent ledger items

Actions:

- block/unblock
- credit
- debit

### 6.6 Call history page

Show:

- `call_session_id`
- `phone_e164`
- `started_at`
- `connected_at`
- `ended_at`
- `billed_seconds`
- `ended_reason`
- `state`

Filters:

- phone number
- date range
- ended reason if supported

### 6.7 Purchase history page

Show:

- `payment_session_id`
- `phone_e164`
- `package_code`
- `status`
- `provider_transaction_id`
- `created_at`
- `updated_at`

Filters:

- phone number
- status
- package code
- date range

### 6.8 Payment detail page

Show:

- payment session summary
- package info
- provider result information
- event timeline
- reconcile action

### 6.9 Audit log page

If exposed by backend, show:

- operator identity
- action type
- target phone number
- created time

At minimum, dashboard mutations must always send or derive `admin_identity` server-side so backend audit logging is complete.

---

## 7. Backend API contracts consumed by this repo

### 7.1 Core summary

`GET /admin/summary`

**Example**
```json
{
  "active_call_count": 3,
  "active_account_count": 1250,
  "blocked_account_count": 7,
  "recent_purchase_count_24h": 16,
  "recent_failed_purchase_count_24h": 2
}
```

### 7.2 Core list accounts

`GET /admin/accounts?search=...&status=...&page=...`

### 7.3 Core get account detail

`GET /admin/accounts/:phone_e164`

### 7.4 Core block account

`POST /admin/accounts/:phone_e164/block`

**Request**
```json
{
  "reason": "support block"
}
```

### 7.5 Core unblock account

`POST /admin/accounts/:phone_e164/unblock`

**Request**
```json
{
  "reason": "support unblock"
}
```

### 7.6 Core credit seconds

`POST /admin/accounts/:phone_e164/credit`

**Request**
```json
{
  "seconds": 300,
  "reason": "support adjustment"
}
```

### 7.7 Core debit seconds

`POST /admin/accounts/:phone_e164/debit`

**Request**
```json
{
  "seconds": 120,
  "reason": "manual correction"
}
```

### 7.8 Core list calls

`GET /admin/calls?page=...&phone=...`

Response items must include live-call computed fields for active calls:

- `estimated_duration_seconds`
- `estimated_remaining_seconds`

### 7.9 Core get single call

`GET /admin/calls/:call_session_id`

### 7.10 Core terminate active call

`POST /admin/calls/:call_session_id/terminate`

**Request**
```json
{
  "reason": "operator terminated call"
}
```

The server-side admin route must attach verified `admin_identity` from session context.

### 7.11 Payments list payments

`GET /admin/payments?page=...&phone=...&status=...`

### 7.12 Payments get payment detail

`GET /admin/payments/:payment_session_id`

### 7.13 Payments reconcile payment

`POST /admin/payments/:payment_session_id/reconcile`

**Request**
```json
{
  "reason": "manual reconciliation retry"
}
```

Again, the server-side route must attach verified `admin_identity` from session context.

---

## 8. Client/server architecture requirements

### 8.1 No browser direct secrets

All backend credentials and service tokens remain server-side only.

### 8.2 API access pattern

Preferred pattern:

- Next.js server components for reads where practical
- route handlers or server actions for mutations
- central API client wrappers for Core and Payments

### 8.3 Error handling

UI must present operator-friendly Hebrew error messages without leaking backend internals.

### 8.4 Firebase auth session architecture

Implement the flow as follows:

- User clicks Google sign-in
- Browser authenticates with Firebase Auth
- Browser sends Firebase ID token to a server route
- Server verifies token with Firebase Admin SDK
- Server checks normalized email against `ALLOWED_ADMIN_EMAILS`
- If allowed, server creates application session cookie
- All protected pages validate server session before rendering
- All mutating actions resolve operator identity from verified server session

---

## 9. Required UI components

At minimum implement:

- layout shell
- authenticated page wrapper
- search bar
- filter bar
- paginated table
- status badge
- confirm dialog
- toast/alert messages
- loading state
- empty state
- detail panel / card list

---

## 10. Formatting rules

### 10.1 Seconds display

Wherever balances or durations are shown, display both:

- raw seconds
- human-readable minutes/seconds when practical

Example:
`887 שניות (14:47)`

### 10.2 Dates

Store UTC from backend, display in Israel local time in UI.

### 10.3 Phone numbers

Display full `phone_e164` in admin views because operators need the full number.

---

## 11. Security requirements

- all pages require authenticated session
- operator allowlist check must be server-enforced
- mutating requests require CSRF-safe implementation
- no service tokens exposed to the browser
- admin identity email must be attached to every mutating backend request from the verified server session
- session timeout / revalidation behavior must be documented

---

## 12. Configuration and environment variables

Provide `.env.example` with at least:

```env
NODE_ENV=development
NEXT_PUBLIC_APP_URL=http://localhost:3000

FIREBASE_PROJECT_ID=replace_me
FIREBASE_CLIENT_EMAIL=replace_me
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nreplace_me\n-----END PRIVATE KEY-----\n"

NEXT_PUBLIC_FIREBASE_API_KEY=replace_me
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=replace_me.firebaseapp.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=replace_me
NEXT_PUBLIC_FIREBASE_APP_ID=replace_me

ALLOWED_ADMIN_EMAILS=admin1@gmail.com,admin2@gmail.com

CORE_API_BASE_URL=https://core.internal
CORE_API_TOKEN=replace_me

PAYMENTS_API_BASE_URL=https://payments.internal
PAYMENTS_API_TOKEN=replace_me
```

---

## 13. Required tests

### 13.1 Unit/component tests

- RTL rendering
- status badge rendering
- confirm dialog behavior
- formatting helpers for durations and dates

### 13.2 Integration tests

- dashboard summary fetch/render
- accounts page fetch/render
- account block action sends correct backend payload with server-derived identity
- account credit action sends correct payload
- payments page fetch/render
- reconcile action sends correct payload
- live calls page polling behavior
- terminate call action sends correct backend payload

### 13.3 Auth tests

- unauthenticated redirect
- authenticated but non-allowlisted email blocked
- authorized operator allowed

---

## 14. Definition of done

This repository is complete only when:

1. An operator can authenticate using Firebase Authentication with Google Sign-In
2. The dashboard is fully in Hebrew and RTL
3. Dashboard summary, accounts, calls, and payments can be listed and inspected
4. Operators can block/unblock accounts and adjust balances
5. Operators can inspect payments and trigger reconciliation
6. Operators can request termination of an active call
7. Every mutating action includes operator identity derived from the verified session
8. The repo contains all code, tests, and deployment instructions needed to run it end to end

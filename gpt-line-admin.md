# GPT-Line Admin Dashboard Repository Specification
**Suggested GitHub repository name:** `gpt-line-admin`  
**Service owner:** Full-stack / internal tools developer  
**Primary runtime:** Next.js 15  
**Primary role:** Provide the internal operations dashboard for support, monitoring, account management, live-call visibility, purchase inspection, and auditability.

---

## 1. Mission

Build the production internal Admin Dashboard for GPT-Line. This repository must be sufficient for a developer to implement the entire admin surface without opening any other repository.

The finished dashboard must:

- authenticate internal operators
- provide a Hebrew RTL interface
- show live calls
- show account balances and statuses
- allow blocking and unblocking accounts
- allow manual credit and debit adjustments
- show call history
- show purchase history
- allow payment reconciliation actions
- keep a clear operator-friendly audit trail
- consume only the Core API and Payment Service; no browser direct DB access
- be ready for production deployment

---

## 2. Hard decisions already locked

Do not change these decisions.

- Framework: **Next.js 15**
- Language: **TypeScript**
- UI language: **Hebrew**
- Layout direction: **RTL**
- Auth method: **Firebase Authentication using Google Sign-In with explicit operator email allowlist**
- Data source for accounts/calls/balance: **Core API**
- Data source for payment views: **Payment Service**
- Browser must never talk directly to PostgreSQL or Redis
- All mutating actions must go through server-side route handlers
- Every operator action must be auditable

---

## 3. Users of the dashboard

The dashboard is for internal operators, not end users.

Typical operator tasks:
- inspect a caller account by phone number
- see current remaining balance
- see whether a caller is active/blocked
- watch live calls
- manually credit minutes after support intervention
- manually debit minutes to correct mistakes
- inspect payment results
- retry payment reconciliation
- investigate why a caller could not connect

---

## 4. Required repository deliverables

The finished repository must include:

- Next.js app
- Firebase authentication setup
- page routing
- server actions or route handlers for backend mutations
- API clients for Core API and Payment Service
- UI components
- tables and filters
- live data refresh/polling strategy
- tests
- Dockerfile
- `.env.example`
- README
- deployment docs

Do not leave core pages or actions as stubs.

---

## 5. UX requirements

### 5.1 Language and direction
- all interface text in Hebrew
- full RTL layout
- phone numbers and numeric values should still render clearly in mixed RTL/LTR contexts

### 5.2 Design principles
- clean operator-oriented layout
- fast search by phone number
- newest items first where sensible
- no public branding constraints required beyond clear GPT-Line naming
- accessible contrast and readable tables

### 5.3 Refresh behavior
- live calls page should auto-refresh at a reasonable interval, for example every 5–10 seconds
- detail pages may expose a manual refresh button and/or periodic polling

---

## 6. Required pages and features

### 6.1 Login / access control
Implement a sign-in flow using Firebase Authentication with Google Sign-In.

Requirements:
- unauthenticated users are redirected to sign-in
- only explicitly allowlisted operator email addresses are authorized
- operator identity email must be available server-side for audit logging
- authentication must work with personal Gmail accounts; no Google Workspace or organization account is required
- authorization must be enforced server-side, not only in the client

Implementation rules:
- Use Firebase Authentication as the identity provider.
- Use Google Sign-In through Firebase Auth.
- After successful sign-in, check the authenticated email against a server-side allowlist.
- The allowlist must come from environment configuration, not hardcoded source.
- If the email is not allowlisted, immediately deny access and destroy the session.
- The authenticated operator email must be exposed server-side and attached to every mutating backend request as `admin_identity`.

### 6.2 Dashboard home
A summary page showing:
- current active call count
- total active accounts
- blocked account count
- recent purchases count
- recent failed purchases count
- quick links to key pages

### 6.3 Live calls page
Show a table of active or recently active calls.

Required columns:
- `call_session_id`
- `phone_e164`
- `state`
- `started_at`
- `connected_at`
- `estimated duration`
- `estimated remaining seconds`
- `ended_reason` if ended
- action button: terminate

Data source:
- Core API `GET /admin/calls`
- optionally a dedicated filter for active calls if the backend provides it

Terminate action:
- sends a server-side request to Core API `POST /admin/calls/:call_session_id/terminate`
- confirmation modal required

### 6.4 Accounts page
Show searchable accounts.

Required columns:
- `phone_e164`
- `status`
- `remaining_seconds`
- `last_call_at`
- `lifetime_purchased_seconds`
- `lifetime_consumed_seconds`

Filters:
- search by phone number
- filter by status
- pagination

Actions:
- view details
- block
- unblock
- add minutes/seconds
- remove minutes/seconds

### 6.5 Account detail page
Show:
- account summary
- remaining seconds
- status
- recent calls
- recent purchases
- recent balance ledger items

Actions:
- block/unblock
- credit seconds
- debit seconds

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
- by phone number
- by date range
- by ended reason if supported

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
- reconcile action button

### 6.9 Audit log page
If exposed by backend, show admin actions including:
- operator identity
- action type
- target phone number
- created time

At minimum, dashboard mutations must send `admin_identity` so the backend can write audit logs even if a separate audit page is postponed.

---

## 7. Backend API contracts consumed by this repo

These are the exact contracts this dashboard must target.

## 7.1 Core API: list accounts
`GET /admin/accounts?search=...&status=...&page=...`

**Example response**
```json
{
  "items": [
    {
      "phone_e164": "+972501234567",
      "status": "active",
      "remaining_seconds": 887,
      "last_call_at": "2026-03-16T09:46:21.011Z",
      "lifetime_purchased_seconds": 3600,
      "lifetime_consumed_seconds": 2713
    }
  ],
  "page": 1,
  "page_size": 20,
  "total": 1
}
```

## 7.2 Core API: get account detail
`GET /admin/accounts/:phone_e164`

**Example response**
```json
{
  "account": {
    "phone_e164": "+972501234567",
    "status": "active",
    "remaining_seconds": 887,
    "created_at": "2026-03-01T10:00:00.000Z",
    "updated_at": "2026-03-16T09:46:21.011Z"
  },
  "recent_calls": [],
  "recent_purchases": [],
  "recent_ledger": []
}
```

## 7.3 Core API: block account
`POST /admin/accounts/:phone_e164/block`

**Request**
```json
{
  "admin_identity": "admin@gmail.com",
  "reason": "support block"
}
```

## 7.4 Core API: unblock account
`POST /admin/accounts/:phone_e164/unblock`

**Request**
```json
{
  "admin_identity": "admin@gmail.com",
  "reason": "support unblock"
}
```

## 7.5 Core API: credit seconds
`POST /admin/accounts/:phone_e164/credit`

**Request**
```json
{
  "seconds": 300,
  "reason": "support adjustment",
  "admin_identity": "admin@gmail.com"
}
```

## 7.6 Core API: debit seconds
`POST /admin/accounts/:phone_e164/debit`

**Request**
```json
{
  "seconds": 120,
  "reason": "manual correction",
  "admin_identity": "admin@gmail.com"
}
```

## 7.7 Core API: list calls
`GET /admin/calls?page=...&phone=...`

**Example response**
```json
{
  "items": [
    {
      "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
      "phone_e164": "+972501234567",
      "state": "ended",
      "started_at": "2026-03-16T09:42:11.120Z",
      "connected_at": "2026-03-16T09:42:13.010Z",
      "ended_at": "2026-03-16T09:46:21.011Z",
      "billed_seconds": 248,
      "ended_reason": "star_exit"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total": 1
}
```

## 7.8 Core API: get single call
`GET /admin/calls/:call_session_id`

## 7.9 Core API: terminate active call
`POST /admin/calls/:call_session_id/terminate`

**Request**
```json
{
  "admin_identity": "admin@gmail.com",
  "reason": "operator terminated call"
}
```

## 7.10 Payment Service: list payments
`GET /admin/payments?page=...&phone=...&status=...`

**Example response**
```json
{
  "items": [
    {
      "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
      "phone_e164": "+972501234567",
      "package_code": "P10",
      "package_price_agorot": 5000,
      "package_seconds": 600,
      "status": "credited",
      "provider_name": "cardcom",
      "provider_transaction_id": "cc_123456",
      "created_at": "2026-03-16T09:40:00.000Z",
      "updated_at": "2026-03-16T09:40:09.000Z"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total": 1
}
```

## 7.11 Payment Service: get payment detail
`GET /admin/payments/:payment_session_id`

## 7.12 Payment Service: reconcile payment
`POST /admin/payments/:payment_session_id/reconcile`

**Request**
```json
{
  "admin_identity": "admin@gmail.com",
  "reason": "manual reconciliation retry"
}
```

---

## 8. Client/server architecture requirements

### 8.1 No browser direct secrets
All backend credentials and service tokens must remain server-side only.

### 8.2 API access pattern
Preferred pattern:
- Next.js server components for reads where practical
- route handlers or server actions for mutations
- central API client wrappers for Core API and Payment Service

### 8.3 Error handling
UI must present operator-friendly Hebrew error messages without leaking raw backend internals.

### 8.4 Firebase auth session architecture

Implement the auth/session flow as follows:

- User clicks "Sign in with Google".
- Browser authenticates with Firebase Auth.
- Browser sends the Firebase ID token to a Next.js server route.
- Server verifies the token using Firebase Admin SDK.
- Server checks that the normalized email appears in `ALLOWED_ADMIN_EMAILS`.
- If allowed, server creates the application session cookie.
- All protected pages validate the server-side session before rendering.
- All mutating actions must resolve the operator identity from the verified server session, never from client-submitted form fields.

---

## 9. Required UI components

At minimum implement reusable components for:
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
- human-readable minutes/seconds if practical

Example:
`887 שניות (14:47)`

### 10.2 Dates
Store UTC from backend, display in Israel local time in the UI.

### 10.3 Phone numbers
Display full `phone_e164` in admin views because operators need to work with the actual phone number.

---

## 11. Security requirements

- all pages require authenticated session
- operator email allowlist check must be server-enforced
- mutating requests require CSRF-safe implementation
- no service tokens exposed to the browser
- admin identity email must be attached to every mutating backend request
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

Implementation notes:

- The repo must use Firebase client SDK in the browser for sign-in.
- The repo must use Firebase Admin SDK on the server to verify the session/token.
- `ALLOWED_ADMIN_EMAILS` is a comma-separated list of exact email addresses allowed into the dashboard.
- The application must normalize emails to lowercase before allowlist comparison.
- Do not rely on Firebase project membership, domain ownership, or Google Workspace.

---

## 13. Required tests

### 13.1 Unit/component tests
- RTL layout rendering
- status badge rendering
- confirm dialog behavior
- formatting helpers for durations and dates

### 13.2 Integration tests
- accounts page fetch/render
- account block action sends correct backend payload
- account credit action sends `admin_identity`
- payments page fetch/render
- reconcile action sends correct payload
- live calls page polling behavior

### 13.3 Auth tests
- unauthenticated redirect
- authenticated but non-allowlisted email blocked
- authorized operator allowed

---

## 14. Definition of done

This repository is complete only when:

1. An internal operator can authenticate using Firebase Authentication with Google Sign-In.
2. The dashboard is fully in Hebrew and RTL.
3. Accounts, calls, and payments can be listed and inspected.
4. Operators can block/unblock accounts and adjust balances.
5. Operators can inspect payment details and trigger reconciliation.
6. Every mutating action includes operator identity for audit logging.
7. The repo contains all code, tests, and deployment instructions needed to run the service end to end.

# GPT-Line Telephony Service Repository Specification
**Suggested GitHub repository name:** `gpt-line-telephony`  
**Service owner:** Telephony / IVR developer  
**Primary runtime:** Asterisk 20 LTS on Ubuntu 24.04 LTS  
**Primary role:** Receive inbound Israeli phone calls, run the Hebrew IVR, query caller balance, connect eligible callers to live GPT conversation, execute payment handoff, and enforce real-time bridge commands from the Core API.

---

## 1. Mission

Build the production telephony service for GPT-Line. This repository must be sufficient, by itself, for a developer to fully implement the phone edge of the product without opening any other repository.

The finished service must:

- register to an Israeli SIP trunk/provider
- answer inbound calls
- normalize caller ID into canonical E.164 format
- run the exact Hebrew IVR described below
- query the Core API for caller creation, balance, call preflight, bridge commands, and call finalization
- connect eligible callers to the Realtime Bridge for live GPT conversation
- allow `*` to instantly leave the GPT conversation and return to the main menu
- poll Core for bridge commands during live calls
- play a one-minute warning exactly once when instructed
- stop the conversation exactly when a force-end command is issued
- send callers into the payment flow and return them back cleanly
- never store or log raw card details
- never own balance truth
- never invent business rules already owned by Core

This service is purely the telephony and IVR execution engine.

---

## 2. Hard decisions already locked

These decisions are final and must not be changed by the implementer.

- PBX software: **Asterisk 20 LTS**
- SIP stack: **PJSIP**
- OS: **Ubuntu 24.04 LTS**
- Caller account identifier: **phone number only**
- Canonical phone format everywhere: `phone_e164`, for example `+972501234567`
- DTMF star behavior: `*` always returns the caller to the main menu, except inside the PCI-isolated payment capture leg where that leg’s own rules apply
- Business truth for balances: owned by the Core API, never by Asterisk
- Package catalog for caller purchase menus: consumed from the **Payments Service**
- AI conversation routing: through the **Realtime Bridge Service**
- Payment routing: through the **Payments Service**
- Payment card entry: during the call, but raw PAN/CVV must never enter Asterisk logs or variables outside the secure PCI route
- Audio prompts: **Hebrew prerecorded WAV files**, not TTS, except optional balance phrasing playback if needed
- This service must expose no public internet endpoints except what is strictly required for SIP/media
- Call recording is disabled by default for AI and payment flows

---

## 3. Canonical shared enums used by this service

### 3.1 Call ended reason enum

Allowed values:

- `star_exit`
- `caller_hangup`
- `time_expired`
- `system_error`
- `backend_revoke`
- `openai_error`
- `bridge_error`
- `telephony_disconnect`

### 3.2 Deny prompt enum

Allowed values:

- `no_minutes`
- `system_error`
- `account_blocked`
- `account_under_review`
- `active_call_exists`

### 3.3 Payment result prompt enum

Allowed values:

- `payment_success`
- `payment_failed`
- `payment_cancelled`
- `payment_unavailable`

### 3.4 Bridge command enum

Allowed values returned by Core command polling:

- `play_warning`
- `force_end`

---

## 4. Product behavior this service must implement

### 4.1 Main menu prompt (Hebrew)

The system must answer with a prerecorded human voice prompt:

> אני היועץ האישי שלך  
> לחזרה לתפריט הזה בכל שלב לחץ כוכבית  
> לשיחה איתי הקש 1  
> לבירור יתרת דקות הקש 2  
> לטעינת דקות הקש 3

### 4.2 Option 1: Talk to GPT

Before connecting the caller to GPT, play this prerecorded prompt:

> אם אתה בסביבה רועשת כדי שאדע מתי תורי לדבר תלחץ על השתק כשסיימת לדבר

Then connect the caller to the Realtime Bridge if the Core API authorizes the call.

During the live conversation:

- If the caller presses `*`, immediately terminate the GPT session and return to the main menu
- If Core command polling returns `play_warning`, play the one-minute warning exactly once
- If Core command polling returns `force_end`, immediately terminate the GPT session, play the timeout message, and return to the main menu
- No DTMF may be forwarded to GPT

### 4.3 Option 2: Balance inquiry

The service must ask the Core API for the balance and speak the returned Hebrew phrasing to the user.

### 4.4 Option 3: Buy minutes

The service must:

- fetch the telephony purchase menu from the Payments Service
- play the package menu in Hebrew
- collect a digit
- ask the Payments Service to create a payment session
- transfer the call into the secure PCI-isolated card-entry route returned by Payments
- poll the Payments Service for the final payment result
- play the matching success or failure prompt
- return the caller to the main menu

---

## 5. External dependencies this repository must integrate with

### 5.1 Core API base URL
Example:
`https://core.internal`

### 5.2 Realtime Bridge base URL
Example:
`https://bridge.internal`

### 5.3 Payments base URL
Example:
`https://payments.internal`

### 5.4 Internal auth between services

Every HTTPS request from Asterisk helper scripts / AGI / ARI app to internal services must include:

- `Authorization: Bearer <INTERNAL_SERVICE_TOKEN>`
- `X-Service-Name: telephony`

Tokens come from deployment secrets.

---

## 6. Canonical phone-number rules

All caller IDs must be normalized into `phone_e164` before being sent anywhere.

### 6.1 Allowed format

The only canonical format is:

- starts with `+`
- then digits only
- Israeli caller example: `+972501234567`

### 6.2 Normalization rules

Implement the following exact logic:

1. Strip spaces, dashes, parentheses, and any non-digit except leading plus
2. If number starts with `00`, replace the leading `00` with `+`
3. If number starts with `0` and appears to be an Israeli domestic number such as `0501234567`, convert to `+972501234567`
4. If number starts with `972` and no plus, convert to `+972...`
5. If final result is not `+` followed by digits only, reject it
6. If caller ID is withheld, anonymous, unavailable, empty, or malformed, play the unavailable-caller-id prompt and hang up

### 6.3 Required helper

Implement a reusable helper named conceptually:

`normalize_phone_to_e164(raw_caller_id) -> phone_e164 | error`

---

## 7. Hebrew prompt inventory

The repository must contain final prompt placeholders and loading logic. The exact filenames below are mandatory.

All files:

- format: WAV
- mono
- 8 kHz
- 16-bit PCM
- normalized for telephony playback

Required files:

- `sounds/he/main_menu.wav`
- `sounds/he/ai_intro_noise_hint.wav`
- `sounds/he/no_caller_id.wav`
- `sounds/he/invalid_option.wav`
- `sounds/he/no_input.wav`
- `sounds/he/system_error.wav`
- `sounds/he/no_minutes.wav`
- `sounds/he/account_blocked.wav`
- `sounds/he/account_under_review.wav`
- `sounds/he/active_call_exists.wav`
- `sounds/he/one_minute_left.wav`
- `sounds/he/time_expired.wav`
- `sounds/he/payment_success.wav`
- `sounds/he/payment_failed.wav`
- `sounds/he/payment_cancelled.wav`
- `sounds/he/payment_unavailable.wav`

### 7.1 Exact Hebrew content

**main_menu.wav**
> אני היועץ האישי שלך  
> לחזרה לתפריט הזה בכל שלב לחץ כוכבית  
> לשיחה איתי הקש 1  
> לבירור יתרת דקות הקש 2  
> לטעינת דקות הקש 3

**ai_intro_noise_hint.wav**
> אם אתה בסביבה רועשת כדי שאדע מתי תורי לדבר תלחץ על השתק כשסיימת לדבר

**no_caller_id.wav**
> לא הצלחנו לזהות את מספר הטלפון שלך. לא ניתן להמשיך בשיחה זו.

**invalid_option.wav**
> בחירה שגויה.

**no_input.wav**
> לא התקבלה בחירה.

**system_error.wav**
> אירעה תקלה זמנית. נסה שוב מאוחר יותר.

**no_minutes.wav**
> לא נותרו לך דקות לשיחה. לטעינת דקות הקש 3.

**account_blocked.wav**
> החשבון שלך חסום כרגע. לא ניתן להתחיל שיחה.

**account_under_review.wav**
> החשבון שלך בבדיקת מערכת. לא ניתן להתחיל שיחה כעת.

**active_call_exists.wav**
> כבר מתנהלת שיחה פעילה עבור מספר זה.

**one_minute_left.wav**
> נותרה לך דקה אחת לשיחה.

**time_expired.wav**
> זמן השיחה הסתיים. לחידוש דקות הקש 3.

**payment_success.wav**
> הטעינה בוצעה בהצלחה.

**payment_failed.wav**
> התשלום לא אושר.

**payment_cancelled.wav**
> פעולת התשלום בוטלה.

**payment_unavailable.wav**
> שירות התשלום אינו זמין כעת.

---

## 8. Required repository output

The repository must include:

- Asterisk configuration files
- dialplan files
- PJSIP config
- AGI or ARI helper application code
- Dockerfile for the helper app if one exists
- systemd service definitions if needed
- deployment instructions
- local development instructions
- sample `.env.example` for the helper app
- automated tests for helper logic
- mocked API contract tests
- health-check script
- log redaction rules
- operational runbook

Do not leave core behavior as TODOs.

---

## 9. Recommended internal implementation structure

The telephony repo may use either:

- pure dialplan + AGI helpers, or
- ARI application + thin dialplan

The required architecture is:

- Asterisk handles SIP/PJSIP, channel control, DTMF capture, prompt playback, transfers, and RTP/media plumbing
- A helper application handles HTTP calls to Core / Bridge / Payments and returns simple machine-readable results to Asterisk
- The helper app may be written in Node.js 22 + TypeScript or Python 3.12. Prefer **Node.js 22 + TypeScript**

### 9.1 Suggested directory structure

```text
/
  README.md
  docs/
    operations.md
    deployment.md
    troubleshooting.md
  asterisk/
    pjsip.conf
    extensions.conf
    modules.conf
    rtp.conf
    logger.conf
  helper/
    package.json
    src/
      index.ts
      api/
      phone/
      balance/
      ai/
      payment/
      logging/
    test/
  sounds/
    he/
  scripts/
    healthcheck.sh
    install.sh
    deploy.sh
  systemd/
    gpt-line-telephony-helper.service
```

---

## 10. Call flow state machine

### 10.1 States

- `incoming`
- `caller_identified`
- `main_menu`
- `balance_inquiry`
- `package_menu`
- `payment_flow`
- `payment_return`
- `ai_preflight`
- `ai_connecting`
- `ai_live`
- `ai_return`
- `hangup`

### 10.2 Transitions

#### incoming → caller_identified
Trigger: inbound call answered and caller ID normalized successfully

#### incoming → hangup
Trigger: caller ID missing or invalid

#### caller_identified → main_menu
Trigger: Core caller ensure succeeded

#### main_menu → ai_preflight
Trigger: digit `1`

#### main_menu → balance_inquiry
Trigger: digit `2`

#### main_menu → package_menu
Trigger: digit `3`

#### balance_inquiry → main_menu
Trigger: balance spoken

#### package_menu → payment_flow
Trigger: valid package chosen

#### package_menu → main_menu
Trigger: `*`, repeated invalid input, or timeout

#### payment_flow → payment_return
Trigger: payment session reaches terminal status

#### payment_return → main_menu
Trigger: result prompt finished

#### ai_preflight → ai_connecting
Trigger: Core returns `allowed=true`

#### ai_preflight → main_menu
Trigger: Core returns `allowed=false`

#### ai_connecting → ai_live
Trigger: Bridge started successfully

#### ai_connecting → main_menu
Trigger: bridge start failed

#### ai_live → ai_return
Trigger: `*`, caller hangup, bridge error, Core `force_end`, or local fatal error

#### ai_return → main_menu
Trigger: if caller remains on line after return-worthy exit

#### any non-PCI state → main_menu
Trigger: `*`

---

## 11. Upstream HTTP API contracts

These contracts are authoritative for this repo.

### 11.1 Ensure caller exists

`POST /internal/telephony/caller/ensure`

**Request**
```json
{
  "phone_e164": "+972501234567",
  "source": "telephony",
  "provider_call_id": "PJSIP-abc-00001234"
}
```

**Response**
```json
{
  "phone_e164": "+972501234567",
  "status": "active"
}
```

Failure handling:

- 5xx or timeout: play `system_error.wav`, hang up
- 4xx: treat as system error and hang up

### 11.2 Balance lookup

`GET /internal/telephony/balance/%2B972501234567`

**Response**
```json
{
  "phone_e164": "+972501234567",
  "remaining_seconds": 287,
  "speakable_hebrew_text": "נותרו לך 4 דקות ו-47 שניות"
}
```

### 11.3 AI preflight

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

Allowed `deny_prompt` values:

- `no_minutes`
- `system_error`
- `account_blocked`
- `account_under_review`
- `active_call_exists`

### 11.4 Start bridge

`POST /internal/bridge/start`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "absolute_cutoff_epoch_ms": 1773654468120,
  "warning_at_seconds": 60,
  "asterisk_media": {
    "ip": "10.0.2.11",
    "port": 40012,
    "codec": "pcmu"
  }
}
```

**Success**
```json
{
  "ok": true,
  "bridge_state": "connected",
  "openai_session_ref": "rt_sess_123456"
}
```

**Failure**
```json
{
  "ok": false,
  "error_code": "bridge_unavailable"
}
```

If bridge start fails:

- play `system_error.wav`
- return to main menu

### 11.5 Poll bridge command from Core

`GET /internal/telephony/calls/%3Acall_session_id/command`

Example:
`GET /internal/telephony/calls/call_01JPK9VV71D3Q0N3G2P5R5B8D1/command`

**Response when no command pending**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "pending_command": null
}
```

**Response when warning pending**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "pending_command": {
    "command": "play_warning",
    "reason": "time_threshold",
    "created_at": "2026-03-16T09:45:13.000Z"
  }
}
```

**Response when force-end pending**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "pending_command": {
    "command": "force_end",
    "reason": "time_expired",
    "created_at": "2026-03-16T09:46:13.000Z"
  }
}
```

Rules:

- Telephony must poll every 500 ms to 1000 ms during `ai_live`
- `play_warning` must be executed at most once
- `force_end` must preempt the conversation immediately

### 11.6 Acknowledge bridge command execution

`POST /internal/telephony/calls/command/ack`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "command": "play_warning",
  "executed_at": "2026-03-16T09:45:13.400Z"
}
```

**Response**
```json
{
  "ok": true
}
```

### 11.7 End AI call in Core

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

**Response**
```json
{
  "ok": true,
  "billed_seconds": 248,
  "remaining_seconds": 39
}
```

### 11.8 End bridge

`POST /internal/bridge/end`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "reason": "star_exit"
}
```

**Response**
```json
{
  "ok": true
}
```

### 11.9 Package list from Payments

`GET /internal/telephony/packages`

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

This endpoint is owned by the **Payments Service**, which itself validates against Core.

### 11.10 Start payment session

`POST /internal/telephony/payment/session/start`

**Request**
```json
{
  "phone_e164": "+972501234567",
  "package_code": "P10",
  "provider_call_id": "PJSIP-abc-00001234"
}
```

**Response**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "flow_type": "ivr_card_entry",
  "transfer_target": {
    "type": "asterisk_route",
    "context": "pci_capture",
    "extension": "start",
    "priority": 1
  }
}
```

### 11.11 Poll payment result

`GET /internal/telephony/payment/session/:payment_session_id`

**Response**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "credited",
  "result_prompt": "payment_success"
}
```

---

## 12. Package menu behavior

The telephony service must speak the package menu in Hebrew.

The exact package catalog is fixed initially:

- Digit `1`: **30 ש"ח = 5 דקות**
- Digit `2`: **50 ש"ח = 10 דקות**
- Digit `3`: **90 ש"ח = 20 דקות**
- Digit `4`: **160 ש"ח = 40 דקות**

### 12.1 Spoken prompt requirement

The service must provide a package menu prompt such as:

> לטעינת חמש דקות בשלושים שקלים הקש 1  
> לטעינת עשר דקות בחמישים שקלים הקש 2  
> לטעינת עשרים דקות בתשעים שקלים הקש 3  
> לטעינת ארבעים דקות במאה ושישים שקלים הקש 4

Use prerecorded audio by default.

### 12.2 Input handling

- Accept one digit
- Retry invalid input at most 2 times
- `*` returns to main menu
- timeout after 6 seconds of silence counts as no input
- after 3 failures total, return to main menu

---

## 13. Live GPT call behavior

### 13.1 Media setup

Asterisk must establish media exchange with the Realtime Bridge using private-network RTP.

Codec: `PCMU`

### 13.2 DTMF handling during AI conversation

While in `ai_live`:

- Asterisk must continue to detect DTMF
- if `*` is pressed, Asterisk must:
  1. immediately stop forwarding the caller into the GPT bridge
  2. call `POST /internal/bridge/end` with reason `star_exit`
  3. call `POST /internal/telephony/calls/end`
  4. return the caller to the main menu

No DTMF may be forwarded to GPT.

### 13.3 Warning and timeout path

This behavior is fixed:

- Bridge emits warning/cutoff timing events to Core
- Telephony polls Core for pending commands
- on `play_warning`:
  - play `one_minute_left.wav`
  - acknowledge command execution
- on `force_end`:
  - call `POST /internal/bridge/end` with reason matching the command reason map
  - call `POST /internal/telephony/calls/end` with canonical end reason
  - play `time_expired.wav` if the reason is time expiration
  - return caller to main menu
  - acknowledge command execution

### 13.4 Force-end reason mapping

When `pending_command.command == force_end`, map as follows:

- `reason == time_expired` -> `ended_reason = time_expired`
- `reason == backend_revoke` -> `ended_reason = backend_revoke`
- `reason == system_error` -> `ended_reason = system_error`

### 13.5 Hangup handling

If caller hangs up:

- immediately end local channel resources
- notify Bridge with reason `caller_hangup`
- notify Core with reason `caller_hangup`
- release channel state

---

## 14. Security requirements

### 14.1 Logging

The implementation must never log:

- full card number
- CVV
- expiry date
- raw DTMF from PCI payment capture
- unredacted Authorization tokens

Caller phone numbers in application logs must be masked to last 4 digits when feasible.

### 14.2 Call recording

Call recording must be OFF by default for:

- AI conversations
- package selection
- payment flow
- PCI handoff

### 14.3 SIP debug

Production must run with SIP debug off by default.

---

## 15. Failure behavior

### 15.1 Core unavailable

- play `system_error.wav`
- hang up if the failure occurs before main menu entry
- return to main menu if the failure occurs while already inside a recoverable state and no active AI bridge exists

### 15.2 Bridge unavailable

- play `system_error.wav`
- return to main menu

### 15.3 Payments unavailable

- play `payment_unavailable.wav`
- return to main menu

### 15.4 Invalid caller ID

- play `no_caller_id.wav`
- hang up

### 15.5 Unexpected local error

- if AI bridge is active, attempt bridge end and Core end-call notification with `telephony_disconnect` or `system_error`
- play `system_error.wav`
- hang up cleanly

---

## 16. Required tests

The repository must include automated tests for:

- phone normalization
- auth header injection
- deny prompt to audio mapping
- payment result prompt to audio mapping
- bridge command polling parsing
- bridge command acknowledgment formatting
- package selection parsing
- retry logic
- bridge-end request formatting
- call-end request formatting

Additionally include integration tests with mocked upstream APIs for:

- successful balance inquiry
- denied AI preflight by each deny prompt
- successful AI preflight and bridge connect
- warning command playback path
- force-end command path
- successful payment flow
- failed payment flow
- missing caller ID

---

## 17. Operational metrics

Expose or log metrics for:

- inbound calls count
- call answer count
- main menu entries
- balance inquiries
- AI preflight success rate
- AI connect success rate
- payment handoff count
- payment success/fail counts
- invalid caller ID count
- star exits count
- timeout disconnect count
- backend-revoke disconnect count
- upstream API latency
- bridge command poll latency

---

## 18. Definition of done

This repository is complete only when a fresh server can be provisioned, configured, and tested so that all of the following work end to end:

1. A caller dials the Israeli number
2. The system identifies the phone number
3. The caller hears the exact main menu in Hebrew
4. Pressing `2` speaks the balance from the Core API
5. Pressing `1` plays the noise-hint message and connects to GPT when allowed
6. Pressing `*` during GPT returns to the main menu immediately
7. One-minute warning is triggered through Core command polling and played exactly once
8. Force-end command ends the GPT call and returns to the main menu
9. Pressing `3` enters the package flow, hands off to secure payment, then returns with success/failure
10. No raw card data is stored or logged anywhere in this service
11. The repository includes all configs, helper code, tests, and deployment docs needed to run it

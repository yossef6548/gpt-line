# GPT-Line Telephony Service Repository Specification
**Suggested GitHub repository name:** `gpt-line-telephony`  
**Service owner:** Telephony / IVR developer  
**Primary runtime:** Asterisk 20 LTS on Ubuntu 24.04 LTS  
**Primary role:** Receive inbound Israeli phone calls, run the Hebrew IVR, route callers to GPT conversations, query balance, and hand off callers into the payment flow.

---

## 1. Mission

Build the production telephony service for GPT-Line. This repository must be sufficient, by itself, for a developer to fully implement the telephony side without opening any other repository.

The finished service must:

- register to an Israeli SIP trunk/provider
- answer inbound calls
- normalize caller ID into canonical E.164 format
- run the exact Hebrew IVR described below
- query the Core API for caller creation and balance
- connect eligible callers to the Realtime Bridge for live GPT conversation
- allow `*` to instantly leave the GPT conversation and return to the main menu
- play a one-minute warning when requested
- stop the conversation exactly when time expires
- send callers to the payment flow and return them back cleanly
- never store or log raw card details
- never own balance truth

This service is purely the telephony and IVR execution engine.

---

## 2. Product behavior this service must implement

### 2.1 Main menu prompt (Hebrew)
The system must answer with a prerecorded human voice prompt:

> אני היועץ האישי שלך  
> לחזרה לתפריט הזה בכל שלב לחץ כוכבית  
> לשיחה איתי הקש 1  
> לבירור יתרת דקות הקש 2  
> לטעינת דקות הקש 3

### 2.2 Option 1: Talk to GPT
Before connecting the caller to GPT, play this prerecorded prompt:

> אם אתה בסביבה רועשת כדי שאדע מתי תורי לדבר תלחץ על השתק כשסיימת לדבר

Then connect the caller to the Realtime Bridge if the Core API authorizes the call.

During the live conversation:
- If the caller presses `*`, immediately terminate the GPT session and return to the main menu.
- If there is one minute left, a one-minute warning must be played exactly once.
- If the caller’s balance reaches zero, the GPT conversation must be terminated immediately, a timeout message must be played, and the caller must return to the main menu.

### 2.3 Option 2: Balance inquiry
The service must ask the Core API for the balance and speak the result to the user in Hebrew.

### 2.4 Option 3: Buy minutes
The service must play a menu of packages in Hebrew, collect a digit, start a payment session using the Payment Service, transfer the call into the PCI-isolated card-entry flow, and then return the caller back to the main menu with a success or failure announcement.

---

## 3. Hard decisions already locked

These decisions are final and must not be changed by the implementer.

- PBX software: **Asterisk 20 LTS**
- SIP stack: **PJSIP**
- OS: **Ubuntu 24.04 LTS**
- Caller account identifier: **phone number only**
- Canonical phone format everywhere: `phone_e164`, for example `+972501234567`
- DTMF star behavior: `*` always returns the caller to the main menu
- Business truth for balances: owned by the Core API, never by Asterisk
- Payment card entry: during the call, but raw PAN/CVV must never enter Asterisk logs or variables
- Audio prompts: **Hebrew prerecorded WAV files**, not TTS, except optional balance/package phrasing fragments if needed
- AI conversation routing: through the **Realtime Bridge Service**
- Payment routing: through the **Payment Service**
- This service must expose no public internet endpoints except what is strictly required for SIP/media
- Call recording is disabled by default for all AI and payment flows

---

## 4. External dependencies this repository must integrate with

This repository must integrate with the following services. The exact contract is fully specified here so the developer does not need outside documentation.

### 4.1 Core API base URL
Example:
`https://core.internal`

### 4.2 Realtime Bridge base URL
Example:
`https://bridge.internal`

### 4.3 Payment Service base URL
Example:
`https://payments.internal`

### 4.4 Auth between services
Every HTTPS request from Asterisk helper scripts / AGI / ARI app to internal services must include:

- `Authorization: Bearer <INTERNAL_SERVICE_TOKEN>`
- `X-Service-Name: telephony`

The token will be provided by deployment secrets. The service must treat all upstream APIs as internal trusted services behind private networking.

---

## 5. Canonical phone-number rules

All caller IDs must be normalized into `phone_e164` before being sent anywhere.

### 5.1 Allowed format
The only canonical format is:

- starts with `+`
- then digits only
- Israeli caller example: `+972501234567`

### 5.2 Normalization rules
Implement the following exact logic:

1. Strip spaces, dashes, parentheses, and any non-digit except leading plus.
2. If number starts with `00`, replace the leading `00` with `+`.
3. If number starts with `0` and appears to be an Israeli domestic number such as `0501234567`, convert to `+972501234567`.
4. If number starts with `972` and no plus, convert to `+972...`.
5. If final result is not `+` followed by digits only, reject it.
6. If caller ID is withheld, anonymous, unavailable, empty, or malformed, play the unavailable-caller-id prompt and hang up.

### 5.3 Required function
Implement a reusable helper, callable by dialplan or AGI, named conceptually:
`normalize_phone_to_e164(raw_caller_id) -> phone_e164 | error`

---

## 6. Hebrew prompt inventory

The repo must contain final prompt file placeholders and loading logic. The exact filenames below are mandatory.

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
- `sounds/he/one_minute_left.wav`
- `sounds/he/time_expired.wav`
- `sounds/he/payment_success.wav`
- `sounds/he/payment_failed.wav`
- `sounds/he/payment_cancelled.wav`
- `sounds/he/payment_unavailable.wav`

### 6.1 Exact Hebrew content
Use these exact texts:

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

## 7. Required repository output

The repository must include all of the following:

- Asterisk configuration files
- dialplan files
- PJSIP config
- AGI or ARI helper application code
- Dockerfile for the helper application if one exists
- systemd service definitions if needed
- deployment instructions
- local development instructions
- sample `.env.example` for the helper app
- automated tests for helper logic
- API contract tests using mocked upstream services
- health-check script
- log redaction rules
- operational runbook

Do not leave "TODO" placeholders for core behavior. The repo must be production-ready.

---

## 8. Recommended internal implementation structure

The telephony repo may use either:
- pure dialplan + AGI helpers, or
- ARI application + thin dialplan

The required architecture is:

- Asterisk handles SIP/PJSIP, channel control, DTMF capture, prompt playback, transfers, and RTP/media plumbing.
- A helper application handles HTTP calls to Core API / Bridge / Payment Service and returns simple machine-readable results to Asterisk.
- The helper app may be written in Node.js 22 + TypeScript or Python 3.12. Pick one and fully implement it. Prefer **Node.js 22 + TypeScript** for consistency.

### 8.1 Suggested directory structure
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

## 9. Call flow state machine

The developer must implement the exact state machine below.

### 9.1 States
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

### 9.2 Transitions

#### incoming → caller_identified
Trigger: inbound call answered and caller ID normalized successfully

#### incoming → hangup
Trigger: caller ID missing or invalid

#### caller_identified → main_menu
Trigger: Core API caller ensure succeeded

#### main_menu → ai_preflight
Trigger: digit `1`

#### main_menu → balance_inquiry
Trigger: digit `2`

#### main_menu → package_menu
Trigger: digit `3`

#### main_menu → main_menu
Trigger: `*`, invalid input, or timeout after retry prompt

#### balance_inquiry → main_menu
Trigger: balance spoken

#### package_menu → payment_flow
Trigger: valid package chosen

#### package_menu → main_menu
Trigger: `*`, repeated invalid input, or timeout

#### payment_flow → payment_return
Trigger: Payment Service reports completed / failed / cancelled

#### payment_return → main_menu
Trigger: result prompt finished

#### ai_preflight → ai_connecting
Trigger: Core API returns `allowed=true`

#### ai_preflight → main_menu
Trigger: `allowed=false`

#### ai_connecting → ai_live
Trigger: Realtime Bridge started successfully

#### ai_connecting → main_menu
Trigger: connection failed

#### ai_live → ai_return
Trigger: `*`, hangup, bridge error, or cutoff

#### ai_return → main_menu
Trigger: if caller remains on line after return-worthy exit

#### any → main_menu
Trigger: `*` while not inside PCI card-entry leg

---

## 10. Upstream HTTP API contracts

These contracts are authoritative for this repo.

### 10.1 Ensure caller exists
**Request**
`POST /internal/telephony/caller/ensure`

```json
{
  "phone_e164": "+972501234567",
  "source": "telephony",
  "provider_call_id": "PJSIP-abc-00001234"
}
```

**Success response**
```json
{
  "phone_e164": "+972501234567",
  "status": "active"
}
```

**Failure handling**
- 5xx or timeout: play `system_error.wav`, hang up
- 4xx: treat as system error and hang up

### 10.2 Balance lookup
**Request**
`GET /internal/telephony/balance/%2B972501234567`

**Success response**
```json
{
  "phone_e164": "+972501234567",
  "remaining_seconds": 287,
  "speakable_hebrew_text": "נותרו לך 4 דקות ו-47 שניות"
}
```

The telephony service must speak `speakable_hebrew_text` back to the caller.  
Implementation choice:
- either concatenate prerecorded fragments, or
- use an approved Hebrew TTS engine that runs locally and never transmits user data outside the private environment.

If TTS is used, it must not replace the prerecorded menu prompts.

### 10.3 AI preflight
**Request**
`POST /internal/telephony/calls/preflight`

```json
{
  "phone_e164": "+972501234567",
  "provider_call_id": "PJSIP-abc-00001234",
  "asterisk_uniqueid": "1742111111.152",
  "started_at": "2026-03-16T09:42:11.120Z"
}
```

**Success allowed**
```json
{
  "allowed": true,
  "remaining_seconds": 287,
  "warning_at_seconds": 60,
  "absolute_cutoff_epoch_ms": 1773654468120,
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1"
}
```

**Success denied**
```json
{
  "allowed": false,
  "deny_prompt": "no_minutes"
}
```

If denied, play the matching prompt:
- `no_minutes` → `no_minutes.wav`
- `system_error` → `system_error.wav`

### 10.4 Start bridge
**Request**
`POST /internal/bridge/start`

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

**Success response**
```json
{
  "ok": true,
  "bridge_state": "connected",
  "openai_session_ref": "rt_sess_123456"
}
```

**Failure response**
```json
{
  "ok": false,
  "error_code": "bridge_unavailable"
}
```

If bridge start fails:
- play `system_error.wav`
- return caller to main menu

### 10.5 End AI call
**Request**
`POST /internal/telephony/calls/end`

```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "ended_reason": "star_exit",
  "ended_at": "2026-03-16T09:46:21.011Z"
}
```

**Success response**
```json
{
  "ok": true
}
```

`ended_reason` allowed values:
- `star_exit`
- `caller_hangup`
- `time_expired`
- `system_error`
- `bridge_error`

### 10.6 Package list
**Request**
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

### 10.7 Start payment session
**Request**
`POST /internal/telephony/payment/session/start`

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
  "transfer_target": "pci_capture_leg_4021"
}
```

The service must then transfer the caller into the PCI card-entry leg designated by `transfer_target`.

### 10.8 Poll payment result
**Request**
`GET /internal/telephony/payment/session/pay_01JPKAT7D3W1K6R9F0N0F4Y8S2`

**Response**
```json
{
  "payment_session_id": "pay_01JPKAT7D3W1K6R9F0N0F4Y8S2",
  "status": "credited",
  "result_prompt": "payment_success"
}
```

`result_prompt` values:
- `payment_success`
- `payment_failed`
- `payment_cancelled`
- `payment_unavailable`

---

## 11. Package menu behavior

The telephony service must speak the package menu in Hebrew.

The exact package catalog is fixed:

- Digit `1`: **30 ש"ח = 5 דקות**
- Digit `2`: **50 ש"ח = 10 דקות**
- Digit `3`: **90 ש"ח = 20 דקות**
- Digit `4`: **160 ש"ח = 40 דקות**

### 11.1 Spoken prompt requirement
The service must provide a package menu prompt such as:

> לטעינת חמש דקות בשלושים שקלים הקש 1  
> לטעינת עשר דקות בחמישים שקלים הקש 2  
> לטעינת עשרים דקות בתשעים שקלים הקש 3  
> לטעינת ארבעים דקות במאה ושישים שקלים הקש 4

This may be a prerecorded file or a sequence of prerecorded fragments.  
Use prerecorded audio by default.

### 11.2 Input handling
- Accept one digit
- Retry invalid input at most 2 times
- `*` returns to main menu
- timeout after 6 seconds of silence counts as no input
- after 3 failures total, return to main menu

---

## 12. Live GPT call behavior

### 12.1 Media setup
Asterisk must establish media exchange with the Realtime Bridge.  
Use External Media / RTP on a private network.  
Codec: `PCMU` preferred.  
Asterisk must supply:
- local media IP
- UDP RTP port
- codec

### 12.2 DTMF handling during AI conversation
This is critical and non-negotiable.

While in `ai_live`:
- Asterisk must continue to detect DTMF
- if `*` is pressed, Asterisk must:
  1. immediately stop forwarding the caller into the GPT bridge
  2. call `POST /internal/bridge/end` with reason `star_exit`
  3. call `POST /internal/telephony/calls/end`
  4. return the caller to the main menu

No DTMF may be forwarded to GPT.

### 12.3 Warning and timeout
The bridge or Core API will signal timing through the existing call session contract.

Telephony must support:
- playing `one_minute_left.wav` exactly once when notified
- playing `time_expired.wav` after the call is forcibly ended for zero balance
- returning the caller to the main menu after timeout prompt

### 12.4 Hangup handling
If the caller hangs up:
- immediately end local channel resources
- notify Bridge with reason `caller_hangup`
- notify Core API with reason `caller_hangup`
- release channel state

---

## 13. Required bridge-end API contract

Telephony must call this on any live AI termination it initiates.

**Request**
`POST /internal/bridge/end`

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

Allowed reasons:
- `star_exit`
- `caller_hangup`
- `time_expired`
- `system_error`
- `telephony_disconnect`

---

## 14. Security requirements

### 14.1 Logging
The implementation must never log:
- full card number
- CVV
- expiry date
- raw DTMF from PCI payment capture
- unredacted Authorization tokens

Caller phone numbers in application logs must be masked to last 4 digits when feasible, for example:
`+972******4567`

### 14.2 Call recording
Call recording must be OFF by default for:
- AI conversations
- package selection
- payment flow
- PCI handoff

If the business later enables recordings for support, it must remain impossible on the payment leg.

### 14.3 SIP debug
Production must run with SIP debug off by default.

---

## 15. Failure behavior

Implement these exact user experiences:

### 15.1 Core API unavailable
- play `system_error.wav`
- hang up

### 15.2 Bridge unavailable
- play `system_error.wav`
- return to main menu once
- if the caller retries immediately and the same failure recurs, still return to menu; do not crash the call

### 15.3 Payment service unavailable
- play `payment_unavailable.wav`
- return to main menu

### 15.4 Invalid caller ID
- play `no_caller_id.wav`
- hang up

### 15.5 Unexpected local error
- play `system_error.wav`
- hang up cleanly

---

## 16. Required tests

The repository must include automated tests for:

- phone normalization
- HTTP request signing / auth header injection
- mapping deny prompts to audio files
- mapping payment result prompts to audio files
- package selection parsing
- timeout / invalid-input retry logic
- bridge-end request formatting
- call-end request formatting

Additionally include integration tests with mocked upstream APIs for:
- successful balance inquiry
- denied AI preflight
- successful AI preflight and bridge connect
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
- upstream API latency

---

## 18. Definition of done

This repository is complete only when a fresh server can be provisioned, configured, and tested so that all of the following work end to end:

1. A caller dials the Israeli number.
2. The system identifies the phone number.
3. The caller hears the exact main menu in Hebrew.
4. Pressing `2` speaks the balance from the Core API.
5. Pressing `1` plays the noise-hint message and connects to GPT when allowed.
6. Pressing `*` during GPT returns to the main menu immediately.
7. One-minute warning can be played during the live GPT call.
8. Time expiration ends the GPT call and returns to the main menu.
9. Pressing `3` enters the package flow, hands off to payment, then returns with success/failure.
10. No raw card data is stored or logged anywhere in this service.
11. The repo includes all configs, helper code, tests, and deployment docs needed to run it.
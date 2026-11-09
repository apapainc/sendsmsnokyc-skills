---
name: smsroute-custom-sender-id
description: "Complete guide to Custom Sender IDs (Alphanumeric, Numeric, Shortcode, and Dynamic Macros) on the SendSMSNoKYC / SMSRoute platform (https://api.sendsmsnokyc.com and ssmpp://smpp.sendsmsnokyc.com:2775). Covers zero-KYC per-message Sender ID configuration via REST API (sender_id) and SMPP v3.4 (source_addr, source_addr_ton: 5), 11-character alphanumeric formatting rules, route types (sid_open, sid_fixed, shortcode, sim), live Sender ID preservation testing (/test/run, /test/results), dynamic numeric macros (@NUMBER5@), and carrier rewrite/block recovery. Use whenever configuring, testing, or troubleshooting custom SMS Sender IDs."
metadata:
  author: SMSRoute
  version: 1.0.0
  category: Messaging & Telecommunications
  tags: sender-id, custom-sender-id, alphanumeric-sender-id, smpp, source-addr,ton, sid-open, sendsmsnokyc, smsroute, brand-sms, otp
---

# SMSRoute Custom Sender ID Skill

## Overview

**SendSMSNoKYC / SMSRoute** (`https://sendsmsnokyc.com`) supports dynamic **Custom Sender IDs** (`sender_id` in REST API / `source_addr` in SMPP v3.4) with **zero pre-registration and zero KYC** on Open Sender ID (`sid_open`) routes.

A Custom Sender ID is the originator name or number displayed at the top of the recipient's SMS thread on their handset (e.g., `AuthAlert`, `MyBrand`, `VerifyCode`, or a numeric originator).

- **Set Per-Message (Never in Account Settings):** Custom Sender IDs are passed dynamically on each API request (`POST /sms/send`, `POST /sms/send/bulk`, `POST /campaigns`, `POST /test/run`), SMPP `submit_sm` PDU, or directly inside the **Send SMS** / **New Campaign** composer in the web dashboard. There is no account-wide Sender ID lock or approval queue.
- **Default Fallback:** If `sender_id` is omitted in REST API calls, the platform defaults to `"vortex"`.

---

## Custom Sender ID Formatting Rules & Character Limits

| Property | Alphanumeric Sender ID | Numeric / Shortcode Sender ID |
| :--- | :--- | :--- |
| **Length Limit** | **3 to 11 characters maximum** (ITU-T E.164 / GSM 03.40 standard) | **4–6 digits** (Shortcode) or **5–15 digits** (Long code / SIM) |
| **Allowed Characters** | ASCII letters (`A–Z`, `a–z`) and digits (`0–9`), must include at least one letter | Digits only (`0–9`, optional leading `+` in REST) |
| **Spaces & Symbols** | **Avoid spaces and special characters.** While `^[a-zA-Z0-9 ]{1,11}$` is syntactically accepted at the API edge, many upstream carrier switches strip spaces or reject the PDU (`ESME_RINVSRCADR`). Use PascalCase or compact strings (`MyBrand`, `AuthAlert`, `InfoNotice`). | Digits only (`0–9`), or dynamic macro tokens on SIM routes (`@NUMBER5@`). |
| **Reply Capability** | **One-way (outbound only).** Recipients cannot reply directly to an alphanumeric Sender ID. Include a URL or support number in the message body if a response is needed. | Outbound A2P notification originators. |

### Best Practices for Alphanumeric Sender IDs
1. **Keep It 3–11 Alphanumeric Characters with No Spaces:**
   - Good: `"AuthAlert"`, `"VerifyNow"`, `"CloudOps"`, `"OrderDesk"`
   - Avoid: `"My Company!"` (11 chars with space and `!` — rejected by strict carriers), `"SUPERLONGBRAND"` (14 chars — exceeds 11-char GSM limit), `"123456"` (all digits with alphanumeric TON — may be rejected unless numeric TON is used).
2. **Avoid Protected / High-Risk Financial or Carrier Brand Names:**
   - Upstream MNO firewalls actively filter third-party brand spoofing (major banks, government agencies, or hyperscale social networks). Use your own distinct service name or neutral transactional identifiers (`InfoSMS`, `SecVerify`, `AuthCode`, `AcctNotice`).

---

## Route Types & How They Handle Your Sender ID

Different carrier pathways treat `sender_id` differently depending on local telecom regulations. You can inspect the route `type` for every country via `GET https://api.sendsmsnokyc.com/prices` or in the **Dashboard → Route Switcher**:

| Route Type (`prices.type`) | Name | Sender ID Behavior | Best Use Case |
| :--- | :--- | :--- | :--- |
| **`sid_open`** | **Open Sender ID** | **Preserves your exact custom `sender_id`** (3–11 chars) dynamically per message with zero registration. | Branded alerts, custom sender identity in EU, UK, Asia, Africa, and Middle East. |
| **`sid_fixed`** | **Closed / Fixed Sender ID** | Overwrites custom `sender_id` with a pre-approved, pre-registered alphanumeric identity required by strict national firewalls. | High-deliverability OTP and transactional messages in countries that block unregistered dynamic SIDs. |
| **`shortcode`** | **Shortcode Route** | Delivers from a high-trust 4–6 digit numeric shortcode (e.g., `22395`, `84901`). Custom alphanumeric `sender_id` is replaced by the carrier shortcode. | United States (`US`), Canada (`CA`), and regions where alphanumeric SIDs are prohibited by carrier regulation. |
| **`sim`** | **SIM Route** | Delivers from real mobile SIM numbers (`+...`). Replaces alphanumeric `sender_id` with a numeric mobile originator. | Bypassing strict A2P alphanumeric content filters; maximum handset inbox placement. |

### Route Markers (`Route 1` – `Route 4`)
- **Route 1 (`marker: "1"`):** Standard global pathway optimized for fast OTP and transactional delivery.
- **Route 2 (`marker: "2"`):** Premium direct pathway with strong Alphanumeric Sender ID preservation across Europe, UK, and Asia.
- **Route 3 (`marker: "3"`):** Flexible multi-carrier routing supporting dynamic sender identities and broad geographic reach.
- **Route 4 (`marker: "4"`):** Specialty backup pathway for strict or heavily filtered destinations.

---

## Dynamic Numeric Sender ID Macros (`@NUMBER@`)

On numeric and SIM routes where alphanumeric IDs are not used or where rotating numeric originators improve delivery across large batches, you can embed the `@NUMBER<N>@` macro inside `sender_id`:

- **`@NUMBER5@`** → Generates a random 5-digit numeric string per message (e.g., `48291`).
- **`38@NUMBER3@`** → Generates `38` followed by 3 random digits per message (e.g., `38742`).
- **`447@NUMBER8@`** → Generates a randomized 11-digit numeric originator starting with `447` for every recipient in the batch.

---

## Usage in the REST API (`https://api.sendsmsnokyc.com`)

### 1. Send Single SMS with Custom Sender ID (`POST /sms/send`)

```typescript
const response = await fetch("https://api.sendsmsnokyc.com/sms/send", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.SMSROUTE_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    to: "+447911123456",
    sender_id: "BrandVerify", // 3-11 alphanumeric chars, no spaces
    message: "Your BrandVerify login code is 739204. Expires in 5 minutes.",
    marker: "2",              // Optional: pin Route 2 for alphanumeric preservation
    callback_url: "https://yourapp.com/webhooks/sms-dlr",
  }),
});

const data = await response.json();
// {
//   "messageId": "cm...",
//   "status": "queued",
//   "to": "+447911123456",
//   "from": "BrandVerify",
//   "message": "Your BrandVerify login code is 739204. Expires in 5 minutes.",
//   "timestamp": "2026-11-09T12:00:00.000Z",
//   "blocked": false
// }
```

### 2. Send Bulk SMS with Custom Sender ID (`POST /sms/send/bulk`)

```bash
curl -X POST "https://api.sendsmsnokyc.com/sms/send/bulk" \
  -H "Authorization: Bearer $SMSROUTE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sender_id": "CloudAlerts",
    "marker": "2",
    "recipients": [
      { "to": "+447911123456", "message": "Node eu-west-1 CPU usage exceeded 90%." },
      { "to": "+4915123456789", "message": "Node eu-central-1 CPU usage exceeded 90%." }
    ]
  }'
```

### 3. Create Scheduled Campaign with Custom Sender ID (`POST /campaigns`)

```bash
curl -X POST "https://api.sendsmsnokyc.com/campaigns" \
  -H "Authorization: Bearer $SMSROUTE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Q4 Security Notice",
    "sender_id": "SecNotice",
    "message": "Your security settings were updated. Visit example.com/security for details.",
    "recipients": ["+447911123456", "+33612345678"],
    "scheduled_at": "2026-11-10T09:00:00Z"
  }'
```

---

## Usage in SMPP v3.4 over TLS (`ssmpp://smpp.sendsmsnokyc.com:2775`)

When submitting messages over SMPP (`submit_sm`), you MUST set `source_addr_ton` (Type of Number) and `source_addr_npi` (Numbering Plan Indicator) accurately to match your `source_addr` format:

| Sender ID Format | `source_addr` Example | `source_addr_ton` | `source_addr_npi` | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **Alphanumeric (3–11 chars)** | `'BrandVerify'` | **`5`** (`TON_ALPHANUMERIC`) | **`0`** (`NPI_UNKNOWN`) | Standard for custom brand Sender IDs |
| **International Numeric** | `'447911123456'` | **`1`** (`TON_INTERNATIONAL`) | **`1`** (`NPI_E164`) | Strip leading `+` in SMPP `source_addr` |
| **Shortcode (4–6 digits)** | `'22395'` | **`3`** (`TON_NATIONAL`) or **`0`** | **`0`** (`NPI_UNKNOWN`) | Used when sending from numeric shortcode |

### Node.js (`smpp`) Custom Alphanumeric Sender ID Example

```javascript
session.submit_sm(
  {
    source_addr: "BrandVerify", // Max 11 alphanumeric chars
    source_addr_ton: 5,         // 5 = Alphanumeric TON
    source_addr_npi: 0,         // 0 = Unknown NPI
    destination_addr: "447911123456",
    dest_addr_ton: 1,           // 1 = International TON
    dest_addr_npi: 1,           // 1 = E.164 ISDN NPI
    short_message: "Your verification code is 482910",
    registered_delivery: 1,     // Request DLR
  },
  (pdu) => {
    if (pdu.command_status === 0) {
      console.log("Submitted with Custom SID! Message ID:", pdu.message_id);
    } else {
      console.error("SMPP submit_sm failed with status:", pdu.command_status);
    }
  }
);
```

### Python (`smpplib`) Custom Alphanumeric Sender ID Example

```python
from smpplib import consts

pdu = client.send_message(
    source_addr_ton=consts.SMPP_TON_ALNUM,       # 5 (Alphanumeric)
    source_addr_npi=consts.SMPP_NPI_UNK,         # 0 (Unknown)
    source_addr="BrandVerify",                   # Max 11 chars
    dest_addr_ton=consts.SMPP_TON_INTL,          # 1 (International)
    dest_addr_npi=consts.SMPP_NPI_ISDN,          # 1 (E.164)
    destination_addr="447911123456",
    short_message=b"Your verification code is 482910",
     data_coding=0,
    registered_delivery=True,
)
```

---

## Verifying Custom Sender ID Preservation Before Bulk Sending

Even on `sid_open` routes, individual destination mobile operators (MCC/MNCs) may rewrite or block certain Sender ID strings depending on local firewall rules. **Always run a live route probe test (`/test/run`) with your exact `sender_id` and `message` before launching a bulk campaign:**

```typescript
const API_BASE = "https://api.sendsmsnokyc.com";
const headers = {
  Authorization: `Bearer ${process.env.SMSROUTE_API_KEY}`,
  "Content-Type": "application/json",
};

// 1. Trigger a live probe across Route 1, Route 2, Route 3, Route 4 with your custom sender_id
const runRes = await fetch(`${API_BASE}/test/run`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    country_code: "GB",
    sender_id: "BrandVerify",
    message: "Your BrandVerify security code is 839201",
    selected_route_markers: ["1", "2", "3", "4"],
  }),
});
const { data: { batch_id } } = await runRes.json();

// 2. Poll /test/results?batch_id=... after completion (usually 15-45 seconds)
const resultsRes = await fetch(`${API_BASE}/test/results?batch_id=${encodeURIComponent(batch_id)}`, {
  headers,
});
const results = await resultsRes.json();
// Inspect results.data to see which route delivered fastest and preserved "BrandVerify" intact,
// then lock in the winning route with POST /test/apply-route { "batch_id": batch_id }
```

---

## Troubleshooting & Operational Recovery for Custom Sender IDs

| Symptom | Root Cause | Resolution |
| :--- | :--- | :--- |
| **Sender ID changed to a random number or generic name on handset** | The active route for that destination is `sim`, `shortcode`, or `sid_fixed`, or the local carrier rewrote the dynamic SID in transit. | Check `GET /prices` for a `sid_open` route in that country, run `POST /test/run` across `["1", "2", "3", "4"]`, and switch to `Route 2` or the route that preserves your Sender ID. |
| **Messages rejected or `UNDELIV` when using a specific `sender_id`** | Upstream carrier firewall blocked the specific Sender ID string (e.g., protected brand name, >11 characters, or spaces/symbols) or the SID was burned after high volume. | 1. Ensure `sender_id` is 3–11 alphanumeric characters with no spaces.<br>2. Rotate to a fresh, neutral Sender ID variant (e.g., `InfoSMS`, `AuthAlert`, `VerifyNotice`).<br>3. Re-test with `POST /test/run`. |
| **SMPP returns `ESME_RINVSRCADR` (`0x0000000A`)** | `source_addr` exceeds 11 characters for `source_addr_ton: 5`, contains invalid symbols, or `source_addr_ton` does not match the address type. | Truncate `source_addr` to `<= 11` ASCII alphanumeric characters and set `source_addr_ton: 5`, `source_addr_npi: 0`. |
| **High-volume campaign delivery degrades mid-send ("SID Burn")** | Carrier spam filters flag a single static `sender_id` + identical text template after sustained volume. | Keep 2–3 compact Sender ID variants ready (`AlertDesk`, `NoticeHub`, `UpdateMsg`) and rotate `sender_id` across batches, or use `@NUMBER5@` on numeric routes. |

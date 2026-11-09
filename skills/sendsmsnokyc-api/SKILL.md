---
name: sendsmsnokyc-api
description: "Complete integration guide and reference for the SendSMSNoKYC / SMSRoute API (https://api.sendsmsnokyc.com) and SMPP v3.4 over TLS gateway (ssmpp://smpp.sendsmsnokyc.com:2775). Covers outbound single and bulk A2P/OTP/transactional SMS dispatch, message cost & encoding validation (GSM-7 vs UCS-2), real-time delivery status (DLR) polling and webhooks, HLR number lookup (single & bulk), live multi-carrier route probe testing (/test/run, /test/results, /test/apply-route), campaign management, account balance, and country pricing. Use this skill whenever a user wants to send SMS without KYC, integrate the sendsmsnokyc.com or SMSRoute REST API or SMPP gateway, validate phone numbers via HLR, or run live carrier route quality tests."
metadata:
  author: SMSRoute
  version: 1.0.0
  category: Messaging & Telecommunications
  tags: sms, otp, 2fa, a2p, sendsmsnokyc, smsroute, no-kyc, crypto, smpp, hlr, bulk-sms, delivery-receipts, webhooks
---

# SendSMSNoKYC / SMSRoute API Skill

## Overview

**SendSMSNoKYC (SMSRoute)** is a high-throughput, privacy-first Application-to-Person (A2P) outbound SMS & OTP gateway with zero-KYC onboarding and instant cryptocurrency settlement (BTC, USDT, XMR, ETH, SOL, LTC, USDC, TRX).

- **Web Platform & Dashboard:** `https://sendsmsnokyc.com`
- **Production REST API Base URL:** `https://api.sendsmsnokyc.com`
- **Production SMPP v3.4 TLS Endpoint:** `ssmpp://smpp.sendsmsnokyc.com:2775`
- **API Documentation:** `https://sendsmsnokyc.com/api-docs`

---

## Authentication

All protected REST endpoints require a Bearer token in the `Authorization` header:

```http
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json
```

### Generate or Rotate an API Key
Users can generate an API key from **Account Settings** in the dashboard (`https://sendsmsnokyc.com/settings`) or programmatically with an authenticated session token:

- `GET https://api.sendsmsnokyc.com/user/generate/api_keys`
  - Returns `{ "data": { "user": { ... }, "token": "<JWT_API_KEY>" }, "ok": true }`

> **Security Rule:** Always read the API key from an environment variable such as `process.env.SENDSMSNOKYC_API_KEY` or `process.env.SMSROUTE_API_KEY`. Never hardcode credentials in source files.

---

## Core REST API Endpoints

### 1. Validate SMS Encoding, Parts & Cost Before Sending
- **Endpoint:** `POST /sms/validate`
- **Purpose:** Pre-flight check that calculates character encoding (`GSM_7BIT` vs `UCS2`), segment count (max 6 parts), per-recipient and total cost in EUR, whether the user balance covers the send (`canAfford`), and whether an active route exists (`routeAvailable`).

```bash
curl -X POST "https://api.sendsmsnokyc.com/sms/validate" \
  -H "Authorization: Bearer $SMSROUTE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Your verification code is 849201",
    "recipients": ["+14155552671", "+447911123456"]
  }'
```

**Response (`200 OK`):**
```json
{
  "validation": {
    "isValid": true,
    "parts": 1,
    "encoding": "GSM_7BIT",
    "characterCount": 31,
    "maxLength": 160,
    "warnings": []
  },
  "cost": {
    "total": 0.14,
    "perRecipient": 0.07,
    "currency": "EUR"
  },
  "recipients": {
    "count": 2,
    "totalParts": 2
  },
  "canAfford": true,
  "routeAvailable": true
}
```

---

### 2. Send Single SMS
- **Endpoint:** `POST /sms/send`
- **Rate Limit:** 1,000 requests/minute
- **Body Parameters:**
  | Field | Type | Required | Description |
  | :--- | :--- | :--- | :--- |
  | `to` | `string` | **Yes** | Recipient number in E.164 format (`^\+?[0-9]{7,15}$`, e.g. `"+14155552671"`). |
  | `message` | `string` | **Yes** | SMS text content (1–1600 characters, max 6 concatenated parts). |
  | `sender_id` | `string` | No | Alphanumeric sender ID (up to 11 chars, default `"vortex"`) or numeric originator. |
  | `callback_url` | `string` | No | Public HTTPS/HTTP webhook URL to receive real-time delivery status callbacks. |
  | `marker` | `string` | No | Optional route marker (`"1"`, `"2"`, `"3"`, or `"4"`) to pin a specific carrier route (`Route 1`–`Route 4`). |

```typescript
const response = await fetch("https://api.sendsmsnokyc.com/sms/send", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.SMSROUTE_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    to: "+14155552671",
    sender_id: "AuthAlert",
    message: "Your security code is 849201. Valid for 10 minutes.",
    callback_url: "https://yourapp.com/webhooks/sms-dlr",
  }),
});

const result = await response.json();
// {
//   "messageId": "cm...",
//   "status": "queued",
//   "to": "+14155552671",
//   "from": "AuthAlert",
//   "message": "Your security code is 849201. Valid for 10 minutes.",
//   "timestamp": "2026-11-08T12:00:00.000Z",
//   "blocked": false
// }
```

---

### 3. Send Bulk SMS
- **Endpoint:** `POST /sms/send/bulk`
- **Rate Limit:** 100 requests/minute
- **Body Parameters:**
  - `recipients` (**required**): Array of `{ "to": "+14155552671", "message": "Personalized message" }` objects.
  - `sender_id` (optional): Alphanumeric sender ID (default `"vortex"`).
  - `callback_url` (optional): Webhook URL for delivery receipts.
  - `marker` (optional): Route marker (`"1"`, `"2"`, `"3"`, `"4"`).

```bash
curl -X POST "https://api.sendsmsnokyc.com/sms/send/bulk" \
  -H "Authorization: Bearer $SMSROUTE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sender_id": "vortex",
    "recipients": [
      { "to": "+14155552671", "message": "Hello Alice, your order #101 has shipped!" },
      { "to": "+447911123456", "message": "Hello Bob, your order #102 has shipped!" }
    ],
    "callback_url": "https://yourapp.com/webhooks/sms-dlr"
  }'
```

**Response (`200 OK`):**
```json
{
  "batchId": "batch_a1b2c3d4e5f6g",
  "status": "processing",
  "totalMessages": 2,
  "blockedCount": 0,
  "blockedList": [],
  "timestamp": "2026-11-08T12:00:00.000Z"
}
```

- **Inspect Bulk Batch Status:** `GET /sms/bulk/{batchId}`
- **List Bulk Reports:** `GET /sms/bulk/reports?page=1&limit=20`
- **Export Bulk Batch CSV/JSON:** `GET /sms_bulk/{batchId}/export?format=csv`

---

### 4. Get Single Message Delivery Status (DLR)
- **Endpoint:** `GET /sms/single/{messageId}/status`
- **Response (`200 OK`):**
```json
{
  "data": {
    "messageId": "cm...",
    "to": "+14155552671",
    "status": "delivered",
    "sentAt": "2026-11-08T12:00:00.000Z",
    "deliveredAt": "2026-11-08T12:00:02.000Z",
    "errorCode": null,
    "errorMessage": null
  },
  "ok": true
}
```
- **Query Paginated SMS Logs:** `GET /sms/logs?page=1&limit=100&status=DELIVERED&to=+14155552671`

---

### 5. Account Balance & Pricing
- **Get Account Balance:** `GET /account/balance` (or `GET /user/account/balance`)
  ```json
  {
    "data": {
      "balance": 125.50,
      "currency": "EUR"
    },
    "ok": true
  }
  ```
- **Get Country & Route Pricing:** `GET /prices`
  ```json
  {
    "data": [
      {
        "id": "...",
        "country_code": "US",
        "country_name": "United States",
        "price_per_sms": "0.0643",
        "type": "shortcode",
        "operator_id": "...",
        "updated_at": "2026-11-08T00:00:00.000Z"
      }
    ],
    "ok": true
  }
  ```

---

### 6. HLR Number Lookup (Carrier & Reachability Validation)
- **Single HLR Lookup (€0.01/lookup):** `POST /hlr/get/lookup`
  ```bash
  curl -X POST "https://api.sendsmsnokyc.com/hlr/get/lookup" \
    -H "Authorization: Bearer $SMSROUTE_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"phone": "+447911123456"}'
  ```
  Returns E.164/national formatting, `number_type` (`mobile`, `landline`, `voip`), `is_ported`, `reachable`, `imsi`, `current_network` (`country_iso2`, `mccmnc`, `network_name`), and `original_network`.
- **Bulk HLR Lookup (up to 9,999 numbers):** `POST /hlr/get/lookup/bulk`
  - Body: `{ "phone_numbers": "+14155552671\n+447911123456" }`
- **HLR History:** `GET /hlr/history?limit=50&offset=0`

---

### 7. Live Carrier Route Probe Testing & Route Optimization
Test real handset delivery, latency, and Sender ID preservation across `Route 1`, `Route 2`, `Route 3`, and `Route 4`, then automatically apply the best route for any country:

1. **List Available Test Countries & Carriers:** `GET /test/countries`
2. **Estimate Probe Test Cost:**
   `GET /test/estimate-cost?country_code=GB&message=Probe+Test&selected_route_markers=1,2,3,4`
3. **Run Live Route Probe Test:** `POST /test/run`
   ```json
   {
     "country_code": "GB",
     "sender_id": "vortex",
     "message": "Route verification probe",
     "selected_route_markers": ["1", "2", "3", "4"]
   }
   ```
   Returns `{ "ok": true, "data": { "batch_id": "test_..." } }`.
4. **Poll Test Batch Status:** `GET /test/status?batch_id=test_...`
5. **Fetch Results & Optimal Route Recommendation:** `GET /test/results?batch_id=test_...`
6. **Apply Recommended Route for Country:** `POST /test/apply-route`
   ```json
   {
     "batch_id": "test_..."
   }
   ```
7. **Manage Per-Country Route Defaults Directly:**
   - `GET /user/country/routes` — List your per-country default routes.
   - `POST /user/country/routes` — Set route for a country (`{ "country_code": "GB", "gateway_id": "<id>" }`).
   - `DELETE /user/country/routes/{country}` — Reset country route to automatic best route.

---

### 8. Webhook Delivery Receipts (`callback_url`)
When `callback_url` is provided on `/sms/send` or `/sms/send/bulk`, SMSRoute sends a `POST` request with `Content-Type: application/json` (up to 3 retries with exponential backoff; private/loopback IPs are blocked for SSRF safety):

```json
{
  "event": "sms.delivery_status",
  "timestamp": "2026-11-08T12:00:02.150Z",
  "data": {
    "messageId": "cm...",
    "status": "delivered",
    "to": "+14155552671",
    "deliveredAt": "2026-11-08T12:00:02.000Z",
    "cost": 0.0643,
    "errorCode": null
  }
}
```

---

### 9. SMPP v3.4 over TLS Quick Reference
For high-throughput persistent telecom binds:
- **URL:** `ssmpp://smpp.sendsmsnokyc.com:2775` (TLS from byte 0; never use plaintext `smpp://`).
- **TLS SNI:** Required (`servername: 'smpp.sendsmsnokyc.com'` in Node.js, `server_hostname='smpp.sendsmsnokyc.com'` in Python).
- **Bind Event:** In Node.js `smpp`, call `session.bind_transceiver(...)` inside `session.on('secureConnect', ...)`, **never** on `'connect'`.
- **DLRs:** Acknowledge every `deliver_sm` PDU with `deliver_sm_resp`.

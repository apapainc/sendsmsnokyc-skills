---
name: smsroute-send-sms
description: "Send single, bulk, and campaign SMS messages globally without KYC using the SMSRoute / SendSMSNoKYC API (https://api.sendsmsnokyc.com). Covers E.164 phone number formatting, GSM 7-bit vs UCS-2 Unicode character encoding and segment limits, pre-send cost validation (/sms/validate), single send (/sms/send), bulk send (/sms/send/bulk), SMS campaigns (/campaigns), delivery status polling (/sms/single/{messageId}/status), and DLR webhooks. Use when building OTP, 2FA, transactional alerts, or bulk SMS workflows."
metadata:
  author: SMSRoute
  version: 1.0.0
  category: Messaging
  tags: sms, send-sms, bulk-sms, otp, 2fa, transactional-sms, sendsmsnokyc, smsroute, gsm-7, unicode
---

# SMSRoute Send SMS Skill

## When to Use This Skill
Use this skill whenever an application or script needs to:
- Send 2FA / OTP verification codes or transactional SMS messages globally via `https://api.sendsmsnokyc.com`.
- Dispatch personalized bulk SMS batches (`/sms/send/bulk`) or scheduled campaigns (`/campaigns`).
- Calculate SMS parts and GSM-7 vs UCS-2 encoding before charging/sending (`/sms/validate`).
- Track SMS delivery receipts via polling (`/sms/single/{messageId}/status`) or webhooks (`callback_url`).

---

## Implementation Rules & Best Practices

1. **Phone Number Formatting (E.164):**
   - Always format recipient numbers in international E.164 notation (`+` followed by country dial code and subscriber number, 7–15 digits, e.g. `+14155552671`, `+447911123456`, `+46762390552`).
2. **Character Encoding & Segment Math:**
   - **GSM 7-bit (`GSM_7BIT`):** Up to **160 characters** for a single SMS, or **153 characters per part** for multi-part concatenated messages (up to 6 parts = 918 characters).
   - **Unicode (`UCS2`):** Triggered automatically if the message contains non-GSM characters (e.g., emojis, Cyrillic, Arabic, CJK, or accented characters outside standard GSM-7 like `ä`, `ö`, `å`). Up to **70 characters** for a single SMS, or **67 characters per part** for multi-part messages (up to 6 parts = 402 characters).
   - Always call `POST /sms/validate` if you want to preview the exact `parts`, `encoding`, and `cost.total` before dispatching.
3. **Custom Sender ID (`sender_id`):**
   - Set per-message in the request payload (defaults to `"vortex"` if omitted; no pre-registration or KYC required on `sid_open` routes).
   - **Alphanumeric Sender IDs:** **3–11 characters** (`A–Z`, `a–z`, `0–9`, at least one letter; avoid spaces and symbols, e.g. `"AuthAlert"`, `"BrandVerify"`, `"InfoSMS"`).
   - **Dynamic Numeric Macros:** Use `@NUMBER5@` or `38@NUMBER3@` on SIM/numeric routes to randomize numeric originators per message.
   - **Route Types (`GET /prices`):** `sid_open` preserves your custom alphanumeric `sender_id`; `sid_fixed` uses a pre-registered sender identity; `shortcode` uses a 4–6 digit numeric code; `sim` uses real mobile SIM numbers. See the `smsroute-custom-sender-id` skill for full details.
4. **Route Selection (`marker`):**
   - Omit `marker` to use your configured per-country default route or SMSRoute's automatic best route.
   - Pass `marker: "1"`, `"2"`, `"3"`, or `"4"` to explicitly route through **Route 1**, **Route 2**, **Route 3**, or **Route 4**.

---

## Code Templates

### TypeScript / Node.js: Send OTP & Poll Delivery Status

```typescript
const API_BASE = "https://api.sendsmsnokyc.com";
const API_KEY = process.env.SMSROUTE_API_KEY!;

export async function sendOtpSms(to: string, otpCode: string) {
  const res = await fetch(`${API_BASE}/sms/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      to,
      sender_id: "AuthVerify",
      message: `Your verification code is ${otpCode}. Do not share this code.`,
    }),
  });

  if (!res.ok) {
    const err = await res.text();
    throw new Error(`SMS dispatch failed (${res.status}): ${err}`);
  }

  return (await res.json()) as {
    messageId: string;
    status: string;
    to: string;
    from: string;
    timestamp: string;
    blocked: boolean;
  };
}

export async function getSmsDeliveryStatus(messageId: string) {
  const res = await fetch(`${API_BASE}/sms/single/${encodeURIComponent(messageId)}/status`, {
    headers: {
      Authorization: `Bearer ${API_KEY}`,
    },
  });

  if (!res.ok) {
    throw new Error(`Status lookup failed (${res.status})`);
  }

  const body = await res.json();
  return body.data as {
    messageId: string;
    to: string;
    status: "queued" | "processing" | "sent" | "delivered" | "failed";
    sentAt: string;
    deliveredAt: string | null;
    errorCode: string | null;
    errorMessage: string | null;
  };
}
```

### Python: Send Bulk Personalized SMS

```python
import os
import requests

API_BASE = "https://api.sendsmsnokyc.com"
API_KEY = os.environ["SMSROUTE_API_KEY"]

def send_bulk_sms(recipients: list[dict], sender_id: str = "vortex", callback_url: str | None = None):
    payload = {
        "sender_id": sender_id,
        "recipients": recipients,  # [{"to": "+14155552671", "message": "Hi!"}]
    }
    if callback_url:
        payload["callback_url"] = callback_url

    response = requests.post(
        f"{API_BASE}/sms/send/bulk",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
        },
        json=payload,
        timeout=30,
    )
    response.raise_for_status()
    return response.json()
```

---

## Campaigns API (`/campaigns`)

For large-scale or scheduled broadcasts with macro expansion:
- **Create Campaign:** `POST /campaigns`
  - Body: `{ "name": "Spring Launch", "message": "Hello!", "recipients": ["+14155552671"], "sender_id": "vortex", "scheduled_at": "2026-11-10T14:00:00Z" }`
- **List Campaigns:** `GET /campaigns?page=1&limit=20`
- **Get Campaign Details:** `GET /campaigns/{campaignId}`
- **Pause / Cancel / Clone Campaign:**
  - `PUT /campaigns/{campaignId}/pause`
  - `PUT /campaigns/{campaignId}/cancel`
  - `POST /campaigns/{campaignId}/clone`
- **Export Campaign Report:** `GET /campaigns/{campaignId}/export?format=csv`


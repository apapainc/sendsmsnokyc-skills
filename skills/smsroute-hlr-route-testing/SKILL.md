---
name: smsroute-hlr-route-testing
description: "Perform HLR (Home Location Register) number validation and live multi-carrier SMS route probe tests using the SMSRoute / SendSMSNoKYC API (https://api.sendsmsnokyc.com). Use when validating phone number reachability, detecting ported numbers or carrier MCC/MNC networks (/hlr/get/lookup, /hlr/get/lookup/bulk), or running live probe tests across Route 1–Route 4 (/test/countries, /test/estimate-cost, /test/run, /test/status, /test/results, /test/apply-route) to automatically select the highest-scoring carrier route."
metadata:
  author: SMSRoute
  version: 1.0.0
  category: Telecommunications & Quality Assurance
  tags: hlr, number-lookup, carrier-lookup, mccmnc, route-testing, smsroute, sendsmsnokyc
---

# SMSRoute HLR Lookup & Live Route Testing Skill

## Part 1: HLR Number Lookup (`/hlr/*`)

Validate phone numbers in real time before sending SMS campaigns to eliminate dead numbers, detect landlines/VoIP, and inspect MCC/MNC carrier networks and number portability.

- **Cost:** `€0.01` per lookup (automatically refunded if a lookup fails).
- **Single Lookup:** `POST https://api.sendsmsnokyc.com/hlr/get/lookup`
  ```bash
  curl -X POST "https://api.sendsmsnokyc.com/hlr/get/lookup" \
    -H "Authorization: Bearer $SMSROUTE_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"phone": "+46762390552"}'
  ```
  **Response Fields:**
  - `id`: Lookup ID
  - `cost`: `0.01`
  - `phone_number`: Input number
  - `number_type`: `"mobile"`, `"landline"`, `"voip"`, `"toll_free"`, `"premium"`, or `"unknown"`
  - `timezone`: e.g. `"Europe/Stockholm"`
  - `format`: `{ "e164": "+46762390552", "international": "+46 76 239 05 52", "national": "076-239 05 52", "rfc3966": "tel:+46-76-239-05-52" }`
  - `is_ported`: `boolean`
  - `reachable`: `"reachable"`, `"undeliverable"`, or `"unknown"`
  - `imsi`: 15-digit IMSI prefix
  - `current_network`: `{ "country_iso2": "SE", "country_prefix": 46, "country_name": "Sweden", "mccmnc": "24001", "mcc": "240", "mnc": "01", "area": "Sweden", "network_name": "Telia" }`
  - `original_network`: Same structure as `current_network`

- **Bulk HLR Lookup (up to 9,999 numbers per call):** `POST https://api.sendsmsnokyc.com/hlr/get/lookup/bulk`
  ```json
  {
    "phone_numbers": "+14155552671\n+447911123456\n+46762390552"
  }
  ```
- **HLR Lookup History:** `GET https://api.sendsmsnokyc.com/hlr/history?limit=50&offset=0`

---

## Part 2: Live Carrier Route Probe Testing (`/test/*`)

Probe real test handsets on destination mobile carriers across **Route 1**, **Route 2**, **Route 3**, and **Route 4** to verify:
1. Delivery receipt (DLR) arrival & latency
2. Sender ID preservation (whether your custom alphanumeric Sender ID arrived intact or was overwritten)
3. Message content integrity (no word filtering or character corruption)

### End-to-End Automated Route Probing Workflow

```typescript
const API_BASE = "https://api.sendsmsnokyc.com";
const headers = {
  Authorization: `Bearer ${process.env.SMSROUTE_API_KEY}`,
  "Content-Type": "application/json",
};

export async function probeAndApplyBestRoute(countryCode: string, senderId: string, sampleMessage: string) {
  // Step 1: Estimate test cost
  const estRes = await fetch(
    `${API_BASE}/test/estimate-cost?country_code=${countryCode}&message=${encodeURIComponent(sampleMessage)}&selected_route_markers=1,2,3,4`,
    { headers }
  );
  const estimate = await estRes.json();
  console.log("Estimated probe cost (EUR):", estimate.data?.total);

  // Step 2: Launch live carrier probe batch
  const runRes = await fetch(`${API_BASE}/test/run`, {
    method: "POST",
    headers,
    body: JSON.stringify({
      country_code: countryCode,
      sender_id: senderId,
      message: sampleMessage,
      selected_route_markers: ["1", "2", "3", "4"],
    }),
  });
  const runData = await runRes.json();
  const batchId = runData.data.batch_id;

  // Step 3: Poll until complete (bounded polling, max 90 seconds)
  for (let i = 0; i < 18; i++) {
    await new Promise((r) => setTimeout(r, 5000));
    const statusRes = await fetch(`${API_BASE}/test/status?batch_id=${encodeURIComponent(batchId)}`, { headers });
    const statusData = await statusRes.json();
    if (statusData.data?.is_complete) break;
  }

  // Step 4: Get telemetry results & recommendation
  const resultsRes = await fetch(`${API_BASE}/test/results?batch_id=${encodeURIComponent(batchId)}`, { headers });
  const results = await resultsRes.json();
  console.log("Recommended route:", results.data?.recommendation);

  // Step 5: Apply recommended route as default for this country
  if (results.data?.recommendation) {
    const applyRes = await fetch(`${API_BASE}/test/apply-route`, {
      method: "POST",
      headers,
      body: JSON.stringify({ batch_id: batchId }),
    });
    return await applyRes.json();
  }

  return results;
}
```

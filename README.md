# SendSMSNoKYC / SMSRoute Agent Skills

Official AI Agent Skills for **[SendSMSNoKYC (SMSRoute)](https://sendsmsnokyc.com)** — the privacy-first, zero-KYC Application-to-Person (A2P) outbound SMS, OTP, HLR Lookup, and SMPP v3.4 over TLS gateway (`https://api.sendsmsnokyc.com`).

## Install via `skills.sh`

Install all skills into your AI coding agent (Claude Code, Cursor, Antigravity, Codex, Windsurf, Cline, etc.):

```bash
npx skills add FridayCandour/sendsmsnokyc-skills --all
```

Or install a specific skill:

```bash
npx skills add FridayCandour/sendsmsnokyc-skills --skill sendsmsnokyc-api
npx skills add FridayCandour/sendsmsnokyc-skills --skill smsroute-send-sms
npx skills add FridayCandour/sendsmsnokyc-skills --skill smsroute-smpp-gateway
npx skills add FridayCandour/sendsmsnokyc-skills --skill smsroute-hlr-route-testing
```

## Included Skills

| Skill | Description |
| :--- | :--- |
| **`sendsmsnokyc-api`** | Complete end-to-end reference for the `https://api.sendsmsnokyc.com` REST API & SMPP TLS gateway (`ssmpp://smpp.sendsmsnokyc.com:2775`). |
| **`smsroute-send-sms`** | Send single, bulk, and campaign SMS messages globally, validate GSM-7 vs UCS-2 segments & cost (`/sms/validate`), and handle DLR webhooks. |
| **`smsroute-smpp-gateway`** | Connect and bind to `ssmpp://smpp.sendsmsnokyc.com:2775` over TLS with SNI in Node.js (`smpp`) and Python (`smpplib`). |
| **`smsroute-hlr-route-testing`** | Perform real-time HLR carrier/portability lookups (`/hlr/get/lookup`) and automated multi-carrier route probe tests (`/test/run`, `/test/apply-route`). |

## Links

- **Platform:** [https://sendsmsnokyc.com](https://sendsmsnokyc.com)
- **API Documentation:** [https://sendsmsnokyc.com/api-docs](https://sendsmsnokyc.com/api-docs)
- **OpenAPI 3.0 Spec:** [`openapi.yaml`](./openapi.yaml)

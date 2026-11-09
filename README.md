# SendSMSNoKYC / SMSRoute Agent Skills

Official Agent Skills for integrating with **SendSMSNoKYC (SMSRoute)** (`https://sendsmsnokyc.com` / `https://api.sendsmsnokyc.com`), the privacy-first, zero-KYC global A2P SMS, OTP, HLR Lookup, and SMPP v3.4 over TLS platform.

## Install via skills.sh

```bash
npx skills add apapainc/sendsmsnokyc-skills
```

## Included Skills

| Skill | Description |
| :--- | :--- |
| [`sendsmsnokyc-api`](./skills/sendsmsnokyc-api/SKILL.md) | Complete REST API & SMPP v3.4 reference for single/bulk SMS, Custom Sender IDs, DLR webhooks, HLR lookup, live carrier route probe testing, balance, and country pricing. |
| [`smsroute-send-sms`](./skills/smsroute-send-sms/SKILL.md) | Focused guide and TypeScript/Python templates for sending OTP, 2FA, transactional, bulk, and campaign SMS with GSM-7 vs UCS-2 segment validation. |
| [`smsroute-custom-sender-id`](./skills/smsroute-custom-sender-id/SKILL.md) | Complete guide to Custom Sender IDs (`sender_id` / `source_addr`), 11-char alphanumeric formatting rules, route types (`sid_open`, `sid_fixed`, `shortcode`, `sim`), dynamic `@NUMBER@` macros, and live Sender ID preservation testing. |
| [`smsroute-smpp-gateway`](./skills/smsroute-smpp-gateway/SKILL.md) | Production SMPP v3.4 over TLS (`ssmpp://smpp.sendsmsnokyc.com:2775`) guide for Node.js (`smpp`) and Python (`smpplib`) with TLS SNI and `secureConnect` handling. |
| [`smsroute-hlr-route-testing`](./skills/smsroute-hlr-route-testing/SKILL.md) | HLR number reachability & MCC/MNC portability lookups (`/hlr/get/lookup`) and live multi-carrier route probe testing (`/test/run`, `/test/apply-route`). |

## Links

- **Platform:** [https://sendsmsnokyc.com](https://sendsmsnokyc.com)
- **REST API Base URL:** `https://api.sendsmsnokyc.com`
- **SMPP v3.4 over TLS:** `ssmpp://smpp.sendsmsnokyc.com:2775`
- **Documentation:** [https://sendsmsnokyc.com/api-docs](https://sendsmsnokyc.com/api-docs)

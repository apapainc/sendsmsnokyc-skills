---
name: smsroute-smpp-gateway
description: "Connect and bind to the SMSRoute / SendSMSNoKYC SMPP v3.4 over TLS gateway (ssmpp://smpp.sendsmsnokyc.com:2775). Prevents common ECONNRESET and TLS SNI handshake errors in Node.js (smpp) and Python (smpplib), handles GSM-7 and UCS-2 data_coding, submit_sm, and deliver_sm delivery receipt acknowledgments. Use whenever configuring an SMPP client, debugging SMPP bind failures, or building high-throughput telecom integrations with smpp.sendsmsnokyc.com."
metadata:
  author: SMSRoute
  version: 1.0.0
  category: Telecommunications & SMPP
  tags: smpp, smpp-tls, ssmpp, telecom, sendsmsnokyc, smsroute, deliver_sm, submit_sm
---

# SMSRoute SMPP v3.4 over TLS Gateway Skill

## Critical Connection Specifications

| Parameter | Canonical Value | Critical Requirement |
| :--- | :--- | :--- |
| **Host** | `smpp.sendsmsnokyc.com` | Anycast hostname. Resolve per session; do not hardcode an IP address. |
| **Port** | `2775` | Speaks **TLS from the very first byte**. Never accepts plaintext SMPP. |
| **Scheme** | `ssmpp://` | `ssmpp://smpp.sendsmsnokyc.com:2775`. Using `smpp://` writes a cleartext bind PDU into the TLS handshake and causes immediate `ECONNRESET`. |
| **TLS SNI** | `smpp.sendsmsnokyc.com` | Required by the edge proxy (`servername` in Node.js, `server_hostname` in Python). Omitting SNI drops the TLS handshake. |
| **Protocol** | SMPP v3.4 (`0x34`) | `bind_transceiver` is recommended so `submit_sm` and `deliver_sm` (DLRs) share one session. |
| **Credentials** | `SMPP_SYSTEM_ID` / `SMPP_PASSWORD` | Found in the SMSRoute dashboard under **SMPP Settings**. Always load from environment variables. |

---

## Reference Implementation: Node.js (`smpp`)

> **Crucial Rule for Node.js `smpp`:**
> Always bind inside `session.on('secureConnect', ...)`, **NEVER** inside `session.on('connect', ...)`. The `'connect'` event fires when the raw TCP socket opens before the TLS handshake completes; sending `bind_transceiver` on `'connect'` injects cleartext bytes into the TLS handshake and results in `ECONNRESET`.

```javascript
// npm install smpp
const smpp = require("smpp");

const SMPP_HOST = "smpp.sendsmsnokyc.com";
const SMPP_PORT = 2775;

const SYSTEM_ID = process.env.SMPP_SYSTEM_ID;
const PASSWORD = process.env.SMPP_PASSWORD;

// 1. Use ssmpp:// and pass servername for TLS SNI routing
const session = smpp.connect({
  url: `ssmpp://${SMPP_HOST}:${SMPP_PORT}`,
  servername: SMPP_HOST,
  rejectUnauthorized: false,
  auto_enquire_link_period: 15000,
});

// 2. Bind ONLY on 'secureConnect' after TLS handshake finishes
session.on("secureConnect", () => {
  session.bind_transceiver(
    {
      system_id: SYSTEM_ID,
      password: PASSWORD,
    },
    (pdu) => {
      if (pdu.command_status !== 0) {
        console.error("SMPP bind failed: 0x" + pdu.command_status.toString(16));
        session.close();
        return;
      }
      console.log("Bound to SMSRoute SMPP over TLS");
      sendSMS("+14155552671", "AuthAlert", "Your verification code is 849201");
    }
  );
});

// 3. Always acknowledge incoming delivery receipts (deliver_sm)
session.on("deliver_sm", (pdu) => {
  session.deliver_sm_resp({
    sequence_number: pdu.sequence_number || 0,
    command_status: 0,
    message_id: "",
  });
});

// 4. Submit SMS (source_addr_ton: 5 = alphanumeric, dest_addr_ton: 1 = international)
function sendSMS(destination, senderId, text) {
  session.submit_sm(
    {
      source_addr: senderId,
      source_addr_ton: 5,
      source_addr_npi: 0,
      destination_addr: destination,
      dest_addr_ton: 1,
      dest_addr_npi: 1,
      short_message: text,
      registered_delivery: 1,
      data_coding: "GSM_7BIT",
    },
    (pdu) => {
      if (pdu.command_status === 0) {
        console.log("Submitted, message_id:", pdu.message_id);
      } else {
        console.error("submit_sm failed: 0x" + pdu.command_status.toString(16));
      }
    }
  );
}

session.on("error", (err) => console.error("SMPP session error:", err));
```

---

## Reference Implementation: Python 3 (`smpplib`)

> **Crucial Rules for Python `smpplib`:**
> 1. `smpplib` truncates `system_id` at 15 bytes and `password` at 9 bytes by default. Raise both `max` limits to `64` before binding.
> 2. `smpplib`'s built-in `connect(ssl=True)` does not send TLS SNI (`server_hostname`). Wrap the socket manually with `ssl.create_default_context().wrap_socket(raw_sock, server_hostname=HOST)`.

```python
# pip install smpplib
import os
import socket
import ssl
import smpplib.client
import smpplib.command
import smpplib.consts
import smpplib.gsm

# 1. Raise credential length limits so longer system_id/password are not truncated
smpplib.command.BindTransmitter.params["system_id"].max = 64
smpplib.command.BindTransmitter.params["password"].max = 64
smpplib.command.BindTransceiver.params["system_id"].max = 64
smpplib.command.BindTransceiver.params["password"].max = 64

HOST = "smpp.sendsmsnokyc.com"
PORT = 2775

SYSTEM_ID = os.environ["SMPP_SYSTEM_ID"]
PASSWORD = os.environ["SMPP_PASSWORD"]

client = smpplib.client.Client(HOST, PORT, allow_unknown_opt_params=True)

# 2. Wrap TCP socket in TLS with explicit SNI (server_hostname)
raw_sock = socket.create_connection((client.host, client.port), timeout=15)
ssl_ctx = ssl.create_default_context()
client._socket = ssl_ctx.wrap_socket(raw_sock, server_hostname=client.host)
client.state = smpplib.consts.SMPP_CLIENT_STATE_OPEN

try:
    client.bind_transceiver(system_id=SYSTEM_ID, password=PASSWORD)
    print("Bound to SMSRoute SMPP over TLS")

    parts, encoding_flag, msg_type_flag = smpplib.gsm.make_parts("Hello from SMSRoute SMPP")
    for part in parts:
        pdu = client.send_message(
            source_addr_ton=smpplib.consts.SMPP_TON_ALNUM,
            source_addr="AuthAlert",
            dest_addr_ton=smpplib.consts.SMPP_TON_INTL,
            destination_addr="14155552671",  # International digits without leading '+'
            short_message=part,
            data_coding=encoding_flag,
            esm_class=msg_type_flag,
            registered_delivery=True,
        )
        print(f"Submitted PDU sequence: {pdu.sequence}")
finally:
    client.unbind()
```


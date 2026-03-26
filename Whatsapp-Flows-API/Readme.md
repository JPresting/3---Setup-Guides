# WhatsApp Flows — Setup Guide

> Encrypted multi-screen forms inside WhatsApp, powered by n8n.

---

## How It Works

WhatsApp Flows let you build multi-screen forms that run inside WhatsApp. All communication between the client and your server is **end-to-end encrypted** (RSA + AES-128-GCM).

When a user clicks a Flow button in a template message, WhatsApp opens the form. The user fills out screens, clicks "Next", and at the end clicks "Submit". The submitted data arrives as a WhatsApp message at your webhook.

---

## Key Concept: One Key Pair Per Phone Number

The RSA encryption key pair is registered at Meta for a **phone number**, not for a flow. If you have 20 flows on the same number, they ALL use the same key pair. Every endpoint for that number decrypts with the same private key.

**If you regenerate the key pair, ALL existing flows on that number break** until you update the private key in every endpoint.

Rule: generate once, store safely, never regenerate unless you intentionally want to rotate.

---

## IDs You Need

| ID | What it is | Where to find |
|----|-----------|---------------|
| Phone Number ID | Identifies your WhatsApp number | WhatsApp Manager → Phone Numbers |
| WABA ID | Your WhatsApp Business Account | Business Settings → Accounts → WhatsApp Accounts |
| App ID | Your Meta App | Developer Portal → App Settings → Basic |
| Flow ID | Identifies a specific flow | WhatsApp Manager → Flows |

> **Phone Number ID ≠ WABA ID.** The encryption endpoint uses the Phone Number ID. Mixing these up causes `oaep decoding error`.

---

## Setup Steps

### 1. Generate & Register Key Pair (run once)

n8n Code Node — generates RSA-2048 key pair and registers the public key at Meta in one step:

```javascript
const crypto = require('crypto');
const https = require('https');

const WABAphoneID = "<PHONE_NUMBER_ID>";
const accessToken = "<SYSTEM_USER_TOKEN>";

const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
  modulusLength: 2048,
  publicKeyEncoding: { type: 'spki', format: 'pem' },
  privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
});

const body = "business_public_key=" + encodeURIComponent(publicKey);

const result = await new Promise((resolve, reject) => {
  const req = https.request({
    hostname: 'graph.facebook.com',
    path: '/v25.0/' + WABAphoneID + '/whatsapp_business_encryption',
    method: 'POST',
    headers: {
      'Authorization': 'Bearer ' + accessToken,
      'Content-Type': 'application/x-www-form-urlencoded',
      'Content-Length': Buffer.byteLength(body)
    }
  }, (res) => {
    let data = '';
    res.on('data', chunk => data += chunk);
    res.on('end', () => resolve(JSON.parse(data)));
  });
  req.on('error', reject);
  req.write(body);
  req.end();
});

return [{ json: { registration: result, publicKey, privateKey } }];
```

**Critical:** Must be `application/x-www-form-urlencoded` with `encodeURIComponent`. JSON body type silently corrupts the PEM key — Meta returns `success: true` but the key is broken.

Copy the `privateKey` from the output → paste into your endpoint code. Never run this again unless you want to rotate keys.

### 2. Create the Encryption Endpoint (always active)

n8n workflow: **Webhook** → **Code Node** → **Respond to Webhook**

- Webhook: POST, Response Mode "Respond to Webhook"
- Respond to Webhook: Body `{{ $json.signature }}`, plain text, status 200

Code Node:

```javascript
const crypto = require('crypto');

const privateKeyPem = `<YOUR PRIVATE KEY>`;

const ROUTING = {
  "SCREEN_A": "SCREEN_B",
  "SCREEN_B": "SCREEN_C"
  // ... your screen order here
};

const encFlowDataB64 = $input.first().json.body.encrypted_flow_data;
const encAesKeyB64 = $input.first().json.body.encrypted_aes_key;
const ivB64 = $input.first().json.body.initial_vector;

const encKeyBuffer = Buffer.from(encAesKeyB64, 'base64');
const ivBuffer = Buffer.from(ivB64, 'base64');

const aesKeyBuffer = crypto.privateDecrypt({
  key: privateKeyPem,
  padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
  oaepHash: 'sha256'
}, encKeyBuffer);

const encFlowDataBuffer = Buffer.from(encFlowDataB64, 'base64');
const encData = encFlowDataBuffer.slice(0, -16);
const tagFromRequest = encFlowDataBuffer.slice(-16);
const decipher = crypto.createDecipheriv('aes-128-gcm', aesKeyBuffer, ivBuffer);
decipher.setAuthTag(tagFromRequest);
let decrypted = decipher.update(encData, null, 'utf8');
decrypted += decipher.final('utf8');
const payload = JSON.parse(decrypted);

let responseData;
if (payload.action === 'ping') {
  responseData = { version: "3.0", data: { status: "active" } };
} else if (payload.action === 'INIT') {
  responseData = { version: "3.0", screen: "FIRST_SCREEN", data: {} };
} else if (payload.action === 'data_exchange') {
  const next = ROUTING[payload.screen];
  responseData = next
    ? { version: "3.0", screen: next, data: {} }
    : { version: "3.0", data: {} };
} else {
  responseData = { version: "3.0", data: { status: "active" } };
}

const flippedIv = Buffer.from(ivBuffer.map(b => b ^ 0xff));
const plaintext = JSON.stringify(responseData);
const cipher = crypto.createCipheriv('aes-128-gcm', aesKeyBuffer, flippedIv);
let encrypted = cipher.update(plaintext, 'utf8');
encrypted = Buffer.concat([encrypted, cipher.final()]);
const responseAuthTag = cipher.getAuthTag();
const encryptedWithTag = Buffer.concat([encrypted, responseAuthTag]);

return { json: {
  signature: encryptedWithTag.toString('base64'),
  decrypted_action: payload.action,
  decrypted_screen: payload.screen || null,
  decrypted_data: payload.data || null,
  flow_token: payload.flow_token || null
}};
```

Update the `ROUTING` object when you change your flow's screen order. Keep the endpoint fast — no external API calls.

### 3. Cloudflare Bypass (if applicable)

If n8n is behind Cloudflare Zero Trust, Meta's requests get blocked. Create an Access Application with a Bypass policy for your webhook path.

### 4. Health Check

Once the endpoint is active and the flow is published, Meta sends periodic `ping` requests. The code above handles this automatically. In WhatsApp Manager → Flows → Endpoint tab, verify the health check passes before publishing.

---

## Screen Navigation

There are three ways a screen can transition to the next. The choice depends on what the screen contains and whether you need server involvement.

### navigate — client-side, no server

The client (WhatsApp) handles the transition directly. No request is sent to your endpoint. The next screen is defined in the Flow JSON itself.

Use this for screens that have **no file uploads** and don't need server-side logic.

```json
"on-click-action": {
  "name": "navigate",
  "next": { "type": "screen", "name": "NEXT_SCREEN" },
  "payload": {}
}
```

The `next` object tells WhatsApp which screen to show. This happens instantly — no network round-trip, no encryption overhead, no timeout risk.

**Limitation:** DocumentPicker and PhotoPicker values are **not allowed** in navigate payloads. If a screen has a file upload component, you must use `data_exchange`.

### data_exchange — server-side, your endpoint decides

WhatsApp encrypts the form data, sends it to your endpoint, and waits for an encrypted response that tells it which screen to show next. Your endpoint controls the routing.

Use this for screens with **file uploads** (DocumentPicker/PhotoPicker) or when you want to **store the form data server-side** on each step.

In the Flow JSON:
```json
"on-click-action": {
  "name": "data_exchange",
  "payload": {
    "vorname": "${form.vorname}",
    "nachname": "${form.nachname}",
    "personalausweis": "${form.personalausweis}"
  }
}
```

Your endpoint receives this payload (decrypted), and must respond with the next screen:
```javascript
// In your endpoint Code Node — the ROUTING object defines screen order
const ROUTING = {
  "PERSONAL_DATA": "QUALIFIKATION",
  "QUALIFIKATION": "DOC_UPLOAD",
  "DOC_UPLOAD": "PHOTO_UPLOAD"
};

// When a data_exchange request comes in:
if (payload.action === 'data_exchange') {
  const next = ROUTING[payload.screen];
  responseData = { version: "3.0", screen: next, data: {} };
}
```

The `ROUTING` object in your endpoint code maps each screen to its successor. When you add, remove, or reorder screens in your Flow JSON, update this object to match.

**Key point:** The `routing_model` in the Flow JSON and the `ROUTING` in your endpoint code must be consistent. The Flow JSON `routing_model` tells Meta which transitions are allowed (for validation). The endpoint `ROUTING` tells your server which screen to actually return.

### complete — final submission

Used on the last screen only. Sends the payload as a WhatsApp message to your WhatsApp Trigger webhook. No server endpoint call.

```json
"on-click-action": {
  "name": "complete",
  "payload": {
    "consent_a": "${form.consent_a}",
    "consent_b": "${form.consent_b}"
  }
}
```

The `complete` payload arrives as an `interactive.nfm_reply` message. Only fields listed in this payload are included — previous screens' data is NOT automatically collected. If you need all data, either chain it through screens (complex) or use `data_exchange` on every screen to store data server-side as the user progresses.

### Summary

| Action | Who decides next screen? | File uploads? | Data sent to server? |
|--------|------------------------|---------------|---------------------|
| `navigate` | Flow JSON (`next` field) | ❌ Not allowed | No |
| `data_exchange` | Your endpoint (`ROUTING`) | ✅ Required for uploads | Yes — encrypted to endpoint |
| `complete` | N/A (flow ends) | ❌ | As WhatsApp message to trigger |

For full component reference: [Components Reference](https://developers.facebook.com/docs/whatsapp/flows/reference/components), [Flow JSON Reference](https://developers.facebook.com/docs/whatsapp/flows/reference/flowjson)

---

## Identifying Users: flow_token

By default `flow_token` is `"unused"`. Pass the recipient's phone number when sending the template:

```json
{
  "type": "button",
  "sub_type": "flow",
  "index": "0",
  "parameters": [{
    "type": "action",
    "action": { "flow_token": "<RECIPIENT_PHONE_NUMBER>" }
  }]
}
```

Every `data_exchange` and `complete` request then includes this token. Use it to identify and group data per user. If the user goes back and resubmits a screen, the same `flow_token` comes in — overwrite the old data.

---


See the video below. This is from a German personnel questionnaire. Obviously, with every slide, the information needs to be sent to the server, and the longer such a questionnaire gets, the more likely it is users will want to go back and forth and correct information. So the flow stores the input with every hit on the "further" button.

[Video Demonstration](https://github.com/user-attachments/assets/441b68c8-e770-495f-bfb5-e35c57582704)

This then leads to the Webhook being called:

![Webhook call](https://github.com/user-attachments/assets/83d0a28f-a8bf-4877-9986-b24abbbc249d)

which then decrypts the input associated with the correct flowtoken / phone number:

![Decryption process](https://github.com/user-attachments/assets/f20701d9-52d0-44be-93de-360ba168b17e)

This also means the storage and logic of which sites of the flow have been submitted and if the entire flow was submitted or only partially.

## Constraints

| Rule | Limit |
|------|-------|
| Screens per flow | 100 max |
| Components per screen | 50 max |
| OptIn per screen | 5 max |
| DocumentPicker/PhotoPicker per screen | 1 max (never both) |
| Screen ID format | Letters + underscores only (no numbers) |
| Label length | 20 characters max |
| Endpoint response time | ~5-10 seconds max |
| Media ID validity | 30 days |
| Key pair scope | Per phone number, not per flow |

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `oaep decoding error` | Private key doesn't match registered public key | Re-register or regenerate key pair, update ALL endpoints |
| `"Es gab ein Problem"` | Endpoint too slow or wrong response format | Remove external API calls, keep routing local |
| `success: true` but decryption fails | Key registration used JSON instead of form-urlencoded | Use `x-www-form-urlencoded` + `encodeURIComponent` |
| Health check fails | Workflow not active or Cloudflare blocks requests | Activate workflow + configure bypass |
| `{{1}}` becomes `1` in n8n | n8n interprets `{{ }}` as expression | Build JSON body in Code Node instead |

---

## References

- [WhatsApp Flows Docs](https://developers.facebook.com/docs/whatsapp/flows)
- [Endpoint Guide](https://developers.facebook.com/docs/whatsapp/flows/guides/implementingyourflowendpoint)
- [Flow JSON Reference](https://developers.facebook.com/docs/whatsapp/flows/reference/flowjson)
- [Components Reference](https://developers.facebook.com/docs/whatsapp/flows/reference/components)
- [Encryption Setup](https://developers.facebook.com/docs/whatsapp/cloud-api/reference/whatsapp-business-encryption)

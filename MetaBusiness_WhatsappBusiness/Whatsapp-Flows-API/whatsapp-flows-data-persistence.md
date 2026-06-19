# Persisting WhatsApp Flow Responses with n8n + Supabase

> How to capture, identify, and store multi-screen Flow data — including file uploads — and reliably trigger downstream processing.

This guide is the **persistence layer** that sits on top of a working Flow endpoint. It assumes you already have an encrypted endpoint that decrypts incoming requests and returns the next screen (see the *WhatsApp Flows — Setup Guide* for that part). Here we answer the harder question: **once data flows in, how do you store it cleanly and act on it?**

All identifiers below are placeholders (`<...>`). Swap in your own.

---

## The Core Problem

WhatsApp Flows do **not** hand you the full form at the end. Two facts drive the entire design:

1. **Each `data_exchange` contains only the current screen's data.** When the user taps "Next", you receive the fields of *that* screen — nothing from earlier screens.
2. **`complete` only sends the final screen's payload.** It does **not** collect everything the user entered along the way.

Therefore: if you want the complete submission, you must **persist every screen server-side as the user progresses**. There is no "give me everything" call. This guide builds that persistence correctly, including the case where users go back and correct earlier answers.

---

## Step 1 — Identify the User

Every `data_exchange` and `complete` request carries a `flow_token`. By default it is `"unused"`. You set it when sending the Flow template, and it becomes your primary key for grouping a user's answers across all screens.

The natural identifier is the recipient's **phone number** — it is unique, stable, and you already have it. If you also need a display value (e.g. for naming generated documents), pack a second value into the token with a delimiter that never appears in either part:

```json
{
  "type": "button",
  "sub_type": "flow",
  "index": "0",
  "parameters": [{
    "type": "action",
    "action": { "flow_token": "<PHONE_NUMBER>|<FULL_NAME>" }
  }]
}
```

Use a pipe `|` (not an underscore or space — names and numbers can contain those). In your endpoint, split it once after decryption:

```javascript
const [phone, fullName] = (payload.flow_token || '').split('|');
```

Then carry `phone` as your database key and `fullName` as a separate field. **Keep the delimiter out of the key itself** — a `|` in a primary key or a storage path will break things downstream.

> If you have an upstream system (CRM, ticket board, ATS), that is where the phone + name come from when you trigger the template. The Flow itself does not need to know the name.

---

## Step 2 — The Data Model

One row per user, keyed by phone number. Store the answers as JSON, **nested per screen** — not flat.

```sql
CREATE TABLE IF NOT EXISTS <schema>.flow_responses (
  flow_token   text PRIMARY KEY,   -- the phone number
  full_name    text,
  data         jsonb DEFAULT '{}'::jsonb,
  last_screen  text,
  completed    boolean DEFAULT false,
  updated_at   timestamptz DEFAULT now()
);

ALTER TABLE <schema>.flow_responses ENABLE ROW LEVEL SECURITY;
```

### Why nested-per-screen matters

A flat merge (`{vorname, nachname, consent, ...}` all in one object) breaks the moment a user navigates **back** and re-submits a screen with a field removed — you can no longer tell which fields belong to which screen, and stale values linger. Nesting by screen solves both problems:

```json
{
  "WELCOME":      { "consent": true },
  "PERSON_BASIC": { "vorname": "...", "nachname": "...", "geschlecht": "m" },
  "CONTACT":      { "email": "...", "phone": "..." }
}
```

When a screen is re-submitted, you **replace its entire block** — old values vanish, the rest of the document is untouched. You also get clean grouping for free, which matters when you later render this into a report.

`last_screen` lets you see where someone abandoned the flow. `completed` is the flag that gates downstream processing (see Step 6).

---

## Step 3 — Persistence RPCs

Wrap writes in `SECURITY DEFINER` functions so the workflow never touches the table directly and the logic lives in one place.

**Save / update a screen block** (replaces the block for that screen, leaves others intact):

```sql
CREATE OR REPLACE FUNCTION <schema>.save_flow_data(p_token text, p_screen text, p_data jsonb)
RETURNS json LANGUAGE plpgsql SECURITY DEFINER AS $$
BEGIN
  INSERT INTO <schema>.flow_responses (flow_token, data, last_screen, updated_at)
  VALUES (p_token, jsonb_build_object(p_screen, p_data), p_screen, now())
  ON CONFLICT (flow_token) DO UPDATE
    SET data        = jsonb_set(<schema>.flow_responses.data,
                                ARRAY[p_screen],
                                EXCLUDED.data -> p_screen,
                                true),
        last_screen = EXCLUDED.last_screen,
        updated_at  = now();
  RETURN json_build_object('status','ok','screen',p_screen);
END $$;
```

**Mark the submission complete:**

```sql
CREATE OR REPLACE FUNCTION <schema>.mark_completed(p_token text)
RETURNS json LANGUAGE sql SECURITY DEFINER AS $$
  UPDATE <schema>.flow_responses SET completed = true, updated_at = now()
  WHERE flow_token = p_token
  RETURNING json_build_object('status','completed');
$$;
```

Call these over PostgREST's RPC endpoint. In n8n, an HTTP Request node (importable as cURL):

```bash
curl --location '<SUPABASE_INTERNAL_HOST>/rest/v1/rpc/save_flow_data' \
--header 'apikey: <SERVICE_ROLE_KEY>' \
--header 'Authorization: Bearer <SERVICE_ROLE_KEY>' \
--header 'Content-Profile: <schema>' \
--header 'Content-Type: application/json' \
--data '{
  "p_token": "{{ $json.flow_token }}",
  "p_screen": "{{ $json.current_screen }}",
  "p_data": {{ JSON.stringify($json.screen_data || {}) }}
}'
```

Notes that save hours of debugging:

- In a "Using JSON" body, expressions inside `{{ }}` are evaluated automatically — **do not** prefix them with `=`. A leading `=` is stored as literal text.
- `p_token`/`p_screen` are strings (wrapped in quotes). `p_data` is a raw object — `{{ JSON.stringify(...) }}` with **no** surrounding quotes, so it lands as JSON, not a string.
- For RPC/POST the header is `Content-Profile`. For a plain GET query it is `Accept-Profile`. Mixing these up silently queries the wrong schema.

---

## Step 4 — Wire the Endpoint: Respond First, Persist in Parallel

The endpoint has a hard latency budget (≈5–10 s) before WhatsApp shows the user an error. **Never block the response on a database write.** Structure the workflow so the encrypted reply leaves immediately, and persistence runs on a parallel branch:

```
Webhook → Get Key → Decrypt/Route (Code) → Respond to Webhook     ← answers WhatsApp instantly
                          │
                          └─→ persistence branch (runs independently)
```

The Code node that decrypts the request outputs both the `signature` (for the response) **and** the decrypted fields (`flow_token`, `current_screen`, `screen_data`). The `Respond to Webhook` node sends `signature`; the parallel branch consumes the decrypted fields.

### Text and files are not either/or

A common mistake is to branch "if the screen has a file, handle the file; otherwise save text". A single screen can contain **both** a file and text fields. So:

- **Always** run text persistence (`save_flow_data`) on every qualifying `data_exchange`.
- **Additionally** run the media branch *when* a file is present.

Gate the persistence branch so pings and empty payloads don't create junk rows:

```
IF  action == "data_exchange"  AND  flow_token is not empty   →  persist
else                                                          →  do nothing
```

(Health-check `ping` requests carry no `flow_token`; without this gate they all collapse into one empty row via the primary key.)

---

## Step 5 — File Uploads: Download, Decrypt, Store, Reference

This is the part that surprises people. A Flow file upload does **not** give you a downloadable media object. The `media_id` you receive is **not** resolvable through the Graph API (`GET /<media_id>` returns "object does not exist"). Instead, the payload hands you an **encrypted file on a CDN** plus the keys to decrypt it:

```json
"<field>": [{
  "media_id": "...",
  "file_name": "...",
  "cdn_url": "https://.../...enc?...",
  "encryption_metadata": {
    "encryption_key": "...", "hmac_key": "...", "iv": "...",
    "plaintext_hash": "...", "encrypted_hash": "..."
  }
}]
```

### Extract every file generically

The field name differs per screen (`photo`, `id_card`, `certificate`, …) and a single field is an **array** (a user may upload several files). Never hard-code a path like `photo[0]`. Iterate:

```javascript
const [phone] = ($json.flow_token || '').split('|');
const data = $json.screen_data || {};
const out = [];

for (const [field, value] of Object.entries(data)) {
  if (Array.isArray(value)) {
    value.forEach((entry, idx) => {
      if (entry && entry.media_id && entry.cdn_url) {
        out.push({ json: {
          flow_token: phone,
          screen: $json.current_screen,
          field, file_index: idx,
          file_name: entry.file_name || (field + '_' + idx),
          cdn_url: entry.cdn_url,
          encryption_metadata: entry.encryption_metadata
        }});
      }
    });
  }
}
return out;   // one item per file
```

(`index` is a reserved key on an n8n item — use `file_index`, and wrap each output in `{ json: {...} }`.)

### Decrypt (same scheme for every file type)

WhatsApp media encryption is AES-256-CBC with a trailing 10-byte HMAC. Identical for images, PDFs, anything:

```javascript
const crypto = require('crypto');
const item = $json;
const meta = item.encryption_metadata;

const resp = await this.helpers.httpRequest({
  method: 'GET', url: item.cdn_url, encoding: 'arraybuffer'   // bytes only — no returnFullResponse
});
const full = Buffer.from(resp);

const ciphertext = full.slice(0, -10);
const mac        = full.slice(-10);
const encKey  = Buffer.from(meta.encryption_key, 'base64');
const hmacKey = Buffer.from(meta.hmac_key, 'base64');
const iv      = Buffer.from(meta.iv, 'base64');

// verify integrity: HMAC over iv + ciphertext, first 10 bytes
const expected = crypto.createHmac('sha256', hmacKey)
  .update(Buffer.concat([iv, ciphertext])).digest().slice(0, 10);
if (!expected.equals(mac)) throw new Error('HMAC mismatch');

const decipher = crypto.createDecipheriv('aes-256-cbc', encKey, iv);
const plaintext = Buffer.concat([decipher.update(ciphertext), decipher.final()]);

if (crypto.createHash('sha256').update(plaintext).digest('base64') !== meta.plaintext_hash) {
  throw new Error('Plaintext hash mismatch');
}

// build a clean, URL-safe name — do NOT reuse the original file_name
// (WhatsApp file names can contain spaces and invisible control characters)
const ext = (item.file_name || 'file.bin').split('.').pop().trim();
const safeName = `${item.field}_${item.file_index ?? 0}.${ext}`;

return [{
  json: { flow_token: item.flow_token, screen: item.screen, field: item.field, safe_name: safeName },
  binary: { data: await this.helpers.prepareBinaryData(plaintext, safeName) }
}];
```

Two things that bite:
- Request **bytes only** (`encoding: 'arraybuffer'`, no `returnFullResponse`). Returning the full HTTP response object causes a "Converting circular structure to JSON" error on the next node.
- The `cdn_url` **expires** (note the expiry parameter in the query string). Download promptly — don't let files sit for hours before processing.

### Store and link

Upload the decrypted binary to object storage, namespaced by user:

```bash
curl --location --request POST \
  '<SUPABASE_INTERNAL_HOST>/storage/v1/object/<bucket>/{{ $json.flow_token }}/{{ $json.safe_name }}' \
--header 'apikey: <SERVICE_ROLE_KEY>' \
--header 'Authorization: Bearer <SERVICE_ROLE_KEY>'
# Body: n8n Binary File, Input Data Field Name = data
```

Then write the storage path **back into the user's row** so the response document and its files live in one record. Reference the decrypt node explicitly (the upload node's output only contains the storage key/id, not your fields):

```sql
CREATE OR REPLACE FUNCTION <schema>.save_file_ref(p_token text, p_screen text, p_field text, p_path text)
RETURNS json LANGUAGE plpgsql SECURITY DEFINER AS $$
BEGIN
  UPDATE <schema>.flow_responses
  SET data = jsonb_set(COALESCE(data,'{}'::jsonb), ARRAY[p_screen, p_field], to_jsonb(p_path), true),
      updated_at = now()
  WHERE flow_token = p_token;
  RETURN json_build_object('status','ok');
END $$;
```

Now `data.<screen>.<field>` holds the storage path, sitting right beside the text fields of that screen.

---

## Step 6 — Detect Completion and Trigger Downstream Work

Make the **last screen submittable only after the rest is filled** (Flow validation handles this). Then "last screen submitted" cleanly means "done" — no need to count screens, and adding screens later doesn't break the logic.

The final submission fires the `complete` action. Unlike `data_exchange`, this arrives as a normal WhatsApp message (`interactive.nfm_reply`) at your **message webhook**, not at the encryption endpoint. That is where you call `mark_completed(phone)` and kick off downstream processing (generate a document, notify a reviewer, sync to a CRM) **only when `completed = true`** — so the artifact is built once, from the full submission, not on every intermediate screen.

---

## Step 7 — Gatekeeping Inbound Messages

If the same number receives both Flow templates and free-text messages, guard who gets what. On an inbound message, look the sender up by phone and branch:

```bash
curl --location \
  '<SUPABASE_INTERNAL_HOST>/rest/v1/flow_responses?flow_token=eq.{{ $json.messages[0].from }}&select=flow_token,completed,last_screen' \
--header 'apikey: <SERVICE_ROLE_KEY>' \
--header 'Authorization: Bearer <SERVICE_ROLE_KEY>' \
--header 'Accept-Profile: <schema>'
```

- Returns a row → known contact → continue (e.g. allow a "resend my form" request).
- Returns `[]` → unknown → send a polite rejection / redirect message.

In the IF node, test `flow_token` **is not empty** rather than array length — it works whether or not n8n split the response into items.

---

## Security & Networking Notes

**Never store the private key in plaintext.** The endpoint needs it at runtime to decrypt, but it should not sit readable in a table or in logs. Use your database's secret vault: store the PEM encrypted, keep only a reference in your table, and expose it through a `SECURITY DEFINER` getter:

```sql
-- writer: encrypts into the vault, stores only a reference
CREATE OR REPLACE FUNCTION <schema>.store_flow_key(p_phone text, p_public text, p_private text)
RETURNS json LANGUAGE plpgsql SECURITY DEFINER AS $$
DECLARE v_name text := 'flow_key_' || p_phone; v_id uuid;
BEGIN
  SELECT id INTO v_id FROM vault.secrets WHERE name = v_name;
  IF v_id IS NULL THEN PERFORM vault.create_secret(p_private, v_name, 'flow key');
  ELSE PERFORM vault.update_secret(v_id, p_private, v_name, 'flow key'); END IF;
  INSERT INTO <schema>.flow_keys (phone_number_id, public_key, secret_name)
  VALUES (p_phone, p_public, v_name)
  ON CONFLICT (phone_number_id) DO UPDATE
    SET public_key = EXCLUDED.public_key, secret_name = EXCLUDED.secret_name;
  RETURN json_build_object('status','stored');
END $$;

-- reader: returns the decrypted PEM
CREATE OR REPLACE FUNCTION <schema>.get_flow_key(p_phone text)
RETURNS text LANGUAGE sql SECURITY DEFINER AS $$
  SELECT decrypted_secret FROM vault.decrypted_secrets WHERE name = 'flow_key_' || p_phone;
$$;
```

A reference table over the vault also means a new flow on a **new phone number** can generate, register, and store its key automatically — without ever hard-coding a key in a workflow. (Remember: the RSA key pair is **per phone number**, shared by all flows on that number — never regenerate per flow.)

**Reach your database internally.** If the database sits behind a reverse proxy with access control (e.g. a zero-trust gateway), an external server's request gets intercepted by the login page and your API key never arrives — you'll see an HTML sign-in page instead of JSON. When the automation host and the database run on the same machine, point the automation at the **internal service host** (e.g. `http://<kong-container>:8000`) so traffic bypasses the gateway entirely. If they are on separate Docker networks, connect them first.

> Note also that embedding a stored image into an externally-hosted document only works if the document service can fetch it. A document service (rendering a report) is a *third party*, not you — it has no access to a gateway-protected private bucket. Keep such files inside the same ecosystem as the document, or expose a time-limited signed URL the service can reach.

---

## Constraints Recap

| Rule | Limit |
|------|-------|
| Each `data_exchange` payload | current screen only |
| `complete` payload | final screen only |
| Endpoint response time | ~5–10 s (no blocking calls) |
| File uploads | not via Graph API — download from `cdn_url`, decrypt |
| Media encryption | AES-256-CBC + trailing 10-byte HMAC |
| `cdn_url` validity | expires — process promptly |
| `flow_token` | your user key; keep delimiters out of keys/paths |
| Key pair scope | per phone number, never per flow |

---

## Summary Flow

```
TEMPLATE SEND        flow_token = phone|name
        │
ENDPOINT (per screen)
  decrypt → respond immediately
          ↘ persist branch:
              save_flow_data(phone, screen, fields)        ← always
              if file:  download → decrypt → store → save_file_ref
        │
COMPLETE (last screen) → message webhook → mark_completed(phone) → build artifact from full row
        │
INBOUND MESSAGE → lookup phone → known? continue : reject
```

Build the data model and RPCs first, then the parallel persistence branch, then media, then completion. Each piece is independently testable.

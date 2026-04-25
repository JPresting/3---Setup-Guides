# WhatsApp Business Profile Manager

A reliable, API-driven approach to managing **WhatsApp Business Profile** information across multiple Business Portfolios — including the profile fields the official Meta Business Manager UI handles inconsistently or fails on entirely.

---

## Table of Contents

- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Editable Profile Fields](#editable-profile-fields)
- [Architecture](#architecture)
- [Token Setup](#token-setup)
- [Database Schema](#database-schema)
- [Token Refresh Job](#token-refresh-job)
- [Reading the Current Profile](#reading-the-current-profile)
- [Updating Profile Fields](#updating-profile-fields)
- [Profile Picture Upload Chain](#profile-picture-upload-chain)
- [Recovery](#recovery)
- [Multi-Tenant Considerations](#multi-tenant-considerations)
- [References](#references)

---

## The Problem

Updating WhatsApp Business Profile information through the official Meta Business Manager UI is unreliable. Common failure modes observed in production:

- **Profile picture uploads fail silently or with cryptic errors.** The UI accepts the file, shows a spinner, and either reverts to the previous image or throws CORS errors in the browser console (`Access to XMLHttpRequest at 'https://rupload.facebook.com/...' has been blocked by CORS policy`). The picture never reaches the WhatsApp account.
- **Long-form fields (description, address) randomly truncate or reject input** with non-actionable validation messages.
- **Multi-account workflows are slow.** Switching between Business Portfolios, WABAs, and phone numbers in the UI requires repeated drilldowns. Updating the same field on multiple numbers means doing it manually for each.
- **No batch operations.** Setting the same `about` text on five numbers takes five round-trips through the UI.
- **Image format quirks are not surfaced.** Transparent PNGs render as solid black or white in WhatsApp. The UI accepts them without warning.

The Graph API path bypasses the UI entirely, surfaces real error messages, and supports clean automation across any number of phone numbers.

## The Solution

A direct integration against the Meta Graph API endpoint `POST /{PHONE_NUMBER_ID}/whatsapp_business_profile`, fronted by:

1. A **single long-lived User Access Token** with cross-Portfolio scope, stored in PostgreSQL
2. A **scheduled refresh job** that keeps the token alive indefinitely
3. **Workflows or scripts** that read the token on demand and call the API

This document describes both the API contract and a reference deployment.

---

## Editable Profile Fields

All fields are managed through one endpoint:

```
POST /{PHONE_NUMBER_ID}/whatsapp_business_profile
```

| Field | Description | Limit |
|---|---|---|
| `about` | Status text shown beneath the business name | 139 characters |
| `address` | Business address | 256 characters |
| `description` | Long business description | 512 characters |
| `email` | Contact email | 128 characters |
| `websites` | Up to two URLs | 256 characters each |
| `vertical` | Industry category (enum) | See list below |
| `profile_picture_handle` | Reference returned by the upload chain | — |

Partial updates are supported. Send only the fields being changed; omitted fields retain their existing value.

### `vertical` Enum

`UNDEFINED`, `OTHER`, `AUTO`, `BEAUTY`, `APPAREL`, `EDU`, `ENTERTAIN`, `EVENT_PLAN`, `FINANCE`, `GROCERY`, `GOVT`, `HOTEL`, `HEALTH`, `NONPROFIT`, `PROF_SERVICES`, `RETAIL`, `TRAVEL`, `RESTAURANT`, `NOT_A_BIZ`

> Once `vertical` is set to a non-empty value it cannot be cleared back to empty. It can only be changed to a different enum value.

---

## Architecture

```
┌──────────────────────────────┐
│  Operator                    │
│  (form / script / workflow)  │
└──────────────────────────────┘
              │
              │  1. Read token
              ▼
┌──────────────────────────────┐
│  PostgreSQL (Supabase)       │
│  meta_tokens                 │
└──────────────────────────────┘
              │
              │  2. Authorization: Bearer <token>
              ▼
┌──────────────────────────────┐
│  Meta Graph API              │
│  /whatsapp_business_profile  │
└──────────────────────────────┘

  Independent loop ─────────────────────────┐
  ┌──────────────────────────┐              │
  │  Refresh Job (every 30d) │ ── refreshes token
  │  fb_exchange_token       │
  └──────────────────────────┘
```

The token layer is decoupled from consumption. Any number of workflows can read the token concurrently. A single refresh job keeps it valid.

---

## Token Setup

### Why a Long-Lived User Access Token

Meta offers two backend token types:

| Token Type | Scope | Lifetime |
|---|---|---|
| **System User Token** | One Business Portfolio | Non-expiring |
| **User Access Token (long-lived)** | All Portfolios the user administrates | 60 days, renewable |

For administrating multiple Business Portfolios from one credential, the User Token is the simpler model. The 60-day lifetime is solved by a refresh job that calls `fb_exchange_token` every 30 days.

### Step 1 — Designate an Anchor App

Pick one Meta App at `developers.facebook.com` to serve as the long-term anchor.

> **The anchor app must never be deleted.** The User Token is bound to the App ID under which it was generated, and refresh requires the corresponding `client_id` and `client_secret`. Deletion permanently invalidates the token.

Required values:

- `<APP_ID>` — App Settings → Basic
- `<APP_SECRET>` — same screen, requires re-auth to reveal

### Step 2 — Generate the Initial Token

In the [Graph API Explorer](https://developers.facebook.com/tools/explorer/):

- **Meta App:** the anchor app
- **User or Page:** User Token
- **Permissions:**
  - `business_management`
  - `whatsapp_business_management`
  - `whatsapp_business_messaging`

Click **Generate Access Token**, confirm in the Facebook login popup, and copy the resulting short-lived token.

### Step 3 — Exchange for Long-Lived Token

```bash
curl -G "https://graph.facebook.com/v22.0/oauth/access_token" \
  --data-urlencode "grant_type=fb_exchange_token" \
  --data-urlencode "client_id=<APP_ID>" \
  --data-urlencode "client_secret=<APP_SECRET>" \
  --data-urlencode "fb_exchange_token=<SHORT_LIVED_TOKEN>"
```

Response:

```json
{
  "access_token": "EAA...",
  "token_type": "bearer",
  "expires_in": 5184000
}
```

`5184000` seconds = 60 days, the maximum Meta provides.

### Step 4 — Verify

```bash
curl -G "https://graph.facebook.com/v22.0/me/businesses" \
  -H "Authorization: Bearer <LONG_LIVED_TOKEN>" \
  --data-urlencode "fields=id,name"
```

Returns all Business Portfolios the Facebook user administrates.

---

## Database Schema

### Schema Setup

```sql
CREATE SCHEMA IF NOT EXISTS <config_schema>;

GRANT USAGE ON SCHEMA <config_schema>
  TO postgres, anon, authenticated, service_role;

GRANT ALL ON ALL TABLES IN SCHEMA <config_schema>
  TO postgres, anon, authenticated, service_role;

GRANT ALL ON ALL SEQUENCES IN SCHEMA <config_schema>
  TO postgres, anon, authenticated, service_role;

ALTER DEFAULT PRIVILEGES IN SCHEMA <config_schema>
  GRANT ALL ON TABLES TO postgres, anon, authenticated, service_role;

ALTER DEFAULT PRIVILEGES IN SCHEMA <config_schema>
  GRANT ALL ON SEQUENCES TO postgres, anon, authenticated, service_role;
```

### Table

```sql
CREATE TABLE <config_schema>.meta_tokens (
  id           serial PRIMARY KEY,
  name         text UNIQUE NOT NULL,
  access_token text NOT NULL,
  expires_at   timestamptz NOT NULL,
  updated_at   timestamptz DEFAULT now()
);
```

### Initial Insert

```sql
INSERT INTO <config_schema>.meta_tokens (name, access_token, expires_at)
VALUES (
  '<token_identifier>',
  '<LONG_LIVED_TOKEN>',
  now() + interval '60 days'
);
```

### PostgREST Exposure

For Supabase deployments, add `<config_schema>` to the `PGRST_DB_SCHEMAS` environment variable and restart the stack.

### Encryption (Optional)

For internal admin use, plain-text storage behind a hardened service role is acceptable. For multi-tenant production deployments, encrypt the `access_token` column using `pgsodium` or store in Supabase Vault.

---

## Token Refresh Job

Run every **30 days** to maintain a 30-day buffer before expiry.

### Reference Implementation (Bash + cron)

```bash
#!/bin/bash
set -euo pipefail

SUPABASE_URL="https://<supabase_host>"
SERVICE_KEY="<SUPABASE_SERVICE_ROLE_KEY>"
APP_ID="<APP_ID>"
APP_SECRET="<APP_SECRET>"
TOKEN_NAME="<token_identifier>"
SCHEMA="<config_schema>"

CURRENT_TOKEN=$(curl -s -X GET \
  "$SUPABASE_URL/rest/v1/meta_tokens?name=eq.$TOKEN_NAME&select=access_token" \
  -H "apikey: $SERVICE_KEY" \
  -H "Authorization: Bearer $SERVICE_KEY" \
  -H "Accept-Profile: $SCHEMA" \
  | jq -r '.[0].access_token')

NEW_TOKEN=$(curl -sG "https://graph.facebook.com/v22.0/oauth/access_token" \
  --data-urlencode "grant_type=fb_exchange_token" \
  --data-urlencode "client_id=$APP_ID" \
  --data-urlencode "client_secret=$APP_SECRET" \
  --data-urlencode "fb_exchange_token=$CURRENT_TOKEN" \
  | jq -r '.access_token')

if [[ -z "$NEW_TOKEN" || "$NEW_TOKEN" == "null" ]]; then
  echo "[$(date)] FAIL: token exchange returned no value"
  exit 1
fi

curl -s -X PATCH \
  "$SUPABASE_URL/rest/v1/meta_tokens?name=eq.$TOKEN_NAME" \
  -H "apikey: $SERVICE_KEY" \
  -H "Authorization: Bearer $SERVICE_KEY" \
  -H "Content-Profile: $SCHEMA" \
  -H "Content-Type: application/json" \
  -d "{
    \"access_token\": \"$NEW_TOKEN\",
    \"expires_at\": \"$(date -u -d '+60 days' +%Y-%m-%dT%H:%M:%SZ)\",
    \"updated_at\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
  }" > /dev/null

echo "[$(date)] OK: token refreshed"
```

```
0 3 1 * * /opt/scripts/refresh_meta_token.sh >> /var/log/meta-token.log 2>&1
```

### Reference Implementation (n8n)

```
Schedule Trigger (every 30 days)
    │
    ▼
Supabase: SELECT access_token FROM meta_tokens WHERE name = '<token_identifier>'
    │
    ▼
HTTP GET /v22.0/oauth/access_token
    grant_type        = fb_exchange_token
    client_id         = <APP_ID>
    client_secret     = <APP_SECRET>
    fb_exchange_token = <current_token>
    │
    ▼
Supabase: UPDATE meta_tokens
   SET access_token = <new_token>,
       expires_at   = now() + interval '60 days',
       updated_at   = now()
 WHERE name = '<token_identifier>'
```

### Failure Handling

Send notifications on failure (Telegram, Slack, email). A failed refresh leaves approximately 30 days to intervene before the token expires. An expired token requires the [Recovery](#recovery) procedure.

---

## Reading the Current Profile

Always read first before updating, both to confirm access and to provide UI defaults if surfacing the editor to an operator.

```bash
curl -G "https://graph.facebook.com/v22.0/<PHONE_NUMBER_ID>/whatsapp_business_profile" \
  -H "Authorization: Bearer <TOKEN>" \
  --data-urlencode "fields=about,address,description,email,profile_picture_url,websites,vertical"
```

Response shape:

```json
{
  "data": [
    {
      "messaging_product": "whatsapp",
      "about": "...",
      "address": "...",
      "description": "...",
      "email": "...",
      "profile_picture_url": "https://pps.whatsapp.net/...",
      "websites": ["https://..."],
      "vertical": "PROF_SERVICES"
    }
  ]
}
```

> Note the response is wrapped in a `data` array of length 1, not a flat object.

---

## Updating Profile Fields

```bash
curl -X POST "https://graph.facebook.com/v22.0/<PHONE_NUMBER_ID>/whatsapp_business_profile" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "about": "Automating Efficiency",
    "description": "...",
    "email": "info@example.com",
    "websites": ["https://example.com"],
    "vertical": "PROF_SERVICES"
  }'
```

Response:

```json
{ "success": true }
```

The `messaging_product` field is mandatory on every request.

To update only specific fields, omit the others:

```bash
curl -X POST "https://graph.facebook.com/v22.0/<PHONE_NUMBER_ID>/whatsapp_business_profile" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "email": "newaddress@example.com"
  }'
```

### Error Examples

**Field too long:**

```json
{
  "error": {
    "message": "(#100) Param description must be a maximum of 512 characters",
    "type": "OAuthException",
    "code": 100
  }
}
```

**Invalid vertical value:**

```json
{
  "error": {
    "message": "(#100) Param vertical does not match any valid value",
    "type": "OAuthException",
    "code": 100
  }
}
```

**Invalid token:**

```json
{
  "error": {
    "message": "Invalid OAuth access token",
    "type": "OAuthException",
    "code": 190
  }
}
```

---

## Profile Picture Upload Chain

The profile picture is the field that fails most consistently in the Business Manager UI. The Graph API requires three sequential calls but works reliably.

### Call 1 — Create Upload Session

```bash
curl -X POST "https://graph.facebook.com/v22.0/<APP_ID>/uploads" \
  -H "Authorization: Bearer <TOKEN>" \
  --data-urlencode "file_name=profile.jpg" \
  --data-urlencode "file_length=<BYTES>" \
  --data-urlencode "file_type=image/jpeg"
```

Response:

```json
{ "id": "upload:<SESSION_ID>" }
```

`<APP_ID>` must match the anchor app from the token setup.

### Call 2 — Upload Binary

> The authorization scheme differs from every other call: `OAuth`, not `Bearer`.

```bash
curl -X POST "https://graph.facebook.com/v22.0/<SESSION_ID>" \
  -H "Authorization: OAuth <TOKEN>" \
  -H "file_offset: 0" \
  --data-binary "@/path/to/profile.jpg"
```

Response:

```json
{ "h": "<HANDLE>" }
```

### Call 3 — Attach Handle to Profile

```bash
curl -X POST "https://graph.facebook.com/v22.0/<PHONE_NUMBER_ID>/whatsapp_business_profile" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "profile_picture_handle": "<HANDLE>"
  }'
```

Response:

```json
{ "success": true }
```

The handle can be combined with other field updates in the same request:

```bash
curl -X POST "https://graph.facebook.com/v22.0/<PHONE_NUMBER_ID>/whatsapp_business_profile" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "profile_picture_handle": "<HANDLE>",
    "about": "Automating Efficiency",
    "email": "info@example.com"
  }'
```

### Image Requirements

| Constraint | Value |
|---|---|
| Format | JPEG or PNG |
| Aspect ratio | Square |
| Resolution | 640×640 px recommended, max 1024×1024 |
| File size | Under 5 MB |
| Transparency | Not supported — alpha channel renders as solid white or black |

> Convert PNG to JPEG before upload to avoid transparency rendering issues. The Graph API accepts PNG without warning, but the rendered result on devices is unpredictable.

---

## Recovery

If the token has died — refresh job failed for over 60 days, anchor app deleted, Facebook password changed:

1. Open the Graph API Explorer with the **same anchor app** as the original setup
2. Generate a new short-lived User Token with the same permission set
3. Run the long-lived exchange (`fb_exchange_token`)
4. `UPDATE` the `meta_tokens` row with the new token and reset `expires_at`
5. Re-enable the refresh job

If the anchor app was deleted, a full re-setup with a different anchor app is required, including updating any workflows that reference the old App ID (notably the upload chain).

---

## Multi-Tenant Considerations

For customer-facing products, the single-token model does not apply. Each customer must own their own token, acquired through Meta's Embedded Signup OAuth flow.

| Aspect | Internal Admin (this document) | SaaS Product |
|---|---|---|
| Token count | One | One per customer |
| Token acquisition | Manual (Graph API Explorer) | Automated (Embedded Signup OAuth) |
| Token ownership | The administrator | The customer |
| App Review | Not required | Required for `whatsapp_business_management`, `whatsapp_business_messaging`, `business_management` |
| Business Verification | Not required | Required |
| Refresh logic | Single row | Iteration over all customer tokens |
| Storage encryption | Optional | Mandatory |

### Suggested Multi-Tenant Schema

```sql
CREATE TABLE <service_schema>.customer_meta_connections (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id             uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  business_id         text NOT NULL,
  business_name       text,
  waba_id             text,
  phone_number_id     text,
  access_token        text NOT NULL,
  token_expires_at    timestamptz NOT NULL,
  scopes              text[],
  reconnect_required  boolean DEFAULT false,
  created_at          timestamptz DEFAULT now(),
  updated_at          timestamptz DEFAULT now(),
  UNIQUE (user_id, business_id)
);
```

The refresh job iterates over `customer_meta_connections`, refreshes each token, and sets `reconnect_required = true` on individual failures. The application UI surfaces a "Reconnect WhatsApp" prompt when this flag is set.

---

## References

- [Meta Graph API](https://developers.facebook.com/docs/graph-api)
- [WhatsApp Business Profile API](https://developers.facebook.com/docs/whatsapp/cloud-api/reference/business-profiles)
- [Resumable Upload API](https://developers.facebook.com/docs/graph-api/guides/upload)
- [Long-Lived Access Tokens](https://developers.facebook.com/docs/facebook-login/guides/access-tokens/get-long-lived)
- [Embedded Signup](https://developers.facebook.com/docs/whatsapp/embedded-signup)

---

## License

Internal documentation. Adapt and redistribute as needed within your organization.

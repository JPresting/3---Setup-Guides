# WhatsApp Business Profile Manager

A reliable, API-driven approach to managing **WhatsApp Business Profile** information across multiple Business Portfolios — bypassing the Meta Business Manager UI which handles certain profile operations inconsistently or fails on them entirely.

This document covers the **architecture, one-time manual setup, and database layout**. The operational flow (form, token routing, API calls, upload chain) is implemented in the n8n workflow `Change WhatsApp WABA Profile Info`. Refer to that workflow for runtime details.

---

## Table of Contents

- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Token Architecture](#token-architecture)
- [One-Time Setup](#one-time-setup)
  - [Part A — User Token (Portfolio Discovery)](#part-a--user-token-portfolio-discovery)
  - [Part B — System User Token per Portfolio](#part-b--system-user-token-per-portfolio)
- [Database Schema](#database-schema)
- [Operational Flow](#operational-flow)
- [API Reference](#api-reference)
- [Recovery](#recovery)
- [Multi-Tenant Considerations](#multi-tenant-considerations)
- [References](#references)

---

## The Problem

Updating WhatsApp Business Profile information through the Meta Business Manager UI is unreliable:

- **Profile picture uploads fail silently or with CORS errors.** The UI accepts the file but the picture never reaches the WhatsApp account.
- **Long-form fields (description, address) randomly truncate** with non-actionable validation messages.
- **Multi-account workflows are slow.** Every Portfolio → WABA → phone number drilldown is manual.
- **No batch operations.** Setting the same `about` text on five numbers takes five UI round-trips.
- **Image format quirks are not surfaced.** Transparent PNGs render as solid black or white. The UI accepts them without warning.

The Graph API path bypasses the UI entirely, surfaces real error messages, and supports clean automation.

---

## The Solution

A direct integration against `POST /{PHONE_NUMBER_ID}/whatsapp_business_profile`, fronted by:

1. A **hybrid token architecture** — one Long-Lived User Token for cross-Portfolio discovery, plus one permanent Admin System User Token per Business Portfolio for WABA-level operations
2. A **scheduled refresh job** that keeps the User Token alive
3. An **n8n workflow** that selects the correct token by Portfolio and orchestrates reads, writes, and the profile picture upload chain

---

## Token Architecture

A naive single-token design breaks: User Tokens carry a frozen `granular_scopes` snapshot of WABA IDs at generation time. New WABAs added to a Portfolio after the token was issued never appear in it, even after `fb_exchange_token` refreshes — refresh extends lifetime, not scope.

The fix is two-tier:

| Token Type | Stored As | Used For | Lifetime | Sees Future WABAs? |
|---|---|---|---|---|
| **Long-Lived User Token** | `<user_token>` | `GET /me/businesses` (Portfolio listing) | 60 days, refreshed every 30 | N/A — only enumerates Portfolios |
| **Admin System User Token** (one per Portfolio) | `system_token_<portfolio>` | All WABA-level reads and writes | Never expires | **Yes** — automatic |

### Why both are needed

System User Tokens cannot answer `GET /me/businesses`. A System User exists *inside* a single Business Portfolio and has no cross-Portfolio scope. Only a User Token attached to the administrator's Facebook account can enumerate the Portfolios that account administrates.

Conversely, the User Token's frozen WABA snapshot makes it unreliable for listing or operating on WABAs. The same `GET /{BUSINESS_ID}/owned_whatsapp_business_accounts` call returns the full current-and-future list when made with a Portfolio's Admin System User Token.

### Why Admin System Users see future WABAs

Per Meta documentation: *"By default, admin system users have full access to all WABAs and their assets owned by or shared with you or your business portfolio … without having to manually grant business asset access to each asset whenever it is created, or shared with your business portfolio."*

Mechanism: System User Tokens are not scoped lists. The token references the System User identity. On every API call Meta evaluates the System User's current asset access live. New WABAs created in or shared with the Portfolio fall under the Admin's default coverage immediately.

---

## One-Time Setup

### Part A — User Token (Portfolio Discovery)

This token answers a single question: which Business Portfolios the administrator's Facebook account administrates. Generated once, refreshed forever.

#### A.1 — Designate an anchor app

Pick one Meta App at `developers.facebook.com` to serve as the anchor for the User Token.

> **The anchor app must never be deleted.** The User Token is bound to the App ID under which it was generated, and refresh requires the corresponding `client_id` and `client_secret`. Deletion permanently invalidates the token.

#### A.2 — Generate short-lived token

In the [Graph API Explorer](https://developers.facebook.com/tools/explorer/):

- **Meta App:** the anchor app
- **User or Page:** User Token
- **Permissions:** `business_management`, `whatsapp_business_management`, `whatsapp_business_messaging`

Click **Generate Access Token**, confirm in the OAuth popup, copy the result.

#### A.3 — Exchange for long-lived

```bash
curl -G "https://graph.facebook.com/v22.0/oauth/access_token" \
  --data-urlencode "grant_type=fb_exchange_token" \
  --data-urlencode "client_id=<USER_TOKEN_APP_ID>" \
  --data-urlencode "client_secret=<USER_TOKEN_APP_SECRET>" \
  --data-urlencode "fb_exchange_token=<SHORT_LIVED_TOKEN>"
```

`expires_in: 5184000` = 60 days, the maximum.

#### A.4 — Verify

```bash
curl -G "https://graph.facebook.com/v22.0/me/businesses" \
  -H "Authorization: Bearer <LONG_LIVED_TOKEN>" \
  --data-urlencode "fields=id,name"
```

Should list all Portfolios the administrator's account administrates.

---

### Part B — System User Token per Portfolio

Repeat this section once per Business Portfolio. Each Portfolio needs its own System User and its own permanent token.

#### B.1 — Designate an anchor app per Portfolio

Each Portfolio has at least one Meta App associated with it (Business Settings → Accounts → Apps). Pick one as the System User Token's anchor.

> **This anchor app must never be deleted either.** Generated System User Tokens stay valid only as long as their generating app exists.

#### B.2 — Create or identify the System User

`business.facebook.com` → top-left dropdown → select Portfolio → Settings → Users → **System Users**

If an Admin System User already exists, reuse it. Otherwise click **Add**:

- Name: descriptive (e.g. `<portfolio>-api-bot`)
- Role: **Admin** — required for automatic future-WABA coverage
- Click **Create System User**

#### B.3 — Assign the anchor app to the System User

This is the step that fails silently in the Meta UI when skipped. The **Generate New Token** button will show *"No permissions available — Assign an app role to the system user or select another app to continue"* with no further explanation.

1. Click the System User → right-hand detail panel
2. **Add Assets**
3. Asset type column: **Apps**
4. Asset selection column: check the Portfolio's anchor app
5. Permissions column → **Full Control** → toggle **Manage app** to ON
6. **Assign Assets**
7. Reload — permission propagation takes a few seconds

#### B.4 — Verify WABA coverage

In the System User detail panel, **Assigned Assets** tab → **WhatsApp accounts** section. Confirm all current WABAs of the Portfolio are listed. If any are missing, the Admin's default coverage has been overridden somewhere — fix at the WABA level or assign the missing WABAs explicitly (Add Assets → WhatsApp accounts → Full Control).

#### B.5 — Generate the token

In the System User detail panel → **Generate New Token**:

- **App:** the anchor app from B.1
- **Expiration:** **Never**
- **Permissions:** `whatsapp_business_management`, `whatsapp_business_messaging`, `business_management`

Click **Generate Token**. **Copy immediately** — shown only once.

#### B.6 — Smoke test

```bash
curl -G "https://graph.facebook.com/v22.0/<PORTFOLIO_ID>/owned_whatsapp_business_accounts" \
  -H "Authorization: Bearer <SYSTEM_USER_TOKEN>" \
  --data-urlencode "fields=id,name" \
  --data-urlencode "limit=200"
```

All Portfolio WABAs should appear. If the count differs from the Business Manager UI, return to B.4 and reconcile asset access before storing the token.

---

## Database Schema

### Schema and grants

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
  id              serial PRIMARY KEY,
  name            text UNIQUE NOT NULL,
  access_token    text,                       -- nullable: placeholder rows for Portfolios without WABAs yet
  expires_at      timestamptz,                -- nullable: System User tokens never expire
  updated_at      timestamptz DEFAULT now(),
  portfolio_id    text,                       -- Meta Business Portfolio ID
  portfolio_name  text,                       -- human-readable Portfolio name
  app_name        text                        -- anchor app — see comment
);

COMMENT ON COLUMN <config_schema>.meta_tokens.app_name
  IS 'Meta App that generated this token — must never be deleted, or the token dies';
```

Both `access_token` and `expires_at` are nullable:

- `expires_at IS NULL` → permanent token (System User), no refresh needed
- `access_token IS NULL` → Portfolio reserved for a future WABA that does not exist yet (e.g. number being provisioned). Row holds Portfolio metadata; token added once available.

### Initial inserts

```sql
-- User Token
INSERT INTO <config_schema>.meta_tokens
  (name, access_token, portfolio_id, portfolio_name, app_name, expires_at, updated_at)
VALUES (
  '<user_token_name>',
  '<LONG_LIVED_USER_TOKEN>',
  NULL,
  'ALL (User Token spans portfolios)',
  '<USER_TOKEN_APP_NAME>',
  now() + interval '60 days',
  now()
);

-- One row per Portfolio (System User Tokens)
INSERT INTO <config_schema>.meta_tokens
  (name, access_token, portfolio_id, portfolio_name, app_name, expires_at, updated_at)
VALUES
  ('system_token_<portfolio_a>', '<TOKEN_A>', '<ID_A>', '<Name A>', '<App A>', NULL, now()),
  ('system_token_<portfolio_b>', '<TOKEN_B>', '<ID_B>', '<Name B>', '<App B>', NULL, now());

-- Placeholder row (Portfolio without WABA yet)
INSERT INTO <config_schema>.meta_tokens
  (name, access_token, portfolio_id, portfolio_name, app_name, expires_at, updated_at)
VALUES
  ('system_token_<portfolio_pending>',
   NULL,
   '<PENDING_ID>', '<Pending Name>', '<Anchor App>', NULL, now());
```

When the pending Portfolio gets its first WABA, generate the System User Token (Part B) and update:

```sql
UPDATE <config_schema>.meta_tokens
SET access_token = '<NEW_TOKEN>', updated_at = now()
WHERE name = 'system_token_<portfolio_pending>';
```

### PostgREST exposure

For Supabase deployments, add `<config_schema>` to the `PGRST_DB_SCHEMAS` environment variable and restart the stack.

### Encryption (optional)

Plain-text storage behind a hardened service role is acceptable for internal admin use. For multi-tenant deployments encrypt `access_token` with `pgsodium` or store in Supabase Vault.

---

## Operational Flow

The runtime logic — form-based Portfolio/WABA/phone selection, token routing, profile read/write, profile picture upload chain — is implemented in n8n:

- **`Change WhatsApp WABA Profile Info`** — interactive form for editing any profile field on any phone number across all Portfolios. Pulls the User Token to list Portfolios, swaps in the matching System User Token after Portfolio selection, and uses that for every subsequent WABA-level call.
- **`WhatsApp Keep Token Online`** — scheduled refresh of the User Token only. System User Tokens are permanent and not touched by this job.

High-level flow:

```
Form submit
  → refresh User Token (fb_exchange_token)
  → list Portfolios via /me/businesses (User Token)
  → operator picks Portfolio
  → look up matching system_token by portfolio_id
  → list WABAs / phone numbers / current profile (System Token)
  → operator edits fields, optionally uploads new picture
  → POST /whatsapp_business_profile (System Token)
```

For node-by-node configuration, see the workflow JSON.

---

## API Reference

For ad-hoc scripting outside n8n, the relevant endpoints:

### Editable fields

```
POST /v22.0/{PHONE_NUMBER_ID}/whatsapp_business_profile
```

| Field | Limit |
|---|---|
| `about` | 139 chars |
| `address` | 256 chars |
| `description` | 512 chars |
| `email` | 128 chars |
| `websites` | 2 entries, 256 chars each |
| `vertical` | enum (see below) |
| `profile_picture_handle` | from upload chain |

`messaging_product: "whatsapp"` is mandatory on every request. Partial updates are supported — omitted fields retain existing values.

`vertical` enum: `UNDEFINED`, `OTHER`, `AUTO`, `BEAUTY`, `APPAREL`, `EDU`, `ENTERTAIN`, `EVENT_PLAN`, `FINANCE`, `GROCERY`, `GOVT`, `HOTEL`, `HEALTH`, `NONPROFIT`, `PROF_SERVICES`, `RETAIL`, `TRAVEL`, `RESTAURANT`, `NOT_A_BIZ`. Once set to a non-empty value `vertical` cannot be cleared, only changed.

### Read profile

```
GET /v22.0/{PHONE_NUMBER_ID}/whatsapp_business_profile
  ?fields=about,address,description,email,profile_picture_url,websites,vertical
```

Response wrapped in `{ "data": [ { ... } ] }` (length 1).

### Profile picture upload chain (3 calls)

1. `POST /v22.0/{APP_ID}/uploads` — create session, returns `id: upload:<SESSION_ID>`
2. `POST /v22.0/{SESSION_ID}` with `Authorization: OAuth <TOKEN>` (not `Bearer`) and binary body — returns `h: <HANDLE>`
3. `POST /v22.0/{PHONE_NUMBER_ID}/whatsapp_business_profile` with `profile_picture_handle: <HANDLE>` (combinable with other field updates)

`<APP_ID>` in step 1 must match the anchor app of the System User Token used for authorization.

### Image requirements

| Constraint | Value |
|---|---|
| Format | JPEG or PNG |
| Aspect ratio | Square |
| Resolution | 640×640 recommended, max 1024×1024 |
| File size | < 5 MB |
| Transparency | Not supported — alpha channel renders as solid white or black |

> Convert PNG to JPEG before upload. The Graph API accepts PNG without warning, but the rendered result on devices is unpredictable.

### Common errors

`(#100) Param description must be a maximum of 512 characters` — field too long
`(#100) Param vertical does not match any valid value` — invalid enum
`Invalid OAuth access token` (code 190) — token died or wrong token type for endpoint

---

## Recovery

### User Token died

Refresh job failed for over 60 days, anchor app deleted, FB password changed:

1. Graph API Explorer with the **same anchor app** as original setup
2. Generate new short-lived User Token, same permissions
3. `fb_exchange_token` → long-lived
4. `UPDATE` the User Token row, reset `expires_at`
5. Re-enable refresh job

If the anchor app was deleted, full re-setup with a new anchor app, including updating the refresh job's `client_id` and `client_secret`.

### System User Token died

System User Tokens don't expire on their own but die if their generating app is deleted, the System User is deleted, or the generating Admin loses Portfolio access.

1. Verify the anchor app is still listed under the System User's Assigned Assets
2. **Generate New Token** → repeat B.5/B.6
3. `UPDATE` the corresponding `system_token_<portfolio>` row

If the System User itself was deleted, recreate following Part B from B.2.

If the upload-chain anchor app was deleted, every `POST /{APP_ID}/uploads` call will fail. Switch to a different valid app under the same Portfolio and update both the workflow URL and the System User's asset assignment.

---

## Multi-Tenant Considerations

The single-administrator model does not apply to customer-facing products. Each customer must own their own token via Embedded Signup OAuth.

| Aspect | Internal Admin (this doc) | SaaS Product |
|---|---|---|
| Token count | 1 User Token + 1 System Token per Portfolio | 1 Business Integration System User Token per customer |
| Acquisition | Manual | Automated (Embedded Signup) |
| Ownership | Administrator | Customer |
| App Review | Not required | Required |
| Business Verification | Not required | Required |
| Refresh logic | Single User Token row | Per-customer iteration |
| Storage encryption | Optional | Mandatory |

### Suggested multi-tenant schema

```sql
CREATE TABLE <service_schema>.customer_meta_connections (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id             uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  business_id         text NOT NULL,
  business_name       text,
  waba_id             text,
  phone_number_id     text,
  access_token        text NOT NULL,
  token_expires_at    timestamptz,
  scopes              text[],
  reconnect_required  boolean DEFAULT false,
  created_at          timestamptz DEFAULT now(),
  updated_at          timestamptz DEFAULT now(),
  UNIQUE (user_id, business_id)
);
```

Refresh job iterates over `customer_meta_connections`, refreshes each token where applicable, sets `reconnect_required = true` on failures. UI surfaces a "Reconnect WhatsApp" prompt when this flag is set.

---

## References

- [Meta Graph API](https://developers.facebook.com/docs/graph-api)
- [WhatsApp Business Profile API](https://developers.facebook.com/docs/whatsapp/cloud-api/reference/business-profiles)
- [Resumable Upload API](https://developers.facebook.com/docs/graph-api/guides/upload)
- [Long-Lived Access Tokens](https://developers.facebook.com/docs/facebook-login/guides/access-tokens/get-long-lived)
- [System User Access Token (WhatsApp)](https://developers.facebook.com/documentation/business-messaging/whatsapp/access-tokens/)
- [Embedded Signup](https://developers.facebook.com/docs/whatsapp/embedded-signup)

---

## License

Internal documentation. Adapt and redistribute as needed within your organization.
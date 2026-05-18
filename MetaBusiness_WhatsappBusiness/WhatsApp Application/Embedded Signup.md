# WhatsApp Embedded Signup — Setup & Flow

How a WhatsApp Embedded Signup integration is wired end to end, in the order you actually have to do it: the foundational Meta setup, the App configuration, the platform backend, the frontend, and what happens click-by-click in the browser.

## 1. Meta Business — foundation

Before the App can be touched, a Meta Business (Business Portfolio) has to exist and pass verification:

- **Business verification** — Meta has to confirm the legal entity behind the Business Portfolio (registration documents, domain ownership, address). Without verification the App stays in Development mode and Embedded Signup is hard-capped at internal testers only.
- **Verified business domain** — the domain serving the platform dashboard is added under Business Settings → Brand Safety → Domains and verified (DNS TXT, meta-tag, or file upload). Required for the Embedded Signup popup and for the Privacy/Terms URLs Meta surfaces.

## 2. Meta App — basics

A single Meta App represents the platform. Under App Settings → Basic, the following are filled in (Meta blocks App Review without them):

- Display Name, App Icon (1024×1024), Category
- **Privacy Policy URL** (shown in the Embedded Signup consent screen)
- **Terms of Service URL** (shown in the consent screen)
- **Data Deletion** instructions URL or callback endpoint
- App Domains (the platform's dashboard origin)
- Business Use field set to the verified Business

Graph API version pinned to `v25.0` (App Settings → Advanced).

## 3. Permissions — Advanced Access

Each permission used by Embedded Signup needs Advanced Access via App Review. Without Advanced Access the popup only works for admins/developers/testers of the App.

- `public_profile` — required for `FB.login` itself. Advanced Access is requested under App Review → Permissions and Features. This is the one most teams forget — Standard Access alone makes the SDK silently restrict the popup.
- `whatsapp_business_messaging`
- `whatsapp_business_management`
- `business_management`

Each one is submitted with a written justification + screencast showing the user-facing feature it powers.

## 4. Add Products to the App

Two products are added under the App:

### Facebook Login for Business — required gotcha

Two sub-products both have to be **enabled** (this trips most teams up):

- **WhatsApp Cloud API**
- **Marketing Messages API for WhatsApp**

With only WhatsApp Cloud API on, the Embedded Signup popup loads but the flow breaks before it returns the WABA and phone number. Both must be on.

All six toggles in the FLB settings tab are enabled — the full set is needed during onboarding (including event-sharing that the consent screen references).

### WhatsApp → Embedded Signup Builder

A branded Embedded Signup config is published under WhatsApp → Embedded Signup Builder. Each published config has its own ID, which the frontend passes to `FB.login`. The builder owns:

- Branding (logo, copy, accent colours) shown in the Meta-hosted popup.
- The list of products the user is being onboarded to (WhatsApp Cloud API + Marketing Messages API).
- Privacy Policy and Terms URLs that Meta surfaces in the popup — must match what's set in App Settings → Basic.

## 5. App Review submission

When everything above is in place, App Review is submitted with:

- A screencast video walking through the Embedded Signup flow end to end (open the popup, sign in, pick Business Portfolio + WABA + phone number, land back on the platform with the number active).
- Per-permission written descriptions naming the user-visible feature each permission powers.
- Privacy Policy URL and Terms URL.
- A test user account Meta reviewers can use to reproduce the flow.

## 6. Go Live + Tech Provider

After Meta approves the requested permissions and the App is **switched to Live mode**, the App is eligible to be listed as a Tech Provider / Solution Partner. Once that step is done, the platform appears under WhatsApp → Partner Solutions for end customers.

## Platform backend

Required env vars on the platform's server:

- `META_APP_ID` — the Meta App's App ID
- `META_APP_SECRET` — the Meta App's App Secret (used as `client_secret` for the OAuth code exchange)
- `GRAPH_VERSION` — `v25.0` (must match the SDK version on the client)

A single endpoint is exposed for the embedded-signup callback. It requires the platform's session JWT. Body: `{ code, waba_id, phone_number_id }`.

What it does:

1. Exchange the short-lived `code` for a short-lived User Access Token:
   `GET graph.facebook.com/v25.0/oauth/access_token?client_id={META_APP_ID}&client_secret={META_APP_SECRET}&code={code}`
2. Exchange the short-lived token for a long-lived (60-day) token:
   same endpoint, `grant_type=fb_exchange_token&fb_exchange_token={short_lived}`.
3. Fetch the phone number's `display_phone_number` via `GET /{phone_number_id}?fields=display_phone_number` using the long-lived token.
4. Store the long-lived token in a server-side vault and upsert a phone-account record keyed on `(owner_user_id, phone_number_id)` with `waba_id`, `display_phone_number`, and the vault reference set.

## Platform frontend

The flow lives inside the "Add WhatsApp Number" wizard, on the "Connect WhatsApp Business" path.

1. **Load the Facebook JS SDK** on demand (`https://connect.facebook.net/en_US/sdk.js`) and call `FB.init({ appId: <META_APP_ID>, cookie: true, xfbml: false, version: "v25.0" })`. The SDK is loaded the first time the user clicks the Open Onboarding CTA.
2. **Register a `message` listener** on `window` that captures `event.data.type === "WA_EMBEDDED_SIGNUP"` payloads. Meta's popup posts the chosen `waba_id` and `phone_number_id` to the parent window via this channel.
3. **Open the popup** with `FB.login(handleLoginResponse, { config_id: <EMBEDDED_SIGNUP_CONFIG_ID>, response_type: "code", override_default_response_type: true, extras: { version: "v4" } })`. Meta renders the consent screen and Embedded Signup flow in a separate window.
4. **On callback** the SDK returns `{ status: "connected", authResponse: { code } }`. The frontend combines this with the `waba_id` / `phone_number_id` it captured via postMessage and POSTs them to the platform's exchange endpoint.
5. **Server returns** the upserted account record. The wizard skips the manual Identity + Access Token steps and jumps straight to Webhook delivery, with the account already created and verified.

## End-to-end flow

1. The user clicks **Add WhatsApp Number** in the platform dashboard.
2. Wizard Step 0 shows two cards. **Connect WhatsApp Business** is the default; clicking Open Onboarding triggers the FB SDK and `FB.login` call described above.
3. The Meta popup loads at `facebook.com/v25.0/dialog/oauth?app_id={META_APP_ID}&...`. The user signs in with Meta, picks Business Portfolio → WABA → phone number, accepts permissions.
4. While the user navigates the popup, Meta posts `WA_EMBEDDED_SIGNUP` messages back to the parent window containing the selected `waba_id` and `phone_number_id`.
5. On finish, the popup closes and the SDK callback fires with `authResponse.code`.
6. The wizard POSTs `{ code, waba_id, phone_number_id }` to the platform's exchange endpoint.
7. The backend exchanges the code for a long-lived token, persists the account, and returns the record to the wizard.
8. The wizard skips Identity + Access Token steps and lands on Webhook delivery with the new number already selected.

## Checklist — what has to exist on Meta before this works

- Verified Meta Business (Business Portfolio)
- Verified business domain
- Meta App with Display Name, Icon, Category
- App Settings → Basic: Privacy Policy URL, Terms of Service URL, Data Deletion URL, App Domains, Business Use
- Advanced Access for: `public_profile`, `whatsapp_business_messaging`, `whatsapp_business_management`, `business_management`
- Facebook Login for Business product added, with **both** WhatsApp Cloud API and Marketing Messages API sub-products enabled, and all six FLB settings toggles on
- WhatsApp product added, Embedded Signup Builder config published, config ID copied for the frontend
- App Review submitted and approved with screencast video + per-permission descriptions
- App switched to Live mode
- (Optional, for visibility) Tech Provider / Solution Partner status approved

## Checklist — what has to exist on the platform server

- `META_APP_ID` and `META_APP_SECRET` env vars configured
- The code-exchange endpoint reachable from the browser (the platform dashboard origin)
- A vault or secret-store for the long-lived tokens — never stored in env vars or returned to the client

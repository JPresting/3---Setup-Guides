# WhatsApp Embedded Signup — Setup & Flow

How a WhatsApp Embedded Signup integration is wired end to end: what gets configured on Meta's side, what the platform backend needs, and what happens click-by-click in the browser.

## Meta App side

A single Meta App on Graph API `v25.0` is used. The customer-facing platform is registered as the App's brand.

### Facebook Login for Business — required gotcha

Under the Meta App → **Facebook Login for Business** product, two sub-products both have to be enabled:

- **WhatsApp Cloud API**
- **Marketing Messages API for WhatsApp**

With only WhatsApp Cloud API enabled, the Embedded Signup popup loads but the flow breaks before it returns the WABA and phone number. Both must be on.

### Settings within Facebook Login for Business

All six toggles in the FLB settings tab are enabled — the full set is needed during onboarding (including event-sharing that the consent screen references).

### WhatsApp → Embedded Signup Builder

A branded Embedded Signup config is published under WhatsApp → Embedded Signup Builder. Each published config has its own ID, which the frontend passes to `FB.login`. The builder owns:

- Branding (logo, copy, accent colours) shown in the Meta-hosted popup.
- The list of products the user is being onboarded to (WhatsApp Cloud API + Marketing Messages API).
- Privacy Policy and Terms URLs that Meta surfaces in the popup.

### Permissions requested at OAuth time

- `whatsapp_business_messaging`
- `whatsapp_business_management`
- `business_management`

### App Review submission

Meta requires for approval:

- A screencast video walking through the Embedded Signup flow end to end (open the popup, sign in, pick a Business Portfolio + WABA + phone number, land back on the platform with the number active).
- A written description of each requested permission and the user-facing feature it powers.
- The Privacy Policy URL and Terms URL (must match what's set in the Embedded Signup Builder).

Once the App is approved as a Solution Partner / Tech Provider, the **Partner Solutions** entry appears under the WhatsApp section and customers see the platform as a selectable onboarding option.

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

## Required from Meta App Review

- Demo video covering steps 2–8 of the flow above.
- A short description for each requested permission, naming the user-visible feature it powers.
- Privacy Policy URL and Terms URL matching the values in the Embedded Signup Builder.
- Verified Business Portfolio + App switched to Live mode.

## Required from the platform server

- `META_APP_ID` and `META_APP_SECRET` env vars configured.
- The code-exchange endpoint reachable from the browser (the platform dashboard origin).
- A vault or secret-store for the long-lived tokens — never stored in env vars or returned to the client.

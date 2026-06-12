# WhatsApp App Review — Submission Walkthrough

Companion to `Embedded Signup.md`. That doc covers how the integration is wired.
This one covers the part that comes after wiring: getting the App Review actually
**submitted** — the click-by-click of the submission form, and the one gotcha that
cost us a full day.

## What you actually request

For a WhatsApp-only app, only two permissions show up as requestable **New requests**:

- `whatsapp_business_messaging`
- `whatsapp_business_management`

`public_profile` shows up separately under **Existing access for renewal** — it's
already on Advanced Access from the `FB.login` setup, so it just gets re-confirmed,
no new evidence needed.

`business_management` does **not** appear as a separately requestable permission for
a WhatsApp-only app. Don't go hunting for it — Meta bundles it implicitly. Requesting
the two above is enough.

Do **not** add Instagram / Pages / Ads / Commerce / Marketing-Messages permissions.
They're irrelevant to a voice/messaging app and only make reviewers suspicious
("why does a call app want Instagram Insights?").

## The submission form has four sections

App Review → **Submit for review** shows:

- **Verification** — auto-green
- **App settings** — auto-green (Privacy/Terms/Icon/Category already set)
- **Data handling** — auto-green
- **Allowed usage** — the only one you fill in by hand

`Submit for review` stays greyed out until **Allowed usage** is fully green.

## Allowed usage — what to fill in per permission

For each of the two permissions you complete four sub-items:

1. **Describe how your app uses this permission** — Meta pre-fills an AI suggestion
   that starts "Dear App Review Team…". **Delete it** and paste your own (texts below).
2. **Upload a screencast** — one end-to-end video of the Embedded Signup flow.
   **The same video works for both permissions** — upload it twice.
3. **Perform required API test call** — see the gotcha section. This is the one that bites.
4. **Agree to comply** — checkbox.

Plus a one-line **Business Description** (same for both permissions):

> Stardawn is an AI development and process optimization company offering multiple
> SaaS platforms — including WhatsApp Business management, AI voice agents, and
> inbound call automation — serving business clients as a Tech Provider.

### Justification texts we used

**whatsapp_business_messaging**

> Stardawn uses this permission to send and receive WhatsApp messages on behalf of
> onboarded business clients, manage and submit message templates for approval, and
> subscribe to webhook events (calls, messages, template status updates) for real-time
> delivery. Each client connects their own WABA via Embedded Signup; Stardawn acts as
> a Tech Provider.

**whatsapp_business_management**

> Stardawn uses this permission to read WABA details and phone number metadata after
> Embedded Signup onboarding, manage phone number settings (AI calling toggle, call
> permissions), and subscribe the app to WABA-level webhooks. Access is strictly scoped
> per client — each business authenticates via Embedded Signup and only their own
> account data is accessed.

## The API test-call gotcha — this cost us a full day

Each permission requires **one real API call as evidence**. The sub-item reads
`0 of 1 API call(s) required` and must turn to `1 of 1` before the section goes green.

The trap: **the counter does not update in real time.** You make a valid call, it
returns `200 OK`, and the counter still reads `0/1` for hours. Meta's own text says
"can take up to 24 hours to show" — and in our case it literally took about a day.
There was nothing broken and nothing to fix. We just had to wait.

Each permission needs a **matching** kind of call:

- **whatsapp_business_messaging → a message send.**
  Easiest path: WhatsApp → **API Setup** → *Generate access token* → *Send message*.
  That fires the `hello_world` template to your own number and returns
  "Test message successfully sent."

- **whatsapp_business_management → a management *read*, NOT a message.**
  A `GET` on the WABA or its phone numbers. The *Send message* button does **not**
  satisfy this one — that's the easy mistake to make.

Commands that worked for the management call (token = any token carrying the
`whatsapp_business_management` scope — a User token from API Setup *or* a System User
admin token both work):

```bash
# WABA details
curl -X GET "https://graph.facebook.com/v25.0/{WABA_ID}?fields=name,currency,timezone_id" \
  -H "Authorization: Bearer {TOKEN}"

# phone numbers under the WABA
curl -X GET "https://graph.facebook.com/v25.0/{WABA_ID}/phone_numbers" \
  -H "Authorization: Bearer {TOKEN}"
```

Both returned real data (`200 OK`). For this app: WABA ID `2137568486642775`,
phone number ID `1054970317706222`.

### What we still don't know — and how to react if it happens again

The counter wasn't moving, so we fired the management call several times — first with
a short-lived User token from API Setup, then with a System User admin token. We
**cannot** say which one (or simply the passage of time) flipped it green. Most likely
it was just the sync delay.

So if you hit `0/1` again: confirm the call returns `200` with the right scope, then
**wait** — do not keep changing settings in a panic. Two practical notes:

- Tokens from API Setup expire in ~1 hour. A `190` / "Session has expired" error means
  exactly that — regenerate the token and re-fire.
- A System User admin token (Business Settings → Users → System Users, with the WABA
  assigned as an asset and `whatsapp_business_management` + `whatsapp_business_messaging`
  scopes) is the more reliable token if you want to rule the token out as the cause.

## Order of operations that worked

1. **Permissions page** → request Advanced Access for the two WhatsApp permissions.
2. **Record one screencast** of the full Embedded Signup flow (open onboarding → sign
   in → pick Business Portfolio + WABA + phone number → land back in the dashboard with
   the number active). ~2–3 min is enough.
3. **Submit for review → Allowed usage** → for each permission: delete Meta's AI text,
   paste the justification, upload the screencast, tick "agree".
4. **Fire the test calls** — message send for messaging, `GET` WABA for management.
5. **Leave it.** Come back the next day — counters at `1/1`, all four sections green.
6. **`Submit for review`** is now clickable. Submit.

# Reddit Bot Setup for n8n (Data API Access)

**Requirements:** Two Reddit accounts + developer registration + an approved access request

**Status as of August 2026:** Self-service creation of new Data API apps is effectively closed. The old form at `reddit.com/prefs/apps` silently rejects new applications. The only remaining path to a working OAuth credential is a manual access request that Reddit staff must approve. Everything below is required, but **none of it guarantees approval** — plan for a waiting period of days, not minutes.

## Why this guide exists

Older tutorials tell you to open `reddit.com/prefs/apps`, click "create an app", and copy your client ID and secret. That workflow no longer works. Reddit's [Responsible Builder Policy](https://support.reddithelp.com/hc/en-us/articles/42728983564564-Responsible-Builder-Policy) now states:

> Approval is required: You must request access and get explicit approval before accessing any Reddit data through our API.

In practice this means the create-app form submits, resets itself, and shows no error. This is not a browser problem, not a captcha problem, and not an account problem — it is the intended behaviour.

## Prerequisites

- A personal Reddit account with a verified email address (the "human" account)
- A second, separate Reddit account that will act as the bot (the "app account")
- A self-hosted or cloud n8n instance with a reachable HTTPS callback URL

**Important:** Reddit requires these to be two different accounts. Attempting to register your own account as the app account returns *"You can't register your own account as an app."*

---

## Step 1: Create the bot account

1. Open reddit.com in a **logged-out** browser window (or a private window) and click **Sign Up**.
2. Use an email address not yet tied to any Reddit account. Gmail sub-addressing works: `yourname+redditbot@gmail.com` delivers to your normal inbox but counts as a distinct address.
   - Note: the `+` must appear **before** the `@`. Reddit rejects a `+` after the domain.
3. Reddit assigns a random username (e.g. `Low-Building6054`). Keep it or reroll — it is not user-facing in most workflows.
4. Set a password and **record it**. You will need it in Step 3, and the bot account is not usually accessed interactively afterwards.
5. Confirm the verification email. An unverified account cannot be registered as a bot.

## Step 2: Register as a developer

Log back in with your **personal** account, then:

1. Go to **https://developers.reddit.com/app-registration**
2. Confirm the page shows your personal account name, then click **Register with \<your account\>**.
3. Accept the Developer Terms and click **Register as Developer**.

## Step 3: Register the bot account

Still logged in as your personal account, you land on the **"Register your app!"** screen:

1. **App account username:** the username from Step 1
2. **App account password:** the password from Step 1
3. Click **Continue**

If this returns *"Unable to add account. Check your information and try again."*, verify:
- You are logged in as the **personal** account, not the bot account
- The bot account's email is verified
- The bot account has no prior Data API app history (accounts that previously owned a now-deleted app are frequently rejected here)

On the **Additional details** screen, fill in:

| Field | Guidance |
|---|---|
| App name | Short, descriptive |
| What does your app do? | One paragraph, concrete. Name the subreddits and the action. |
| App public repo URL | Optional, leave blank if closed source |
| What manual actions will you take using this app's account? | Be honest. If a human reviews content before posting, say so — it materially helps. |
| Have you tried porting to the Developer Platform? | **No** (unless you actually have) |
| What's preventing you from porting? | See the Devvit section below |

Submit. A successful registration shows **"Registration submitted"** with your bot account marked **✓ Registered** and the note that account restrictions have been lifted.

**This is not yet API access.** The bot account is now labelled and permitted to act automatically, but you still have no client ID or secret.

## Step 4: Attempt app creation (will likely fail — do it anyway)

Log in **as the bot account** and go to `reddit.com/prefs/apps`:

1. Click **are you a developer? create an app...**
2. Fill in:
   - **name:** your app name
   - **type:** `web app`
   - **redirect uri:** `https://<your-n8n-domain>/rest/oauth2-credential/callback`
3. Complete the captcha and click **create app**

If the form resets and displays a link to the Responsible Builder Policy, the request was rejected. Proceed to Step 5. Document this attempt — referencing it strengthens your access request.

## Step 5: Submit an access request (the actual path)

Two channels exist. **Use both.**

### 5a. Modmail to r/Devvit (recommended — visible and human-staffed)

r/Devvit is moderated by Reddit's Developer Platform team. Unlike the ticket form, a sent modmail is verifiable in your sent messages.

**Link:** https://www.reddit.com/message/compose?to=r%2FDevvit

Template:

```
Subject: Data API app creation broken - need OAuth access for self-hosted bot

Hi,

I registered my automated account u/<BOT_ACCOUNT> via
developers.reddit.com/app-registration (labeled, restrictions lifted),
and my personal account u/<PERSONAL_ACCOUNT> as a developer.

I now need an OAuth client ID and secret for a self-hosted n8n
instance, but creating an app at reddit.com/prefs/apps fails silently:
the form resets on submit with no error message.

Devvit does not fit this use case: <one sentence, see Devvit section>.

How do I get Data API access for this? Thanks.
```

### 5b. Data Access Request ticket

**Link:** https://support.reddithelp.com/hc/en-us/requests/new?ticket_form_id=14868593862164&tf_42139884615700=api_request_type_developer_clone

Selections:
- **What do you need assistance with:** Data Access Request
- **Which role:** I'm a developer
- **What is your inquiry:** *I'm a developer and want to build a Reddit App that does not work in the Devvit ecosystem*

Then complete the free-text fields. Be specific — the form explicitly states that detail improves your chances:

- **What benefit/purpose will the bot/app have for Redditors?** Frame the value to users, not to you.
- **Detailed description of what the Bot/App will be doing:** Number the steps. Name the endpoints. State expected volume. Include a concrete worked example.
- **What is missing from Devvit:** see below
- **Link to source code or platform:** your n8n instance or repo
- **What subreddits:** an explicit, short list. Do not write "various".
- **Username operating the bot:** your bot account

**Note:** This form does not reliably send a confirmation email. Absence of a confirmation does not prove failure, but it is why 5a is recommended alongside it.

### What gets rejected — read this before writing your request

Rejections arrive as a form letter: *"the submission is not in compliance with Reddit's Responsible Builder Policy and/or lacks necessary details."* This wording is deliberately ambiguous. In practice the decisive factor is usually **what the bot does**, not how well the form was filled in.

**Use cases that are reliably rejected:**

- Posting or commenting links to your own content (YouTube, blog, product) in subreddits you do not moderate — regardless of how relevant the content is or whether a human approves each reply. Reddit classifies this as automated self-promotion.
- Lead generation, outreach, or any workflow whose output is contact with users who did not ask for it.
- Anything where the benefit accrues primarily to you rather than to the subreddit.

**Use cases with a realistic chance:**

- Read-only analysis, monitoring, dashboards, alerting.
- Moderation tooling for subreddits you moderate (but build this in Devvit — it will be approved immediately).
- Tools whose output stays outside Reddit.

**Critical:** Do not submit a second request for the same use case after a rejection. The Responsible Builder Policy explicitly prohibits *"submitting multiple requests for the same use case"* and treats it as misrepresentation. If your first request is denied, either narrow the scope genuinely — for example to read-only, with any posting done manually by a person — and raise that in the existing r/Devvit modmail thread, or accept the read-only path below.

**Design implication:** If your workflow needs to both read and write, split it. The reading half is solvable today without Reddit's permission (see below). The writing half either gets approved or gets done by hand. Plan for the second case.

---

## Why Devvit usually doesn't apply

Reddit steers all new development toward [Devvit](https://developers.reddit.com/docs), their in-platform app framework. Devvit is the right answer for some projects and impossible for others. It does **not** work if:

- **Your app must operate in subreddits you do not moderate.** Devvit apps only run in communities where you are a moderator and where the app has been installed.
- **An external system must authenticate against Reddit.** Devvit provides no OAuth client ID or secret. A self-hosted n8n, a VPS script, or any service outside Reddit cannot authenticate as a Devvit app.
- **Your orchestration lives outside Reddit.** Devvit apps run on Reddit's infrastructure.

If any of these apply, state it plainly in your request. If none apply, build the Devvit app — approval is immediate and you skip this entire process.

## Step 6: After approval — connect n8n

Once Reddit issues a client ID and secret:

1. In n8n, create a credential of type **Reddit OAuth2 API**
2. Copy the **OAuth Redirect URL** shown in the credential dialog — it must match the redirect URI registered with the app exactly
3. Paste the **Client ID** and **Client Secret**
4. Click **Connect my account**, complete the Reddit login, and confirm access
5. The credential is now usable in Reddit nodes

**Which account authorizes:** OAuth authorization does not have to be performed by the app's owner. Whichever account is logged in when you click "Connect my account" is the account the workflow will act as. For posting or commenting, authorize as the **bot account**.

---

## Interim option: reading without API access

Reading public subreddit content requires no credentials at all and works while your request is pending.

**RSS (free, no dependencies):**

```
https://www.reddit.com/r/<subreddit>/new/.rss
```

Use n8n's **RSS Read** node. Returns the newest ~25 posts with title, link, author, date and body. No score or comment counts. Note that the JSON endpoint (`/new.json`) is blocked for datacenter IPs, while RSS is not — this is why a self-hosted server can read the feed but not the JSON API.

**Commercial scrapers (paid, richer data):** Providers such as Apify offer Reddit scrapers requiring no Reddit API key, including a ready-made n8n integration. These return upvotes, comment trees and full post content, and are not subject to the 600-requests-per-10-minutes limit of the official API.

**Neither option can post or comment.** Writing to Reddit requires an authenticated account, which means an approved OAuth app. There is no third-party workaround, and browser-automation workarounds violate Reddit's terms and risk permanent account suspension.

## Rules that get bots banned

From the Responsible Builder Policy — worth reading before designing the workflow:

- **No duplicate posting across subreddits.** Posting identical or substantially similar content to multiple subreddits is classified as spam.
- **App accounts must be single-purpose.** Do not mix personal use into the bot account.
- **Apps must not manipulate votes or karma.**
- **Explicit consent is required for private messages.**
- Each target subreddit's own self-promotion rules apply independently of Reddit's global policy.

Enforcement includes token revocation, app suspension, and suspension of associated accounts — including your personal one.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `create app` form resets, no error | New app creation is closed. Use Step 5. |
| "You can't register your own account as an app" | You entered your personal account in the app account field. Use a second account. |
| "Unable to add account. Check your information" | Wrong account logged in, unverified bot email, or a bot account with prior app history. |
| Authorize URL returns "invalid client id" | The app was deleted or never existed. The stored credentials are dead; no reconnect will fix it. |
| Reddit login page shows "blocked by network security" | Reddit's bot protection triggered on the connection, not on the account. Unrelated to your credentials. |
| `/api/v1/me.json` returns no username while clearly logged in | That endpoint requires an OAuth token, not a session cookie. It is not a valid login check. |

## References

- [Responsible Builder Policy](https://support.reddithelp.com/hc/en-us/articles/42728983564564-Responsible-Builder-Policy)
- [Developer Platform & Accessing Reddit Data](https://support.reddithelp.com/hc/en-us/articles/14945211791892-Developer-Platform-Accessing-Reddit-Data)
- [App Migration Program 2026 Terms](https://support.reddithelp.com/hc/en-us/articles/47822311698452-Reddit-Developer-Platform-App-Migration-Program-2026-Terms) — registration route reserved for apps that existed before 25 March 2026
- [Devvit documentation](https://developers.reddit.com/docs)

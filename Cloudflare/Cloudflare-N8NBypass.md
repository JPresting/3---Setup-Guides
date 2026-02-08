### n8n Webhook Bypass for Cloudflare Access

**The Problem**
If you protect your self-hosted infrastructure with Cloudflare Zero Trust (Access), external services like Telegram, Stripe, or Google cannot reach your n8n webhooks. They get blocked by the Cloudflare login screen (`403 Forbidden`), causing your workflows to fail.

**The Solution**
Create a **specific application rule** in Cloudflare Access for the `/webhook` path. Cloudflare prioritizes specific path rules over general domain rules, allowing you to bypass authentication just for webhook endpoints while keeping the rest of your n8n instance secure.

---

### Setup Steps

1. Open **Cloudflare Zero Trust** → **Access** → **Applications**.
2. Click **Add an application** → **Self-hosted**.
3. **Application Configuration:**
* **Application Name:** `n8n Webhooks`
* **Domain:** `n8n` . `your-domain.com` (Enter your actual n8n domain)
* **Path:** `/webhook*` (The `*` is crucial to match all webhook IDs).


4. Click **Next**.
5. **Add a Policy:**
* **Policy Name:** `Bypass Auth`
* **Action:** `Bypass` (This disables the login requirement).
* **Assign To (Include):** Select `Everyone`.


6. Click **Next** → **Add application**.

**Result:**

* `https://n8n.your-domain.com/webhook/123...` → **Open** (External services can connect).
* `https://n8n.your-domain.com` (Dashboard) → **Protected** (You still need to login).

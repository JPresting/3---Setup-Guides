### **Setting up OAuth in N8N**  

---

## **1. Getting OAuth Client ID and Client Secret**  
1. Go to **[developers.facebook.com](https://developers.facebook.com)**.  
2. Select your app and go to **App Settings > Basic**.  
3. Copy the following:  
   - **App ID**  
   - **App Secret**  

---

## **2. Setting up OAuth in N8N**  
1. In **N8N**, go to **Credentials** and create a new **OAuth2** credential.  
2. Enter:  
   - **Client ID**: Use the App ID from Facebook.  
   - **Client Secret**: Use the App Secret from Facebook.  
3. Complete the OAuth flow and save the credential.  

---

## **3. Refreshing in Facebook Developer**  
1. After setting up the connection in N8N, go back to **[developers.facebook.com](https://developers.facebook.com)**.  
2. Refresh the page.  
3. Go to **WhatsApp > Configuration** to test the Webhook connection.  
4. Make sure to use the **Production URL** for testing.  

---

## **4. Cloudflare Tunnel / Zero Trust — Webhook Bypass (IMPORTANT)**

If your N8N instance is behind **Cloudflare Access** (Zero Trust), Meta's webhook verification will fail with **403 Forbidden**. This is because Cloudflare Access requires authentication on every request, and Meta's verification is a simple unauthenticated GET request.

### The Problem
When Meta tries to verify your webhook, it sends a GET request like:
```
GET /webhook/your-webhook-id/webhook?hub.mode=subscribe&hub.verify_token=xxx&hub.challenge=xxx
```
Cloudflare Access intercepts this, sees no session/cookie, and redirects to the login page → Meta gets 302/403 → verification fails.

### The Fix
You need to create a **Bypass policy** in Cloudflare Access that allows unauthenticated requests to both the `/webhook` and `/webhook-test` paths.

1. Go to **Cloudflare Zero Trust Dashboard** → **Access** → **Applications**
2. Find (or create) the Application that protects your N8N subdomain
3. Set the **Public hostname**:
   - **Subdomain**: your n8n subdomain (e.g. `n8n`)
   - **Domain**: your domain
   - **Path**: leave **empty** — this ensures both `/webhook` and `/webhook-test` paths are covered
4. Under **Access policies**, create a policy:
   - **Policy name**: `General Bypass`
   - **Action**: **Bypass**
   - **Selector**: `Everyone`
5. Save

> **Why the Path field must be empty or cover both paths:**  
> When you click "Test step" in N8N, it registers the **Test URL** which uses `/webhook-test/...`. When the workflow is active in production, it uses `/webhook/...`. If you only bypass `/webhook`, the test verification will still get blocked by Cloudflare Access. Leaving the path empty ensures both work.

> **Alternative:** If you don't want to bypass the entire subdomain, add **two** public hostname entries — one with path `webhook` and one with path `webhook-test`.

---

## **5. Deleting an Existing Webhook Subscription**

If you get this error in N8N:
```
The WhatsApp App ID XXXXX already has a webhook subscription. 
Delete it or use another App before executing the trigger.
```

This means there's already a webhook subscription registered for your App. You need to delete it first using the **App Access Token** (which is `APP_ID|APP_SECRET`):

```bash
curl -s -X DELETE "https://graph.facebook.com/v25.0/YOUR_APP_ID/subscriptions?access_token=YOUR_APP_ID|YOUR_APP_SECRET&object=whatsapp_business_account"
```

Replace `YOUR_APP_ID` and `YOUR_APP_SECRET` with the values from **App Settings > Basic** in the Meta Developer Dashboard.

**Expected response:**
```json
{"success": true}
```

> **Important:** This uses an **App Access Token** (`APP_ID|APP_SECRET`), NOT a System User Token. If you use a System User Token you'll get the error: `(#15) This method must be called with an app access_token.`

After deleting, go back to N8N and click **"Test step"** again.

---

---

### **Setting up the WhatsApp API (sending Messages, Templates, etc.) in N8N**  

---

## **1. After setting up OAuth for Receiving Data**  
We now want to set up the API to **send messages and templates**.  

---

## **2. Navigate to Business Settings**  
- Go to **[business.facebook.com](https://business.facebook.com)**.  
- In the bottom left, click on **Settings**.  
- **Make sure** to select the correct **Business Portfolio**.  

---

## **3. Get the Business Account ID**  
1. In **Settings**, go to:  
   - **Accounts → WhatsApp Accounts**  
   - **Select the WhatsApp Account** you want to send messages from.  
2. Copy the **Business Account ID** shown on this page.

![Screenshot 2025-02-23 113117](https://github.com/user-attachments/assets/e6b0569b-d17c-43a2-91bd-f612f33b1135)


---

## **4. Generating the Access Token**  
1. In the **same Business Settings**, go to:  
   - **Users → System Users**
![Screenshot 2025-02-23 113827](https://github.com/user-attachments/assets/07c8b564-7c24-4b60-8428-db2b12fd4ed6)

2. **Click on your System User**.  
   - If not listed, **add yourself as a System User first**.  
3. Click on **Generate Token** and follow these steps:  
   - **Select the correct App**.  
   - Set **Token Expiration to Never** (otherwise, the token is valid only for 60 days).  
   - **Select the following permissions**:  
     - `whatsapp_business_management`  
     - `whatsapp_business_messaging`
![Screenshot 2025-02-23 113922](https://github.com/user-attachments/assets/4dd4078b-6ee6-4d98-81f4-79a6a6270a6f)

4. Click on **Generate Token**.  
   - If prompted, **verify your account** via email and then repeat the selection process.  
5. **Copy and Paste the Token** in the Access Token field in N8N.

![Screenshot 2025-02-23 114140](https://github.com/user-attachments/assets/ec054842-3e9c-458f-b7f2-3c58677eced3)

---

## **5. Adding Credentials in N8N**  
1. In **N8N**, go to **Credentials**.  
2. Add the following:  
   - **Business Account ID**: Paste the ID from Step 3.  
   - **Access Token**: Paste the Token from Step 4.  
3. **Save the credentials**.  

---

## **6. Done**  
The setup is now complete and you can send messages and templates using the WhatsApp API in N8N.

## Important: Using the Same Phone Number in Multiple Workflows

If you want to use a phone number that is already integrated in another N8N workflow, you need to follow this process to avoid the "phone number already in use" error:

1. **Set up your new workflow** with the WhatsApp webhook node
2. **Click "Listen for Test Event"** in the new workflow  
3. **Immediately send a message** to the WhatsApp number from your phone (within 2-5 seconds)
4. **The new workflow will now be active** and the old workflow will be automatically deactivated (or at least the webhook does not trigger any longer)

If you don't send a message within the 2-5 second window, you'll get an error that the phone number is already in use.



**Need a more advanced deployment, integration, or enterprise automation?** Visit [Stardawnai.com](https://stardawnai.com) for professional consulting and development on AI-driven process automation, SAP integration, self-hosted enterprise N8N workflows, and custom hybrid infrastructure solutions tailored to your business needs.
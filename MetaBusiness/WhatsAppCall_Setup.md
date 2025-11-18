# WhatsApp Cloud API - Calling Feature Setup Guide

Complete guide to enable and test WhatsApp Business Calling API.

---

## Prerequisites Check

Before enabling calling, verify ALL requirements:

### 1. Business Number Requirements
- ✅ Business number registered with **Cloud API** (not WhatsApp Business App)
- ✅ Messaging limit: **minimum 2000** business-initiated conversations per 24h
- ✅ Business verification: **COMPLETED**
- ✅ Payment method: **CONFIGURED**

### 2. App Configuration
- ✅ App **subscribed to WABA**
- ✅ App has `whatsapp_business_messaging` permission
- ✅ Webhook subscribed to **`calls`** field

### 3. Country Restrictions
⚠️ **Business-initiated calling NOT supported for numbers from:**
US, Nigeria, Canada, Vietnam, Turkey

📖 **Documentation:** https://developers.facebook.com/docs/whatsapp/cloud-api/calling/

---

## Step 1: Get Phone Numbers from WABA

Retrieve all phone numbers to identify the correct Phone Number ID:

```bash
curl -X GET 'https://graph.facebook.com/v23.0/{WABA_ID}/phone_numbers' \
  -H 'Authorization: Bearer {ACCESS_TOKEN}'
```

**Replace:**
- `{WABA_ID}` - Your WhatsApp Business Account ID
- `{ACCESS_TOKEN}` - Your access token

**Example Response:**
```json
{
  "data": [
    {
      "verified_name": "Your Business Name",
      "display_phone_number": "+1 234 567 8900",
      "quality_rating": "GREEN",
      "platform_type": "CLOUD_API",
      "webhook_configuration": {
        "application": "https://yourapp.com/webhook"
      },
      "id": "123456789012345"
    }
  ]
}
```

**Save the `id` field** - this is your **PHONE_NUMBER_ID**

---

## Step 2: Check App Subscription to WABA

Verify which apps are subscribed:

```bash
curl -X GET 'https://graph.facebook.com/v23.0/{WABA_ID}/subscribed_apps' \
  -H 'Authorization: Bearer {ACCESS_TOKEN}'
```

**If empty `{"data": []}`**, subscribe your app:

```bash
curl -X POST 'https://graph.facebook.com/v23.0/{WABA_ID}/subscribed_apps' \
  -H 'Authorization: Bearer {ACCESS_TOKEN}'
```

**Success:**
```json
{
  "success": true
}
```

---

## Step 3: Subscribe Webhook to `calls` Field

**In Meta App Dashboard:**

1. Go to: `https://developers.facebook.com/apps/{YOUR_APP_ID}/whatsapp-business/wa-settings/`
2. Navigate to **WhatsApp** → **Configuration**
3. Under **Webhook**, click **Edit**
4. Click **Manage** under Webhook fields
5. ✅ Check the **`calls`** checkbox
6. Click **Done** and **Save**

---

## Step 4: Enable Calling on Phone Number

Enable calling on your production number:

```bash
curl -X POST 'https://graph.facebook.com/v23.0/{PHONE_NUMBER_ID}/settings' \
  -H 'Authorization: Bearer {ACCESS_TOKEN}' \
  -H 'Content-Type: application/json' \
  -d '{
    "messaging_product": "whatsapp",
    "calling": {
      "status": "ENABLED",
      "call_icon_visibility": "DEFAULT"
    }
  }'
```

**Replace:**
- `{PHONE_NUMBER_ID}` - From Step 1 response
- `{ACCESS_TOKEN}` - Your token

**Success:**
```json
{
  "success": true
}
```

---

## Step 5: Verify Calling Status

Check that calling is enabled:

```bash
curl -X GET 'https://graph.facebook.com/v23.0/{PHONE_NUMBER_ID}/settings?fields=calling' \
  -H 'Authorization: Bearer {ACCESS_TOKEN}'
```

---

## Testing Calls

### Request Call Permission

Before calling, you MUST request permission:

```bash
curl -X POST 'https://graph.facebook.com/v23.0/{PHONE_NUMBER_ID}/messages' \
  -H 'Authorization: Bearer {ACCESS_TOKEN}' \
  -H 'Content-Type: application/json' \
  -d '{
    "messaging_product": "whatsapp",
    "to": "1234567890",
    "type": "interactive",
    "interactive": {
      "type": "call_permission_request",
      "body": {
        "text": "Can I call you?"
      }
    }
  }'
```

**Note:** 
- `to` must be in international format without + sign
- Limit: 1 request per 24h, max 2 in 7 days
- User must click "Allow"

### Permission Webhook Response

```json
{
  "call_permission_requests": [{
    "from": "1234567890",
    "status": "granted"
  }]
}
```

### Initiating Calls

⚠️ **Important:** Direct REST API calls NOT supported. You need:
- **WebRTC** implementation
- **SIP** client

Calls require real-time SDP negotiation.

---

## Troubleshooting

### Error: (#138018) Technical pre-requisites not met

**Common causes:**
1. Using test number (555 numbers) - calling only on production numbers
2. App not subscribed to WABA
3. Webhook not subscribed to `calls`
4. Messaging limit below 2000
5. Number from unsupported country

### Verify Setup

```bash
# Check token
curl -X GET 'https://graph.facebook.com/v23.0/debug_token?input_token={TOKEN}&access_token={TOKEN}'

# Check subscribed apps
curl -X GET 'https://graph.facebook.com/v23.0/{WABA_ID}/subscribed_apps' \
  -H 'Authorization: Bearer {TOKEN}'

# Check number details
curl -X GET 'https://graph.facebook.com/v23.0/{PHONE_NUMBER_ID}?fields=display_phone_number' \
  -H 'Authorization: Bearer {TOKEN}'
```

---

## Important Values Reference

Track these IDs:

| Name | Example | Where to Find |
|------|---------|---------------|
| WABA ID | 123456789012345 | Meta Business Manager → WhatsApp Accounts |
| Phone Number ID | 987654321098765 | GET /{WABA_ID}/phone_numbers |
| App ID | 111222333444555 | Meta App Dashboard |
| Access Token | EAA... | System User in Business Manager |

---

## Resources

- [WhatsApp Calling Docs](https://developers.facebook.com/docs/whatsapp/cloud-api/calling/)
- [Meta Developer Dashboard](https://developers.facebook.com/apps)
- [Meta Business Manager](https://business.facebook.com)
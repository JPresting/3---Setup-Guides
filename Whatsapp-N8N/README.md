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

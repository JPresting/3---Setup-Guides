# Setting Up a Meta and Whatsapp Business Account/Portfolio from Scratch

**Note:** You must have a registered business to follow these steps and verify your account.


## 1. **Go to business.facebook.com and create a business portfolio (if its your first you should be prompted with it directly)**

![image](https://github.com/user-attachments/assets/c66ac791-7262-4d45-8322-5cf7bcd06f62)

![image](https://github.com/user-attachments/assets/0ecd23a2-46f5-4240-b931-879fdffd9cc8)

![image](https://github.com/user-attachments/assets/229cc624-57ea-4e12-ac1d-358fd7fc690c)

### 1.2 **Now, go to the **Business Portfolio Info** section (very top) and fill out the required details by clicking **Edit** under **Business Details**.**

![image](https://github.com/user-attachments/assets/67b1ea24-8618-4bde-8fa0-5fde70bb1ef4)


### 1.3 **After that, select the newly created **Business Portfolio** and navigate to **Business Manager Settings**.**

![image](https://github.com/user-attachments/assets/50fb48a1-bcfa-43fc-bb05-823dda45a64c)

### 1.4 **Now, if you click on **Security Center** to verify your business, it will most likely display the message: **"Your organization does not need to be verified."****

![Screenshot 2025-03-12 103414](https://github.com/user-attachments/assets/60d651bf-644a-4e3c-8a05-6ac984a56860)

That’s because you haven’t created or added anything that would prompt Meta to require verification. This is needed for using the WhatsApp Business API!

👉 **Quick workaround:** Simply create an app.

![image](https://github.com/user-attachments/assets/38e1db2f-7035-444b-b8ba-923dd429e243)

Fill out the parameters however you like—you can always delete it later.

It will redirect you to the [developers.facebook.com](https://developers.facebook.com) page. You can close that tab for now and return to the Business Manager site. Then, click on **Security Center** again. It should now prompt you with a button to **Start your verification**. Click on "Start Verification".

![image](https://github.com/user-attachments/assets/cafe1814-ad2b-442c-bb57-e3717396b88f)


Enter your information, and if your company can't be found, you can select **"My business isn't listed."** However, I'm not sure how the setup process changes from that point onward.

![image](https://github.com/user-attachments/assets/05f984c8-6c7b-4ccd-900a-bc2cbbd04115)

It then requires you to attach a document which approves your connection to the business

![image](https://github.com/user-attachments/assets/5363c95f-90a8-43e7-be2e-5e8b5fbf58e9)

it will prompt you with "We'll use the official government-issued ID or official document you provide to help confirm your relationship to the business."

![image](https://github.com/user-attachments/assets/5c1f32a9-23a4-4b3d-9f14-16085e419e03)

After submitting your ID, they will notify you that it takes approximately 48 hours for them to review it.

![image](https://github.com/user-attachments/assets/2c03c8f7-6da2-4742-90be-cd841f47685e)

## 2. **Setting up a Whatsapp Business Account and verifying it**
After your account has hopefully been verified, you can create and add your accounts. I will now focus on explaining how to add and verify a WhatsApp Business Account. Please follow the steps carefully.
Since you don’t have a WhatsApp account yet, start by visiting [developers.facebook.com](https://developers.facebook.com/) and navigating to **My Apps**.

![image](https://github.com/user-attachments/assets/995aa402-1d01-4b0b-90e7-fe6324e70e6c)

You should now see the one we created earlier. To verify your business, you can either use that one, remove it, or leave it as is—we’ll be starting from scratch.

Go to "Create App"

![image](https://github.com/user-attachments/assets/f471d0b6-dfa6-466d-a3da-b98731ebb05b)

1. Enter a name (make sure to name it in a way that allows you to easily assign it later, as each number currently requires its own app).
2. Under **Use Cases**, select **"Other"**.
3. Select **Type: "Business"**.
4. You can now assign your **Business Portfolio** to the app. While this is optional, it’s recommended to do it here, as you will be prompted for it later if needed.
5. Click **Create App**.


Depending on when you watch this, your interface should look something like this:

![image](https://github.com/user-attachments/assets/d7f4feda-ec58-46c0-994f-c6199a8018c6)

Click on **Set Up WhatsApp**.

![image](https://github.com/user-attachments/assets/e0cef1bf-3ad8-4846-b784-52700810b3ed)

After clicking on **"Get Started,"** you should now see your **WhatsApp Product** displayed in the bottom left. Go to **API Setup**.

![image](https://github.com/user-attachments/assets/3e7b7c97-673f-49f9-86a6-b174529cb3be)


Now, you can use your test number to send a message to your own number—especially if you have little experience with API handling.

![image](https://github.com/user-attachments/assets/316d2724-65b6-4bf2-a127-e483961db40d)

Click on **Test Number** → **Add Phone Number**.

![image](https://github.com/user-attachments/assets/ad4cf011-7cfd-4e5c-9554-8197d37a86ba)

Here, we can create the **WhatsApp Business Account**.

![image](https://github.com/user-attachments/assets/19036dae-866d-4625-ab95-1d3d7c5b51ee)

Be aware that the name you enter here must somewhat match your actual business name or at least include it. You can follow their provided link for guidelines, but it's not entirely clear what will be accepted and what won’t.

Enter a number. This cannot be a number that’s already associated with any WhatsApp account—whether private or business. It must be a new number.
Since you only need to receive an SMS once and then won’t need the number anymore, I suggest ordering a bunch of free numbers. In Germany, you can order up to 10 free numbers per year from [O2 Free Card](https://www.o2-freikarte.de/). Alternatively, you can also purchase prepaid numbers from various providers, as they typically don't cost much either.

![image](https://github.com/user-attachments/assets/90951ca9-bdf5-4ebf-ad52-d2811be5f04a)

![image](https://github.com/user-attachments/assets/846caa91-8779-48e0-99f2-e52433302c3d)

Now you should be prompted with a missing payment method. Click on it.

![image](https://github.com/user-attachments/assets/6c476cff-dffd-48f4-9384-e14b89f1351e)

It will lead you to this page - click on Payment Settings (if it doesn't load just refresh it):

![image](https://github.com/user-attachments/assets/b3a577c0-48f1-417d-936d-3d9bd7131f2d)

Now, here’s something many people miss: You must first **add your payment method** and then **connect it to the account**. Simply adding the payment method won't be enough—it will still show an error if you don’t connect it properly.

![image](https://github.com/user-attachments/assets/369438eb-caca-4281-b49f-4c1e44d93eb7)

![image](https://github.com/user-attachments/assets/07e9bf17-04fc-4cf4-8087-690538433253)

Click on the 3 dots and set as default.

![Screenshot 2025-03-13 125354](https://github.com/user-attachments/assets/434070cb-21a1-443e-967c-11f961824afb)

This method adds the payment method to your account (there are other ways, but this is the quickest solution).

![image](https://github.com/user-attachments/assets/b4cda34c-98ae-4843-a697-33a91bcd5357)


Go back to [developers.facebook.com](https://developers.facebook.com) and refresh the page. The missing payment info should now have disappeared.

### Now we need to verify the phone number
If you go back to the [business.facebook.com](https://business.facebook.com) page and refresh it, it should now say **Business verification: approved**, **Account status: approved**, and show your **Payment Method**. Click on **WhatsApp Manager**.

![image](https://github.com/user-attachments/assets/ce569e85-120f-4ce6-98ba-d2b27bafd677)



Under Phone Numbers it now says Pending (its not really Pending nothing will happen once you dont act from here)

![image](https://github.com/user-attachments/assets/96986918-d9f0-4922-946c-69f2496b0694)

Make sure you have added a system user and granted all the whatsapp and app permissions otherwise, you will get an error like this:

<img width="614" height="301" alt="Screenshot 2025-11-13 111015" src="https://github.com/user-attachments/assets/1fdcd8a4-92ed-453e-a15a-43a4e04b9e48" />
<img width="1982" height="1074" alt="Screenshot 2025-11-13 114301" src="https://github.com/user-attachments/assets/3959e58e-4c88-48ac-9198-0da1738b3e90" />

It might take a couple of minutes to register.



Go back to the developers.facebook.com page click on Generate access token and remember the last couple of digits of your WabaID which is displayed on the same page.


![image](https://github.com/user-attachments/assets/44a9c0cf-1eb1-4e7f-ad34-4e79b7cc5988")



![image](https://github.com/user-attachments/assets/6e9bf94e-cbb4-46f2-b557-c6fbfb43d483)

1. Click on Edit Previous Settings

![image](https://github.com/user-attachments/assets/adc1aedb-237a-4438-bd39-a9c45f637f4b)

2. Select the one that matches your **WhatsApp Business Account ID**. In your case, there should be just two options: the one you created and the Test Account.

![image](https://github.com/user-attachments/assets/946155e7-0551-42d6-a38c-c3009cc0d901)


3. Click on "Continue" , "Save", and "Got it".

4. Go back to the [business.facebook.com](https://business.facebook.com) page and refresh it. It should now say **Status: Connected**.

![image](https://github.com/user-attachments/assets/a1b549fa-7f38-4e22-ab7c-c0055c69fc55)

5. If it does not show a **Messaging limit** and says **"Unavailable"**, this usually means your business is not verified yet or your display name is not approved. In that case, if you're using WhatsApp Desktop, contact the new number manually from your private number—just send any message. After at most one day, it should show you the messaging limit.

![image](https://github.com/user-attachments/assets/627f36f7-a63f-4185-b38c-65e05ebc409e)


### Official business account checkmark

The last thing we are going to do is

1. Go back to the Whatsapp Business Manager page and select the phone number
2. It should now show you a limit of 1K messages per 24 hours if you click on Insights

![image](https://github.com/user-attachments/assets/2360bb03-c566-4e97-b89d-a0fd49e1353b)

3. If your business has a notable online presence, such as published articles, blog posts, or independent reviews, you can apply for an official WhatsApp Business Account. If approved, WhatsApp will add a blue checkmark to your profile, enhancing trust and credibility. However, their criteria for approval are somewhat unclear. As they state:
*"Articles, blog posts, or independent reviews that show your business is well-known and recognized by consumers."*
This means that simply having a website or social media accounts is not sufficient—WhatsApp looks for third-party sources that establish your brand's recognition.

4. Click on Submit Request and fill out the form (NOTE: your display name has to be approved and your messaging limit needs to be >= 1K messages /24hrs)

![image](https://github.com/user-attachments/assets/a84971f3-3e65-4a1b-9d1b-d04878f55eb7)

![image](https://github.com/user-attachments/assets/feede424-0a06-4fff-891e-4ed22fd7a516)





## **2.1 Using the WhatsApp Business API**
How to set up this API on N8N or other API platforms is shown in detail in this repo: https://github.com/JPresting/Whatsapp-N8N.



**Need a more advanced deployment, integration, or enterprise automation?** Visit [Stardawnai.com](https://stardawnai.com) for professional consulting and development on AI-driven process automation, SAP integration, self-hosted enterprise N8N workflows, and custom hybrid infrastructure solutions tailored to your business needs.

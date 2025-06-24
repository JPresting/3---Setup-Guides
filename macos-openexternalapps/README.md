# How to Disable Gatekeeper on macOS

Complete guide to allow apps from unidentified developers on macOS.

## The Problem

When you try to open an app downloaded from the internet, macOS shows:
- "App can't be opened because it is not from an identified developer"
- App bounces in dock but never opens
- "Allow Anyway" button doesn't work or doesn't appear

## The Solution

### Step 1: Disable Gatekeeper via Terminal

1. Open **Terminal** (press `Cmd + Space`, type "Terminal")
2. Run this command:
   ```bash
   sudo spctl --master-disable
   ```
3. Enter your **admin password** when prompted
4. You'll see: `"Globally disabling the assessment system needs to be confirmed in System Settings"`

### Step 2: Confirm in System Settings

1. Open **System Settings** → **Privacy & Security**
2. **Navigate away** from Privacy & Security (click "Lock Screen" or any other section)
3. **Navigate back** to **Privacy & Security**
4. Scroll down to the **"Security"** section
5. Find **"Allow applications downloaded from:"**
6. Select **"Anywhere"** (this option only appears after Step 1)
7. A dialog will appear asking "Allow From Anywhere?"
8. Click **"Allow From Anywhere"** button
9. Enter your **admin password** to confirm
<img width="996" alt="Bildschirmfoto 2025-05-27 um 13 08 42" src="https://github.com/user-attachments/assets/c34016d4-1047-4ca9-89ae-6b1e5b756974" />

### Step 3: Test Your App

Your previously blocked app should now open normally. Double-click to launch it.

If the app still doesn't open, try:
- Right-click the app → **"Open"** 
- Or go back to System Settings → Privacy & Security and look for a message about the blocked app with an **"Allow Anyway"** button

## Notes

- This disables macOS security checks for all downloaded apps
- Only do this if you trust the apps you're installing
- You can re-enable Gatekeeper anytime with: `sudo spctl --master-enable`

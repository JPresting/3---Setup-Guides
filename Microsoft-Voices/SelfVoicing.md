# Fix: Enable Natural English Voices for Self-Voicing Games (Ren'Py / SAPI)

This guide fixes the issue where games read English text with a German accent or force the usage of robotic voices (like Microsoft Zira) on Windows 11.

## 1\. Prerequisites (Install Voice Data)

1.  Go to **Settings** \> **Time & language** \> **Language & region**.
2.  Click **Add a language** \> Select **English (United States)**.
3.  **Important:** In the install window, ensure **Text-to-speech** is checked.
4.  Click **Install**.

## 2\. Install SAPI Adapter

Most games use the old SAPI5 interface, which hides modern "Natural" voices. You need an adapter to make them visible.

1.  Download **NaturalVoiceSAPIAdapter** (by gexgd0419):
    [https://github.com/gexgd0419/NaturalVoiceSAPIAdapter/releases](https://github.com/gexgd0419/NaturalVoiceSAPIAdapter/releases)
2.  Run the tool.
3.  Click **Install 32-bit** (Required for most Ren'Py games).
4.  Click **Install 64-bit** (Optional, but recommended).
5.  Ensure **"Enable Narrator natural voices"** is checked.

## 3\. Set Default Legacy Voice

1.  Press `Win + R` on your keyboard.
2.  Paste the following command to open the 32-bit Speech Control Panel:
    ```
    C:\Windows\SysWOW64\Speech\SpeechUX\sapi.cpl
    ```
3.  In the "Voice Selection" dropdown, choose the high-quality voice (e.g., **Microsoft Jenny Natural - English**).
    *(Note: Without step 2, this voice will not appear here).*
4.  Click **Apply** and **OK**.

## 4\. Final Step

1.  **Restart your game** completely.
2.  Enable self-voicing (usually `V`).
3.  The game will now use the Natural English voice.
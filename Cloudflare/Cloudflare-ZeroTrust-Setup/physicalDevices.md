# Cloudflare Zero Trust: Device-Based Access Setup

## Problem
Applications protected by Cloudflare Access with Email Verification or IP restrictions cannot be accessed from mobile devices or changing IP addresses.

## Solution
Use Cloudflare Zero Trust with WARP device authentication to allow trusted devices from anywhere, independent of IP address.

---

## Step 1: Configure Device Enrollment Permissions

**Path:** `Team & Resources` → `Devices` → `Device profiles` → `Management` tab → `Device enrollment permissions` → `Manage`

<img width="1254" height="436" alt="Screenshot 2026-01-10 175253" src="https://github.com/user-attachments/assets/bf00f8b3-427a-455a-9c8d-17d0b94f426b" />


**Add enrollment policy:**
- **Policy name:** Any descriptive name (e.g., "Team Devices")
- **Action:** Allow
- **Include:**
  - **Selector:** Emails
  - **Values:** Enter email addresses authorized to enroll devices
    ```
    user1@example.com
    user2@example.com
    user3@example.com
    ```
- **Authentication:** One-time PIN (default)

**Save**

---

## Step 2: Create WARP Posture Check

**Path:** `Reusable components` → `Posture checks` → `Add a check`

**Select:** `Warp`

**Configuration:**
- **Check name:** Leave default or customize
- Click **Save**

This creates a check that verifies devices are connected via WARP VPN.

---

## Step 3: Configure Application Access Policy

**Path:** `Access controls` → `Access` → `Applications` → Select your application → `Policies` tab

**Add new policy at position 1 (top of policy list):**

- **Policy name:** "WARP Devices Bypass"
- **Action:** **Bypass**
- **Configure rules → Add include:**
  - **Selector:** WARP
  - **Value:** Select the WARP check from Step 2
- **Save**

**Recommended policy order:**
1. WARP Devices Bypass (BYPASS) - for enrolled mobile devices
2. IP ranges (BYPASS) - optional, for trusted network locations
3. Email access (ALLOW) - for web browser access

**Important:** Bypass policies must be above Allow/Deny policies.

---

## Step 4: Enroll User Devices

**On each device (iPhone/Android/Desktop):**

1. **Install:** "Cloudflare One Agent" app
2. **Open app** → Enter your organization's team name
   - Find your team name in Cloudflare dashboard: `Settings` → `Team name`
3. **Authenticate:** Login with email from enrollment permissions (Step 1)
4. **Verify:** Enter One-Time PIN sent to email
5. **Install VPN profile** when prompted
6. **Confirm:** Status shows "Connected"

**Verify enrollment:**
- `Team & Resources` → `Devices` → `Your devices` tab
- Enrolled devices appear in the list

---

## Step 5: Access Application

**With WARP connected:**
- Enrolled devices can access the application from any network
- Any IP address is accepted
- No additional authentication required (Bypass policy)

**With WARP disconnected:**
- Access depends on other policies (IP ranges, email authentication)
- May require login or be blocked

---

## Use Cases

**Remote work:**
- Employees access company resources from any location
- Mobile devices work on cellular networks
- No VPN configuration needed on infrastructure side

**Home network + mobile:**
- Optional: IP-based bypass for home network
- WARP-based bypass for mobile/travel
- Users choose connection method

**Family/team sharing:**
- Each member enrolls with their own email
- Individual device management
- Shared access to protected resources

---

## Troubleshooting

**Device cannot enroll:**
- Email not in enrollment permissions list (Step 1)
- Check team name spelling

**Cannot access application with WARP connected:**
- WARP bypass policy not at position 1
- Device not enrolled (check Devices list)
- WARP posture check not created (Step 2)

**Application works from home but not mobile:**
- WARP not connected on mobile device
- Device enrollment failed

---

## Summary

This setup enables:
- **Device-based authentication:** Trusted devices access from anywhere
- **IP-independent:** No need to maintain IP allowlists for mobile users
- **Simple enrollment:** Users enroll via email verification
- **Flexible policies:** Combine with IP rules, email authentication as needed

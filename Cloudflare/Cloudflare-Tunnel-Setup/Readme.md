# Cloudflare Tunnel Setup - Complete Guide

## Overview

This guide shows how to set up **Cloudflare Tunnels** as a powerful alternative to services like ngrok, localtunnel, or other third-party tunneling providers. With Cloudflare Tunnels, you maintain full control over your infrastructure while only paying for a domain name (~$10/year). You can create unlimited tunnels to expose any number of local services to the internet.

### Benefits over alternatives:
- ✅ **Complete control** - No third-party dependencies
- ✅ **Unlimited tunnels** - Only pay for domain registration
- ✅ **Enterprise-grade security** - Built-in DDoS protection
- ✅ **Custom domains** - Professional appearance
- ✅ **Zero Trust integration** - Advanced access controls
- ✅ **No session limits** - Unlike ngrok free tier

---

## Prerequisites

- A domain purchased through Cloudflare (~$10/year)
- Windows machine for local services
- Basic command line knowledge

---

## Step 1: Domain Setup

1. **Purchase domain** via Cloudflare Domains
2. **Add domain** to Cloudflare if not automatically added
3. **Verify DNS** is managed by Cloudflare

---

## Step 2: Install Cloudflared

### Windows Installation:
```cmd
# Download from: https://github.com/cloudflare/cloudflared/releases
# Or use package manager of choice
```

### Authenticate with Cloudflare:
```cmd
cloudflared tunnel login
```
- Browser opens automatically
- Login to Cloudflare account
- Grant permissions

---

## Step 3: Create Tunnels

### Create individual tunnels for each service:
```cmd
cloudflared tunnel create ollama
```

**Output example:**
```
Tunnel credentials written to C:\Users\username\.cloudflared\a1b2c3d4-e5f6-7890-abcd-ef1234567890.json
Created tunnel ollama with id a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

```cmd
cloudflared tunnel create n8n
```

**Output example:**
```
Tunnel credentials written to C:\Users\username\.cloudflared\x9y8z7w6-v5u4-3210-zyxw-vu9876543210.json
Created tunnel n8n with id x9y8z7w6-v5u4-3210-zyxw-vu9876543210
```

### List existing tunnels:
```cmd
cloudflared tunnel list
```

**Output example:**
```
ID                                   NAME   CREATED              CONNECTIONS
a1b2c3d4-e5f6-7890-abcd-ef1234567890 ollama 2025-08-18T13:21:12Z
x9y8z7w6-v5u4-3210-zyxw-vu9876543210 n8n    2025-08-18T13:40:13Z 1xfra08, 1xfra10
```

**IMPORTANT:** Copy the tunnel IDs from these outputs - you'll need them for the configuration files!

---

## Step 4: Configure DNS Routing

### Route domains to tunnels:
```cmd
cloudflared tunnel route dns ollama api.yourdomain.com
```

**Output example:**
```
Added CNAME api.yourdomain.com which will route to this tunnel tunnelID=a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

```cmd
cloudflared tunnel route dns n8n automation.yourdomain.com
```

**Output example:**
```
Added CNAME automation.yourdomain.com which will route to this tunnel tunnelID=x9y8z7w6-v5u4-3210-zyxw-vu9876543210
```

### Verify DNS routing:
```cmd
cloudflared tunnel route dns list
```

**Output example:**
```
api.yourdomain.com         CNAME   a1b2c3d4-e5f6-7890-abcd-ef1234567890.cfargotunnel.com
automation.yourdomain.com  CNAME   x9y8z7w6-v5u4-3210-zyxw-vu9876543210.cfargotunnel.com
```

---

## Step 5: Create Configuration Files

### For Ollama service (`config-ollama.yml`):
```yaml
tunnel: a1b2c3d4-e5f6-7890-abcd-ef1234567890
credentials-file: C:\Users\username\.cloudflared\a1b2c3d4-e5f6-7890-abcd-ef1234567890.json

ingress:
  - hostname: api.yourdomain.com
    service: http://localhost:11434
    originRequest:
      httpHostHeader: localhost:11434
  - service: http_status:404
```

### For N8N service (`config-n8n.yml`):
```yaml
tunnel: x9y8z7w6-v5u4-3210-zyxw-vu9876543210
credentials-file: C:\Users\username\.cloudflared\x9y8z7w6-v5u4-3210-zyxw-vu9876543210.json

ingress:
  - hostname: automation.yourdomain.com
    service: http://localhost:5678
    originRequest:
      httpHostHeader: localhost:5678
  - service: http_status:404
```

**REPLACE WITH YOUR VALUES:**
- `tunnel:` → Use the tunnel ID from Step 3 output
- `credentials-file:` → Use the .json path from Step 3 output  
- `hostname:` → Your actual domain
- `username` → Your Windows username

**Important:** The `httpHostHeader` is crucial for services that validate host headers (like Ollama).

---

## Step 6: Start Tunnels

### Manual start:
```cmd
# Terminal 1
cloudflared tunnel --config C:\Users\username\.cloudflared\config-ollama.yml run

# Terminal 2  
cloudflared tunnel --config C:\Users\username\.cloudflared\config-n8n.yml run
```

### Automated startup scripts:

Create `.bat` files for easy management:

**Ollama-Cloudflare.bat:**
```batch
@echo off
title Ollama Auto-Service - Cloudflare Edition
set OLLAMA_ORIGINS=*
echo Starting Ollama server...
start /B ollama serve
timeout /t 15 /nobreak >nul
echo Starting Cloudflare tunnel...
start /B cloudflared tunnel --config config-ollama.yml run
echo Ollama ready at https://api.yourdomain.com
pause
```

---

## Step 7: Test Your Setup

1. **Start your local service** (Ollama, N8N, etc.)
2. **Start the corresponding tunnel**
3. **Access via your custom domain**

Example:
- Local: `http://localhost:11434/api/tags`
- Public: `https://api.yourdomain.com/api/tags`

---

## Step 8: Service Management Strategy

### Organizing Your Local Services

Create individual `.bat` files for each service you want to expose:
- `Ollama-Cloudflare.bat` - AI model server
- `N8N-Cloudflare.bat` - Automation platform  
- `Database-Cloudflare.bat` - Local database
- `API-Server-Cloudflare.bat` - Custom API
- Any other local Docker containers or applications

**Store all `.bat` files in:** `C:\Users\username\Documents\Quick-Start\`

### Optional: Master Startup Script

For convenience, you can create a master launcher that lets you selectively start services. This is **completely optional** but useful if you have many services.

**Place in Windows Startup folder:** `C:\Users\username\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Master-Startup-Launcher.bat`

```batch
@echo off
title Quick-Start Master Launcher
color 0B
setlocal enabledelayedexpansion

echo.
echo ========================================
echo       QUICK-START MASTER LAUNCHER
echo ========================================
echo.

REM Set the directory path
set "QUICKSTART_DIR=C:\Users\username\Documents\Quick-Start"

REM Check if directory exists
if not exist "%QUICKSTART_DIR%" (
    echo ERROR: Quick-Start directory not found!
    echo Path: %QUICKSTART_DIR%
    pause
    exit /b
)

REM Count and list all .bat files
set /a count=0
echo Available scripts:
echo.

for %%f in ("%QUICKSTART_DIR%\*.bat") do (
    set /a count+=1
    set "script[!count!]=%%~nxf"
    set "path[!count!]=%%f"
    echo !count!^) %%~nxf
)

if %count%==0 (
    echo No .bat files found in Quick-Start directory!
    pause
    exit /b
)

echo.
echo ========================================
echo.
echo Options:
echo   A = Run ALL scripts
echo   N = Run NOTHING (exit)
echo   Numbers = Run specific scripts (e.g., "1", "12", "135")
echo.
set /p choice="Enter your choice: "

REM Convert to uppercase for easier handling
for %%i in (A B C D E F G H I J K L M N O P Q R S T U V W X Y Z) do call set choice=%%choice:%%i=%%i%%

REM Handle choices
if /i "%choice%"=="A" goto run_all
if /i "%choice%"=="N" goto exit_script

REM Handle number selection
set "numbers_to_run="
set "input=%choice%"

:parse_numbers
if "%input%"=="" goto run_selected
set "digit=%input:~0,1%"
set "input=%input:~1%"

REM Check if digit is valid (1-9, we'll validate range later)
if "%digit%" geq "1" if "%digit%" leq "9" (
    REM Check if this number exists in our list
    if %digit% leq %count% (
        set "numbers_to_run=!numbers_to_run! %digit%"
    ) else (
        echo Warning: Script number %digit% does not exist (max: %count%)
    )
)
goto parse_numbers

:run_selected
if "%numbers_to_run%"=="" (
    echo No valid script numbers provided!
    goto ask_again
)

echo.
echo Running selected scripts...
for %%n in (%numbers_to_run%) do (
    echo Starting: !script[%%n]!
    start "" "!path[%%n]!"
    timeout /t 2 /nobreak >nul
)
goto success_exit

:run_all
echo.
echo Running ALL scripts...
for /l %%i in (1,1,%count%) do (
    echo Starting: !script[%%i]!
    start "" "!path[%%i]!"
    timeout /t 2 /nobreak >nul
)
goto success_exit

:ask_again
echo.
echo Please try again with valid input.
echo.
goto main_menu

:success_exit
echo.
echo ========================================
echo All selected scripts have been started!
echo ========================================
echo.
echo This window will close in 5 seconds...
timeout /t 5 /nobreak >nul
exit

:exit_script
echo.
echo No scripts executed. Goodbye!
timeout /t 2 /nobreak >nul
exit
```

### Usage Example:
```
Available scripts:
1) Ollama-Cloudflare.bat
2) N8N-Cloudflare.bat  
3) Database-Server.bat

Options:
  A = Run ALL scripts
  N = Run NOTHING (exit)
  Numbers = Run specific scripts (e.g., "1", "12", "135")

Enter your choice: 12
```
This would start Ollama and N8N, but skip the database server.

---

## ⚠️ SECURITY WARNING

**By default, anyone with your domain URL can access your services!** This creates a significant security risk.

### The Problem:
- `https://api.yourdomain.com` is publicly accessible
- External services (like OpenWebUI) can reference your endpoints
- No authentication required

### Solution: Cloudflare Access (Zero Trust)

---

## Step 9: Implement Security with Cloudflare Access

### Enable Zero Trust:
1. **Cloudflare Dashboard** → Your domain
2. **Zero Trust** → **Access** → **Applications**
3. **Add an application** → **Self-hosted**

### Configure Application:
- **Application name:** `Ollama API`
- **Public hostname:** `api.yourdomain.com`
- **Session Duration:** `24 hours`

### Create Security Policies:

**IMPORTANT:** Policy order matters! Cloudflare evaluates Bypass policies first, then Allow policies.

#### Policy 1: Bypass for Server (Priority 1)
- **Policy name:** `Server IP Bypass`
- **Action:** `Bypass`
- **Include:**
  - **IP ranges:** `your.server.ip/32`

#### Policy 2: Email Authentication (Priority 2)  
- **Policy name:** `Email Access`
- **Action:** `Allow`
- **Include:**
  - **Emails:** `your-email@domain.com`

### Result:
- ✅ **Your server IP** → Direct access (no login)
- ✅ **Your browser** → Email authentication required
- ❌ **Everyone else** → Blocked

---

## Step 10: Integration with External Services

### For OpenWebUI (Docker):
```bash
sudo docker run -d \
  --name openwebui \
  -p 3001:8080 \
  -e OLLAMA_BASE_URL="https://api.yourdomain.com" \
  -e ENABLE_SIGNUP="false" \
  -v /path/to/data:/app/backend/data \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:main
```

The server IP bypass allows OpenWebUI to access Ollama without authentication, while still protecting against unauthorized access from other IPs.

---

## Troubleshooting

### Common Issues:

**Error 1033 - Tunnel Error:**
- Check if tunnel is running: `cloudflared tunnel list`
- Verify DNS routing: `cloudflared tunnel route dns list`
- Ensure correct tunnel ID in configuration

**🚨 DNS Routing Conflicts (Most Common Issue):**

When experimenting with tunnels, DNS entries often point to wrong/deleted tunnels causing Error 1033.

**FASTEST SOLUTION:**
1. **Cloudflare Dashboard** → Your domain
2. **DNS** → **Records** tab
3. **Find the problematic CNAME** (e.g., `api`, `automation`)
4. **Delete it manually** (trash icon)
5. **Re-create via CLI:** `cloudflared tunnel route dns your-tunnel subdomain.yourdomain.com`

**Why this happens:**
- Created multiple tunnels with same subdomain
- Deleted tunnel but DNS entry remained
- Wrong tunnel ID in configuration

**Manual DNS cleanup is 10x faster than CLI troubleshooting!**

**403 Access Denied:**
- Verify `httpHostHeader` in configuration
- Check if local service is running
- Confirm `OLLAMA_ORIGINS=*` for Ollama

**Browser Security Warnings:**
- Normal for new domains (1-2 days)
- Report false positive to Google Safe Browsing
- Use different browser temporarily

**Multiple Tunnel Conflicts:**
- Each service needs separate tunnel ID
- Check `credentials-file` path matches tunnel ID
- Verify policy order: BYPASS → ALLOW

---

## Cost Analysis

### Cloudflare Tunnels:
- **Domain registration:** ~$5/year (for .uk domains)
- **Tunnel usage:** FREE (unlimited)
- **Access policies:** FREE (up to 50 users)
- **Total:** ~$5/year for unlimited tunnels, meaning all local instances can be made accessible via the web.

### Alternatives:
- **ngrok Pro:** $8/month = $96/year
- **localtunnel:** Limited reliability
- **Self-hosted:** Server costs + maintenance

---

## Advanced Tips

1. **Multiple environments:** Create separate tunnels for dev/staging/prod
2. **Load balancing:** Use multiple origin servers in configuration
3. **Custom headers:** Add authentication headers for specific services
4. **Monitoring:** Use Cloudflare Analytics to track usage
5. **Automation:** Script tunnel creation for rapid deployment

---

## Conclusion

Cloudflare Tunnels provide enterprise-grade tunneling capabilities at a fraction of the cost of alternatives. With proper security policies, you maintain complete control while enabling secure access to your local services from anywhere in the world.

The combination of custom domains, unlimited tunnels, and Zero Trust security makes this solution ideal for developers, small businesses, and anyone needing reliable remote access to local services.
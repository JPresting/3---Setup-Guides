# WhatsApp MCP Integration Setup
**Requirements:** Go installation + C compiler + QR code authentication + autostart configuration

WhatsApp MCP provides direct access to your personal WhatsApp account through Claude Desktop, enabling message reading, sending, contact management, and media handling. This integration uses the official WhatsApp Web multi-device API to maintain security and compliance.

**Architecture:** Two-component system requiring both Go WhatsApp Bridge + Python MCP Server to run simultaneously.

## Why These Prerequisites Are Required

**Go Language:** The WhatsApp bridge uses the `whatsmeow` library (written in Go) to interface with WhatsApp's official multi-device API. This is the same protocol used by WhatsApp Web.

**C Compiler:** The Go application uses SQLite for local message storage via the `go-sqlite3` package, which requires CGO (C bindings) to compile. Without a C compiler, you'll get the error: `Binary was compiled with 'CGO_ENABLED=0', go-sqlite3 requires cgo to work`.

**UV Python Package Manager:** The MCP server component is written in Python and uses UV for dependency management and execution, providing faster and more reliable package handling than pip.

## Prerequisites Installation

1. **Install Go**: Download from [golang.org/dl/](https://golang.org/dl/) and install
   - Verify installation: `go version`
   - Should return something like `go version go1.21.x windows/amd64`

2. **Install UV (Python Package Manager)**:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```
   - Verify installation: `uv --version`

3. **Install C Compiler (MSYS2) - Required for Windows SQLite Integration**:
   
   **Why needed:** Windows doesn't include a C compiler by default, but Go's SQLite integration requires one to compile C bindings.
   
   **Installation steps:**
   - Download and install [MSYS2](https://www.msys2.org/)
   - Open MSYS2 UCRT64 terminal (not regular Command Prompt!)
   - Run: `pacman -S mingw-w64-ucrt-x86_64-gcc`
   - This installs the MinGW-w64 GCC compiler specifically designed for Windows

   **Add to Windows PATH (Critical Step):**
   1. Press `Win + X` → Select "System"
   2. Click "Advanced system settings" on the right
   3. Click "Environment Variables" button
   4. In "System variables" section, find and select "Path"
   5. Click "Edit" → "New"
   6. Add: `C:\msys64\ucrt64\bin`
   7. Click "OK" on all windows
   8. **Restart PowerShell/Command Prompt** for changes to take effect
   9. Verify: `gcc --version` should show MinGW-w64 GCC version

4. **Optional - FFmpeg** (for audio message conversion):
   - Without FFmpeg: Can send raw audio files, but they won't appear as playable voice messages
   - With FFmpeg: Automatically converts MP3/WAV/etc. to WhatsApp's required .ogg Opus format
   - Install: Download from [ffmpeg.org](https://ffmpeg.org) or `choco install ffmpeg`

## Installation Steps

1. **Clone Repository**:
```bash
git clone https://github.com/lharries/whatsapp-mcp.git
cd whatsapp-mcp
```

2. **Configure C Compiler Environment** (Windows specific):
```powershell
# Navigate to bridge directory
cd whatsapp-bridge

# Enable CGO for Go compilation (required for SQLite)
go env -w CGO_ENABLED=1

# Verify CGO is enabled
go env CGO_ENABLED
# Should return: 1

# Set MSYS2 compiler in PATH for current session (if not in system PATH)
$env:PATH = "C:\msys64\ucrt64\bin;" + $env:PATH

# Verify compiler is found
gcc --version
# Should show: gcc.exe (Rev5, Built by MSYS2 project) 15.x.x
```

3. **First-time WhatsApp Authentication**:
```bash
# In whatsapp-bridge directory
go run main.go
```
   
   **What happens:**
   - First run downloads Go dependencies (may take 2-3 minutes)
   - Creates local SQLite database in `whatsapp-bridge/store/`
   - Displays QR code in terminal for WhatsApp authentication
   - Shows "Connected to WhatsApp!" message when successful
   - Starts REST API server on port 8080 for MCP communication

   **Authentication process:**
   - Open WhatsApp mobile app
   - Go to Settings → Linked Devices → Link a Device
   - Scan the QR code displayed in your terminal
   - Authentication persists for approximately 20 days
   - **Keep this terminal window open** - the bridge must run continuously

4. **Create Autostart Script for Windows Boot**:

   **Why needed:** The Go bridge must always be running for WhatsApp integration to work. Manual startup every time is impractical.

   Create `C:\whatsapp-bridge-autostart.bat`:
```batch
@echo off
echo Starting WhatsApp Bridge...
cd /d "C:\Users\YOUR_USERNAME\whatsapp-mcp\whatsapp-bridge"

REM Set PATH to include MSYS2 compiler - critical for compilation
set PATH=C:\msys64\ucrt64\bin;%PATH%

REM Enable CGO for SQLite integration
go env -w CGO_ENABLED=1

REM Start WhatsApp Bridge with status messages
echo WhatsApp Bridge is starting...
go run main.go

REM Auto-restart mechanism if bridge crashes
:restart
echo WhatsApp Bridge crashed. Restarting in 10 seconds...
timeout /t 10 /nobreak
go run main.go
goto restart
```

   **Replace `YOUR_USERNAME` with your actual Windows username**

5. **Add to Windows Startup System**:

   **Method 1 - Startup Folder (Recommended):**
```powershell
# Copy batch file to Windows startup folder
copy C:\whatsapp-bridge-autostart.bat "%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\"

# Verify it's there
explorer shell:startup
```

   **Method 2 - Create Quick Manual Restart Option:**
```powershell
# Create convenient restart location
mkdir "C:\Users\YOUR_USERNAME\Documents\Quick-Start"

# Create desktop shortcut in this folder for manual restart
$shortcut = (New-Object -comObject WScript.Shell).CreateShortcut("C:\Users\YOUR_USERNAME\Documents\Quick-Start\WhatsApp Bridge.lnk")
$shortcut.TargetPath = "C:\whatsapp-bridge-autostart.bat"
$shortcut.Save()

# Pin this folder to Quick Access for easy access
```

   **Why both methods:** Startup folder ensures automatic boot startup, while Quick-Start folder provides manual restart capability if you accidentally close the terminal.

## Claude Desktop Configuration

**Determine Required Paths:**
```powershell
# Find UV executable location
where.exe uv
# Example result: C:\Users\YOUR_USERNAME\AppData\Local\Programs\Python\Python313\Scripts\uv.exe

# Get full path to MCP server directory
cd C:\Users\YOUR_USERNAME\whatsapp-mcp\whatsapp-mcp-server
pwd
# Copy this full path for configuration
```

**Claude Desktop Config Addition:**

Add this to your `claude_desktop_config.json` (remember to replace paths with your actual results):
```json
"whatsapp": {
  "command": "C:\\Users\\YOUR_USERNAME\\AppData\\Local\\Programs\\Python\\Python313\\Scripts\\uv.exe",
  "args": [
    "--directory",
    "C:\\Users\\YOUR_USERNAME\\whatsapp-mcp\\whatsapp-mcp-server",
    "run",
    "main.py"
  ],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\"
  }
}
```

**Configuration Notes:**
- Use full absolute paths to avoid path resolution issues
- Double backslashes (`\\`) are required in JSON for Windows paths
- APPDATA environment variable is required for proper npm/Python package functionality on Windows
- The MCP server connects to the Go bridge via localhost:8080

## Available WhatsApp Tools & Capabilities

Once configured, Claude Desktop provides comprehensive WhatsApp integration:

**Message Management:**
- `search_contacts`: Find contacts by name or phone number pattern matching
- `list_messages`: Retrieve message history with date/sender/content filters and context
- `list_chats`: Display all available chats (individual + groups) with metadata
- `get_chat`: Get detailed information about specific chat (participants, settings, etc.)
- `send_message`: Send text messages to individual contacts or group chats

**Advanced Message Operations:**
- `get_message_context`: Retrieve messages before/after a specific message for context
- `get_last_interaction`: Find most recent message exchange with specific contact
- `get_direct_chat_by_contact`: Locate direct message thread with specific person
- `get_contact_chats`: List all chats (including groups) involving specific contact

**Media Handling:**
- `send_file`: Send any file type (images, videos, documents, raw audio files)
- `send_audio_message`: Convert and send audio as playable WhatsApp voice messages (.ogg format)
- `download_media`: Download received media files to local storage with full file paths

**Contact & Group Management:**
- Search across all contacts and group members
- Access group metadata (creation date, admin status, participant list)
- Differentiate between individual chats and group conversations

## Usage Examples & AI-Enhanced Capabilities

**Basic Message Operations:**
```
"Show me all my WhatsApp chats with their last messages"
"Search for messages containing 'meeting' from John in the last week"
"Send message 'Running 10 minutes late, starting without me' to the project team group"
"Find all messages from Sarah mentioning the Q4 budget proposal"
```

**Smart Contact Management:**
```
"Who are all my contacts with 'Maria' in their name?"
"When did I last talk to my colleague from the marketing department?"
"Show me all group chats that include both John and Sarah"
```

**Media Operations (requires full file paths):**
```
"Send the file C:\Users\stand\Documents\quarterly-report.pdf to my manager via WhatsApp"
"Download the last 3 images that Lisa sent me to my Documents folder"
"Send this Excel spreadsheet from my Desktop to the finance team group"
```

**AI-Enhanced Analysis:**
```
"Summarize the last 50 messages in the 'Family' group chat - what are the main topics?"
"Analyze my WhatsApp activity patterns: who do I message most and when?"
"Find all appointments, meetings, or deadlines mentioned in my chats from this week"
"Create a list of all phone numbers shared in my work-related group chats"
```

## Detailed Troubleshooting Guide

**CGO/Compilation Errors:**

**Error:** `Binary was compiled with 'CGO_ENABLED=0', go-sqlite3 requires cgo to work`
- **Cause:** Go can't find C compiler or CGO is disabled
- **Solutions:**
  1. Verify GCC installation: `gcc --version` (should show MSYS2 GCC)
  2. Check PATH includes MSYS2: `echo $env:PATH | Select-String msys64`
  3. Re-enable CGO: `go env -w CGO_ENABLED=1`
  4. Restart PowerShell after PATH changes
  5. If using Anaconda, it may conflict: `$env:PATH = "C:\msys64\ucrt64\bin;" + $env:PATH`

**WhatsApp Connection Issues:**

**Error:** "Server disconnected" with Group Messages
- **Cause:** Group chat IDs differ from individual chat IDs
- **Individual format:** `49123456789@s.whatsapp.net`
- **Group format:** `12345678901234567890-987654321@g.us`
- **Solutions:**
  1. Use: `"List all my WhatsApp group chats with their exact IDs"`
  2. Copy exact group ID from results for message sending
  3. Verify group ID format includes `@g.us` suffix

**Error:** `"unknown user server 'lid'"`
- **Cause:** Invalid or outdated contact/group ID
- **Solutions:**
  1. Use: `"Show me all my WhatsApp chats with their current IDs"`
  2. Restart WhatsApp Bridge: Stop current process, run `go run main.go` again
  3. Re-sync contacts: `"Search all my WhatsApp contacts to refresh the database"`

**Authentication & Sync Problems:**

**QR Code Not Appearing:**
- Check terminal supports QR code display (Windows Terminal, PowerShell, Command Prompt all work)
- Ensure no firewall blocking WhatsApp Web connections
- Try restarting authentication: Delete `whatsapp-bridge/store/*.db` files and restart

**Messages Out of Sync:**
- **Cause:** Database corruption or connection interruption
- **Nuclear option:** Delete both database files:
  - `whatsapp-bridge/store/messages.db`  
  - `whatsapp-bridge/store/whatsapp.db`
- Restart bridge: `go run main.go`
- Re-authenticate with QR code
- **Note:** This forces complete re-sync of all message history

**Autostart Issues:**

**Bridge Not Starting on Boot:**
1. Verify batch file location: `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\`
2. Test batch file manually: `C:\whatsapp-bridge-autostart.bat`
3. Check Windows Event Viewer for startup errors
4. Ensure Windows PATH includes MSYS2 (system-wide, not just session)
5. Alternative: Use Task Scheduler instead of Startup folder

**Performance & Resource Usage:**

**High CPU Usage:**
- Normal during initial sync (can take 10-30 minutes for large message histories)
- After sync: Should use minimal CPU unless actively processing messages
- Consider limiting message history sync if database becomes very large

**Memory Usage:**
- Typical usage: 50-150MB RAM for the Go bridge
- Large group chats or media-heavy conversations may increase usage
- Monitor via Task Manager: Look for `go.exe` process

## Security & Privacy Considerations

**Local Data Storage:**
- All messages stored in local SQLite database (`whatsapp-bridge/store/`)
- No cloud synchronization - data never leaves your computer
- Database files can be backed up or deleted as needed

**WhatsApp Terms Compliance:**
- Uses official WhatsApp Web multi-device API (same as browser WhatsApp Web)
- Respects WhatsApp's device linking limits (typically 4 devices max)
- Authentication follows WhatsApp's standard QR code protocol

**Claude Integration Privacy:**
- Message data only sent to Claude when you explicitly request WhatsApp operations
- No automatic scanning or background data transmission
- You control what messages/contacts Claude can access through your specific requests

**Network Security:**
- Bridge communicates with WhatsApp servers via encrypted connections
- Local MCP server uses localhost:8080 for Claude communication
- No external API calls except to WhatsApp's official endpoints

## Architecture Deep Dive

**Two-Component System Explained:**

1. **Go WhatsApp Bridge** (`whatsapp-bridge/`):
   - **Purpose:** Maintains persistent connection to WhatsApp servers
   - **Technology:** Uses `whatsmeow` library (official WhatsApp Web protocol)
   - **Responsibilities:**
     - QR code authentication and session management
     - Real-time message synchronization from WhatsApp servers
     - SQLite database management for local message storage
     - REST API server (port 8080) for MCP communication
   - **Must run continuously:** If stopped, loses WhatsApp connection

2. **Python MCP Server** (`whatsapp-mcp-server/`):
   - **Purpose:** Translates Claude requests into WhatsApp operations
   - **Technology:** Implements Model Context Protocol specification
   - **Responsibilities:**
     - Receives tool calls from Claude Desktop
     - Converts requests to REST API calls to Go bridge
     - Formats WhatsApp data for Claude consumption
     - Handles error translation and user feedback
   - **Started on-demand:** Claude Desktop launches when needed

**Data Flow Architecture:**
```
Claude Desktop ↔ Python MCP Server ↔ Go WhatsApp Bridge ↔ WhatsApp Servers
                     (MCP Protocol)    (REST API)        (WhatsApp Web API)
```

**Why Two Components:**
- **Language Specialization:** Go excels at WhatsApp API integration, Python excels at MCP protocol
- **Reliability:** Go bridge maintains persistent WhatsApp connection independent of Claude Desktop
- **Modularity:** Can restart MCP server without losing WhatsApp authentication
- **Performance:** Go handles real-time message processing, Python handles AI integration

**Critical Operational Requirement:** The Go WhatsApp Bridge must run continuously. If it stops, you lose WhatsApp connectivity and must re-authenticate. The autostart script ensures this doesn't happen on system reboot.

---

## Repository Information

- **GitHub Repository:** [lharries/whatsapp-mcp](https://github.com/lharries/whatsapp-mcp)
- **Protocol:** Uses official WhatsApp Web multi-device API via `whatsmeow` library
- **License:** Check repository for current licensing terms
- **Updates:** Monitor repository for updates and security patches
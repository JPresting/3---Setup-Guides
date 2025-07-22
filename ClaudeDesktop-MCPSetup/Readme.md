Here's the complete updated README file with the n8n section added:

# Claude Desktop MCP Setup Guide

A comprehensive guide to setting up Model Context Protocol (MCP) servers with Claude Desktop on Windows.

## Prerequisites

### Required Software
- **Claude Desktop** (latest version)
- **Node.js** (v18+) - for npm-based MCP servers
- **Docker Desktop** - for Docker-based MCP servers

### Optional
- **WSL2** - for Linux-based development environments

## Initial Setup

### Enable Developer Mode (REQUIRED)
Before you can configure any MCP servers, you must enable Developer Mode:

1. **Open Claude Desktop**
2. **Click "File" in the menu bar**
3. **Select "Settings"**
4. **Click "Developer" in the left sidebar**
5. **The "Edit Config" button will now appear**

**Without Developer Mode**: No MCP configuration is possible - the config file and logs are not accessible.

## Configuration File Location

After enabling Developer Mode, you can access your configuration file:
```
%APPDATA%\Claude\claude_desktop_config.json
```

Click the **"Edit Config"** button in Developer settings to open this file.

## Basic Configuration Structure

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx|docker|node",
      "args": ["server-specific-arguments"],
      "env": {
        "ENVIRONMENT_VARIABLES": "values"
      }
    }
  }
}
```

## Server Types

### NPX-Based Servers
For npm package servers (Slack, Apify, Make.com, etc.):

```json
"slack": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-slack"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "SLACK_BOT_TOKEN": "your-bot-token",
    "SLACK_TEAM_ID": "your-team-id"
  }
}
```

### Docker-Based Servers
For containerized servers (GitHub, custom Docker images):

```json
"github": {
  "command": "docker",
  "args": [
    "run", "-i", "--rm", "-e", "GITHUB_PERSONAL_ACCESS_TOKEN", "mcp/github"
  ],
  "env": {
    "GITHUB_PERSONAL_ACCESS_TOKEN": "your-github-token"
  }
}
```

### Remote SSE Servers
For remote Server-Sent Events endpoints:

```json
"n8n-email": {
  "command": "npx",
  "args": [
    "-y", "supergateway", "--sse", "https://your-server.com/mcp/endpoint/sse"
  ]
}
```

## Critical Windows Fix: APPDATA Environment Variable

**Problem**: Claude Desktop on Windows doesn't properly expand `${APPDATA}` variables, causing npm-based servers to fail.

**Solution**: Always include the expanded APPDATA path in your environment variables:

```json
"env": {
  "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
  "OTHER_ENV_VARS": "values"
}
```

Replace `YOUR_USERNAME` with your actual Windows username.

## Node.js Configuration

### Use Claude's Built-in Node.js (RECOMMENDED)
1. Open Claude Desktop Settings
2. Go to Developer → Extensions  
3. Enable "Use built-in Node.js for MCP"
4. **This handles Node.js installation and PATH issues automatically**

**Note**: Even with built-in Node.js, you still need the APPDATA fix for Windows!

## Docker Setup

### Installation & Configuration
1. **Install Docker Desktop** from [docker.com](https://docker.com)
2. **Enable Auto-start**:
   - Open Docker Desktop Settings
   - General → "Start Docker Desktop when you log in"
3. **Verify Installation**:
   ```bash
   docker --version
   docker ps
   ```

### Docker + WSL2 (Optional)
For improved performance and Linux compatibility:
1. Install WSL2: `wsl --install`
2. Enable WSL2 integration in Docker Desktop
3. Use WSL2 backend for better container performance

## Server Configuration Examples

Most MCP servers only require simple API keys, but some require more complex setup procedures.

### Simple API Key Servers
Most servers work with just an API key:
```json
"server-name": {
  "command": "npx",
  "args": ["-y", "package-name"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "API_KEY": "your-simple-api-key-here"
  }
}
```

### Advanced Setup Servers

Some servers require additional configuration steps beyond simple API keys.

#### n8n Workflow Automation Setup
**Requirements:** n8n instance URL + API key + 3 different MCP servers for complete functionality

n8n integration requires three separate MCP servers because each serves a different purpose:
- **n8n-mcp**: Direct API access for creating, editing, and managing workflows
- **n8n-workflows**: Provides workflow templates and examples
- **docs-mcp-server**: Ensures Claude has up-to-date n8n documentation

**Getting your n8n credentials:**

1. **n8n Instance URL**: 
   - If self-hosted: Your n8n instance URL (e.g., `https://n8n.yourdomain.com`)
   - If using n8n Cloud: Your cloud instance URL (e.g., `https://your-instance.n8n.cloud`)

2. **API Key Generation**:
   - Open your n8n instance
   - Click on your user avatar (top right)
   - Go to "Settings" → "API"
   - Click "Create an API Key"
   - Give it a descriptive name (e.g., "Claude MCP Integration")
   - Copy the generated API key immediately (you won't see it again!)

3. **Configuration** (all three servers needed for full functionality):
```json
"n8n-mcp": {
  "command": "npx",
  "args": ["n8n-mcp"],
  "env": {
    "MCP_MODE": "stdio",
    "LOG_LEVEL": "error",
    "DISABLE_CONSOLE_OUTPUT": "true",
    "N8N_API_URL": "https://your-n8n-instance.com",
    "N8N_API_KEY": "your-n8n-api-key-here"
  }
},
"n8n-workflows": {
  "command": "npx",
  "args": [
    "mcp-remote",
    "https://gitmcp.io/Zie619/n8n-workflows"
  ]
},
"docs-mcp-server": {
  "command": "npx",
  "args": [
    "mcp-remote",
    "https://gitmcp.io/arabold/docs-mcp-server"
  ]
}
```

**Why three servers?**
- **n8n-mcp**: Provides direct API access to your n8n instance for real-time workflow management
- **n8n-workflows**: Offers pre-built workflow templates and examples that Claude can reference
- **docs-mcp-server**: Keeps Claude updated with the latest n8n documentation, ensuring accurate assistance

**Note**: You can use just the n8n-mcp server if you only need basic workflow management, but having all three provides the best experience.

#### Google Maps API Setup
**Requirements:** Multiple API activations + API key

1. **Create API Key**: [Google Cloud Console](https://console.cloud.google.com) → APIs & Services → Credentials → "Create Credentials" → "API key"

2. **Enable Required APIs** in Google Cloud Console → APIs & Services → Library:
   - Maps JavaScript API
   - Places API  
   - Geocoding API
   - Routes API
   - Elevation API
   - Distance Matrix API

3. **Configuration:**
```json
"google-maps": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-google-maps"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "GOOGLE_MAPS_API_KEY": "your-google-maps-api-key"
  }
}
```

#### Google Calendar OAuth2 Setup
**Requirements:** OAuth2 credentials + browser authentication + API activation

1. **Enable Google Calendar API**: [Google Cloud Console](https://console.cloud.google.com) → APIs & Services → Library → Search "Google Calendar API" → Enable

2. **Create OAuth2 Credentials**: APIs & Services → Credentials → "CREATE CREDENTIALS" → "OAuth client ID" → "Desktop application" → Name: "Claude Calendar MCP" → CREATE

3. **Download JSON file** and save as `gcp-oauth.keys.json`

4. **Setup Authentication:**
```cmd
# Create directory and save JSON file
mkdir .google-calendar
# Move gcp-oauth.keys.json to: C:\Users\YOUR_USERNAME\.google-calendar\

# Navigate to directory
cd C:\Users\YOUR_USERNAME\.google-calendar

# Set environment variable and authenticate
set GOOGLE_OAUTH_CREDENTIALS=C:\Users\YOUR_USERNAME\.google-calendar\gcp-oauth.keys.json
npx @cocal/google-calendar-mcp auth
```

5. **Configuration:**
```json
"google-calendar": {
  "command": "npx",
  "args": ["-y", "@cocal/google-calendar-mcp"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "GOOGLE_OAUTH_CREDENTIALS": "C:\\Users\\YOUR_USERNAME\\.google-calendar\\gcp-oauth.keys.json"
  }
}
```

**Note**: Provides full calendar management (create/edit/delete events, find free time, manage multiple calendars).

#### Gmail OAuth2 Setup
**Requirements:** OAuth2 credentials + browser authentication

1. **Create OAuth2 Credentials**: [Google Cloud Console](https://console.cloud.google.com) → APIs & Services → Credentials → "Create Credentials" → "OAuth client ID" → "Desktop application"

2. **Enable Gmail API**: APIs & Services → Library → Search "Gmail API" → Enable

3. **Download JSON file** and save as `gcp-oauth.keys.json`

4. **Setup Authentication:**
```cmd
# Create directory
mkdir .gmail-mcp

# Move JSON file to: C:\Users\YOUR_USERNAME\.gmail-mcp\gcp-oauth.keys.json

# Run authentication (opens browser)
npx @gongrzhe/server-gmail-autoauth-mcp auth
```

5. **Configuration:**
```json
"gmail": {
  "command": "npx",
  "args": ["@gongrzhe/server-gmail-autoauth-mcp"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\"
  }
}
```

**Note**: OAuth2 setup requires one-time browser authentication but provides full Gmail access (send/receive emails, manage labels, search, etc.).

## Complete Example Configuration

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "C:\\Users\\YOUR_USERNAME\\Documents"],
      "env": {
        "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\"
      }
    },
    "github": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm", "-e", "GITHUB_PERSONAL_ACCESS_TOKEN", "mcp/github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_github_token_here"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
        "SLACK_BOT_TOKEN": "xoxb-your-slack-bot-token",
        "SLACK_TEAM_ID": "T0123456789"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
        "BRAVE_API_KEY": "your-brave-api-key"
      }
    }
  }
}
```

## Troubleshooting

### Common Issues

**"Server disconnected" errors**:
1. Check Claude Desktop logs: `%APPDATA%\Claude\logs\`
2. Verify all environment variables are set
3. Ensure APPDATA is explicitly defined for npm servers

**Docker servers not starting**:
1. Verify Docker Desktop is running
2. Check if the Docker image exists: `docker images`
3. Test manually: `docker run -it image-name`

**NPX servers failing**:
- **Root cause**: Windows doesn't expand `${APPDATA}` variables in Claude Desktop (affects both built-in and system Node.js)
- **Fix**: Add explicit APPDATA path to every npm server's env section:
  ```json
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "YOUR_API_KEY": "value"
  }
  ```
- **Note**: This fix is required even when using Claude's built-in Node.js

### Debugging Steps

1. **Check Claude Desktop logs**: `%APPDATA%\Claude\logs\`
2. **Look for APPDATA errors** in logs like: `ENOENT: no such file or directory, lstat '...\${APPDATA}'`
3. **Add APPDATA to env section** for any failing npm server
4. **Restart Claude Desktop** after config changes

## Security Notes

- Keep API keys and tokens secure
- Never commit configuration files with secrets to version control
- Use environment variables or secure credential storage
- Regularly rotate API keys and access tokens

## Getting Help

- **Official Documentation**: [Model Context Protocol](https://modelcontextprotocol.io)
- **Claude Desktop Logs**: `%APPDATA%\Claude\logs\`

## Server Ecosystem

Popular MCP servers include:
- **@modelcontextprotocol/server-filesystem** - File system access
- **@modelcontextprotocol/server-slack** - Slack integration
- **@apify/actors-mcp-server** - Web scraping via Apify
- **@modelcontextprotocol/server-brave-search** - Web search
- **mcp/github** - GitHub repository access

---

*This guide covers Windows-specific setup. For macOS/Linux, the APPDATA workaround is not needed.*
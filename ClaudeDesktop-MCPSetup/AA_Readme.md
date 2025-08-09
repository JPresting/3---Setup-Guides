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

## Server Types & Setup Guides

### General Setup Methods

**For detailed setup instructions for each server type, see the dedicated guides:**

- **[Simple API Key Servers](SimpleAPIKey-MCP-Setup.md)** - Most common type (Slack, Brave Search, Apify, etc.)
- **[Docker-Based Servers](Docker-MCP-Setup.md)** - Containerized servers (GitHub, databases, custom services)
- **[Remote SSE Servers](RemoteSSE-MCP-Setup.md)** - Cloud-hosted servers via Server-Sent Events

### Specific Platform Integrations

**For platform-specific setup instructions, see the `/Useful-MCP-Servers/` directory:**

- **[Gmail Integration](Useful-MCP-Servers/Gmail-MCP-Setup.md)** - OAuth2 email management
- **[Google Calendar](Useful-MCP-Servers/GoogleCalendar-MCP-Setup.md)** - OAuth2 calendar operations  
- **[Google Maps API](Useful-MCP-Servers/GoogleMaps-MCP-Setup.md)** - Location services and mapping
- **[WhatsApp Integration](Useful-MCP-Servers/WhatsApp-MCP-Setup.md)** - Personal WhatsApp messaging
- **[n8n Workflow Automation](Useful-MCP-Servers/n8n-MCP-Setup.md)** - n8n instance integration

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
2. **Look for APPDATA errors** in logs like: `ENOENT: no such file or directory, lstat '...\\${APPDATA}'`
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


**Need a more advanced deployment, integration, or enterprise automation?** Visit [Stardawnai.com](https://stardawnai.com) for professional consulting and development on AI-driven process automation, SAP integration, self-hosted enterprise N8N workflows, and custom hybrid infrastructure solutions tailored to your business needs.
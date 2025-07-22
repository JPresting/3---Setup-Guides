# n8n Workflow Automation MCP Setup

**Requirements:** n8n instance URL + API key + 3 different MCP servers for complete functionality

n8n integration requires three separate MCP servers because each serves a different purpose:
- **n8n-mcp**: Direct API access for creating, editing, and managing workflows
- **n8n-workflows**: Provides workflow templates and examples
- **docs-mcp-server**: Ensures Claude has up-to-date n8n documentation

## Getting Your n8n Credentials

### 1. n8n Instance URL
- **If self-hosted**: Your n8n instance URL (e.g., `https://n8n.yourdomain.com`)
- **If using n8n Cloud**: Your cloud instance URL (e.g., `https://your-instance.n8n.cloud`)

### 2. API Key Generation
1. Open your n8n instance
2. Click on your **user avatar** (top right)
3. Go to **"Settings"** → **"API"**
4. Click **"Create an API Key"**
5. Give it a descriptive name (e.g., "Claude MCP Integration")
6. **Copy the generated API key immediately** (you won't see it again!)

## Configuration

Add all three servers to your `claude_desktop_config.json` for full functionality:

```json
"n8n-mcp": {
  "command": "npx",
  "args": ["n8n-mcp"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
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

## Why Three Servers?

### n8n-mcp Server
- **Purpose**: Provides direct API access to your n8n instance for real-time workflow management
- **Capabilities**: Create, edit, execute, and delete workflows directly from Claude
- **Requirements**: Your n8n instance URL and API key

### n8n-workflows Server
- **Purpose**: Offers pre-built workflow templates and examples that Claude can reference
- **Capabilities**: Browse community workflow templates, get inspiration, copy proven patterns
- **Requirements**: No additional setup - uses remote repository

### docs-mcp-server Server
- **Purpose**: Keeps Claude updated with the latest n8n documentation
- **Capabilities**: Ensures Claude provides accurate assistance based on current n8n features
- **Requirements**: No additional setup - uses remote repository

## Simplified Setup Option

**If you only need basic workflow management**, you can use just the n8n-mcp server:

```json
"n8n-mcp": {
  "command": "npx",
  "args": ["n8n-mcp"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "MCP_MODE": "stdio",
    "LOG_LEVEL": "error",
    "DISABLE_CONSOLE_OUTPUT": "true",
    "N8N_API_URL": "https://your-n8n-instance.com",
    "N8N_API_KEY": "your-n8n-api-key-here"
  }
}
```

**However, having all three provides the best experience** with templates and up-to-date documentation.

## Environment Variables Explained

- **MCP_MODE**: Set to "stdio" for Claude Desktop compatibility
- **LOG_LEVEL**: Set to "error" to reduce console noise
- **DISABLE_CONSOLE_OUTPUT**: Set to "true" to prevent terminal spam
- **N8N_API_URL**: Your complete n8n instance URL (with https://)
- **N8N_API_KEY**: The API key generated from your n8n settings

## Capabilities After Setup

With all three servers configured, Claude can:

**Direct Workflow Management** (via n8n-mcp):
- Create new workflows from scratch
- Edit existing workflows
- Execute workflows manually
- Monitor workflow execution status
- Manage workflow settings and permissions

**Template Access** (via n8n-workflows):
- Browse pre-built workflow templates
- Get suggestions for common automation patterns
- Copy and adapt community workflows
- Learn n8n best practices

**Documentation Support** (via docs-mcp-server):
- Get up-to-date information about n8n nodes
- Understand current n8n features and limitations
- Receive accurate guidance on n8n configuration

## Troubleshooting

### API Key Issues
- **Invalid API Key**: Regenerate the key in n8n settings
- **Permissions**: Ensure the API key has sufficient permissions for workflow management

### Connection Problems
- **URL Format**: Ensure URL includes `https://` and no trailing slash
- **Network Access**: Verify Claude can reach your n8n instance
- **Firewall**: Check if firewall is blocking connections

### Server Not Starting
- **APPDATA Path**: Ensure the APPDATA environment variable is correctly set
- **Dependencies**: The npx command will automatically install required packages

**Remember**: Replace `YOUR_USERNAME` with your actual Windows username and update the n8n URL and API key with your real credentials.
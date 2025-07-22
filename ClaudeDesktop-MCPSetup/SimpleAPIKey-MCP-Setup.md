# Simple API Key MCP Servers Setup

**Requirements:** API key from service provider + npm package installation

Most MCP servers require only a simple API key for authentication. This template covers the general setup pattern for API key-based services like Slack, Brave Search, Apify, and similar services.

## General Setup Pattern

### Step 1: Obtain API Key

Each service has its own process for API key generation:

**Common patterns**:
- Sign up for the service (if not already registered)
- Navigate to account settings or developer section
- Create/generate new API key
- Copy and securely store the API key

### Step 2: Claude Desktop Configuration

Basic template for API key-based servers:

```json
"server-name": {
  "command": "npx",
  "args": ["-y", "package-name"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "API_KEY_NAME": "your-api-key-here"
  }
}
```

**Replace**:
- `server-name`: Descriptive name for the server (e.g., "slack", "brave-search")
- `package-name`: npm package name (e.g., "@modelcontextprotocol/server-slack")
- `API_KEY_NAME`: Environment variable name expected by the server
- `your-api-key-here`: Your actual API key from the service
- `YOUR_USERNAME`: Your Windows username

## Common API Key Servers

### Slack Integration
**API Key Source**: Slack App configuration with Bot User OAuth Token

```json
"slack": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-slack"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "SLACK_BOT_TOKEN": "xoxb-your-slack-bot-token",
    "SLACK_TEAM_ID": "T0123456789"
  }
}
```

**Setup Steps**:
1. Create Slack App at [api.slack.com](https://api.slack.com/apps)
2. Add Bot Token Scopes (channels:read, chat:write, etc.)
3. Install app to workspace
4. Copy "Bot User OAuth Token" from OAuth & Permissions

### Brave Search API
**API Key Source**: Brave Search API Developer Portal

```json
"brave-search": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-brave-search"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "BRAVE_API_KEY": "your-brave-api-key"
  }
}
```

**Setup Steps**:
1. Register at [brave.com/search/api](https://brave.com/search/api)
2. Create API key from dashboard
3. Note usage limits and pricing

### Apify Web Scraping
**API Key Source**: Apify Console Account Settings

```json
"apify": {
  "command": "npx",
  "args": ["-y", "@apify/actors-mcp-server"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "APIFY_TOKEN": "apify_api_your_token_here"
  }
}
```

**Setup Steps**:
1. Create account at [apify.com](https://apify.com)
2. Go to Settings → Integrations → API tokens
3. Create new token with appropriate permissions

### Make.com Automation
**API Key Source**: Make.com Profile Settings

```json
"make": {
  "command": "npx",
  "args": ["-y", "@makehq/mcp-server"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "MAKE_API_KEY": "your-make-api-key",
    "MAKE_ZONE": "eu2.make.com",
    "MAKE_TEAM": "your-team-id"
  }
}
```

**Setup Steps**:
1. Access Make.com account settings
2. Generate API token
3. Note your region (zone) and team ID

### YouTube Music
**API Key Source**: Google Cloud Console (YouTube Data API)

```json
"youtube-music": {
  "command": "npx",
  "args": ["-y", "@instructa/mcp-youtube-music"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "YOUTUBE_API_KEY": "your-youtube-api-key"
  }
}
```

**Setup Steps**:
1. Create Google Cloud project
2. Enable YouTube Data API v3
3. Create API key with YouTube API restrictions

## Critical Windows Configuration

### APPDATA Environment Variable
**Why Required**: Windows doesn't properly expand `${APPDATA}` variables in Claude Desktop, causing npm-based servers to fail.

**Solution**: Always include the explicit APPDATA path:
```json
"env": {
  "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
  "YOUR_API_KEY": "value"
}
```

**Replace `YOUR_USERNAME`** with your actual Windows username.

## Security Best Practices

### API Key Security
1. **Never commit to version control**: Keep API keys out of git repositories
2. **Use environment variables**: Store keys separately from code when possible
3. **Regular rotation**: Change API keys periodically
4. **Minimum permissions**: Grant only necessary API permissions
5. **Monitor usage**: Track API key usage for unusual activity

### Access Control
- **Restrict by IP**: If service supports it, limit API key to your IP address
- **Rate limiting**: Be aware of and respect API rate limits
- **Scope limitations**: Use most restrictive scopes that still allow functionality

## Troubleshooting

### Common Issues

**"Server disconnected" or startup failures**:
1. **Check APPDATA**: Ensure APPDATA environment variable is set correctly
2. **Verify API key**: Confirm API key is valid and has proper permissions
3. **Package installation**: npm will automatically install packages on first run
4. **Network connectivity**: Ensure services are reachable from your network

**"Invalid API key" errors**:
- **Copy carefully**: Ensure no extra spaces when copying API keys
- **Check expiration**: Some API keys expire and need renewal
- **Permissions**: Verify API key has required permissions for the service
- **Format**: Some services have specific API key formats (prefixes, etc.)

**Rate limiting or quota exceeded**:
- **Usage limits**: Check service documentation for API limits
- **Billing**: Some services require billing setup even for free tiers
- **Throttling**: Implement delays between requests if hitting limits

### Testing Your Setup

After configuration, test with service-specific commands:

**Slack**: `"List my Slack channels"`
**Brave Search**: `"Search the web for 'Claude AI news'"`
**Apify**: `"Scrape the homepage of example.com"`

## Finding New API Key Servers

### Discovery Resources
- **MCP Server Registry**: [modelcontextprotocol.io](https://modelcontextprotocol.io)
- **npm packages**: Search for packages with "mcp" or "model-context-protocol"
- **GitHub**: Search repositories with "MCP server" or "Model Context Protocol"
- **Community**: Claude Desktop community forums and Discord

### Evaluating New Servers
Before adding new API key servers:
1. **Check documentation**: Ensure setup instructions are clear
2. **Verify maintenance**: Look for active development and recent updates  
3. **Review permissions**: Understand what access the server requires
4. **Test safely**: Try with minimal permissions first
5. **Community feedback**: Look for user reviews and experiences

## Template for Custom Services

When adding a new API key-based service, use this template:

```json
"custom-service": {
  "command": "npx",
  "args": ["-y", "@organization/mcp-server-name"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "SERVICE_API_KEY": "your-service-api-key",
    "ADDITIONAL_CONFIG": "value-if-needed"
  }
}
```

Replace the placeholders with service-specific values and consult the server's documentation for required environment variables.

**Remember**: Most MCP servers follow this pattern, making it easy to add new services as they become available in the ecosystem.
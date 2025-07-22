# Gmail OAuth2 MCP Setup

**Requirements:** Google Cloud Console account + OAuth2 credentials + browser authentication

The Gmail MCP server provides full email management capabilities through Claude Desktop, including sending/receiving emails, managing labels, searching messages, and handling attachments.

## Prerequisites

- Google Cloud Console account (free)
- Gmail account for email access
- Browser access for one-time OAuth authentication

## Step 1: Enable Gmail API

1. **Open Google Cloud Console**: Go to [console.cloud.google.com](https://console.cloud.google.com)

2. **Select or Create Project**:
   - Click the project dropdown at the top
   - Select existing project or click "New Project"
   - Give it a name like "Claude Gmail Integration"

3. **Enable Gmail API**:
   - Go to **APIs & Services** → **Library**
   - Search for "Gmail API"
   - Click on "Gmail API" from results
   - Click **"Enable"** button

## Step 2: Create OAuth2 Credentials

1. **Navigate to Credentials**:
   - Go to **APIs & Services** → **Credentials**

2. **Configure OAuth Consent Screen** (if first time):
   - Click **"OAuth consent screen"** tab
   - Choose **"External"** (unless you have Google Workspace)
   - Fill required fields:
     - App name: "Claude Gmail MCP"
     - User support email: Your email
     - Developer contact information: Your email
   - **Add Scopes** (click "Add or Remove Scopes"):
     - `https://www.googleapis.com/auth/gmail.readonly` - Read emails
     - `https://www.googleapis.com/auth/gmail.send` - Send emails
     - `https://www.googleapis.com/auth/gmail.modify` - Modify emails (labels, etc.)
   - Click **"Save and Continue"** through all steps

3. **Create OAuth2 Client ID**:
   - Go back to **"Credentials"** tab
   - Click **"CREATE CREDENTIALS"** → **"OAuth client ID"**
   - Choose **"Desktop application"**
   - Name: "Claude Gmail MCP"
   - Click **"CREATE"**

4. **Download Credentials File**:
   - Click the **download button** (⬇) next to your new OAuth client
   - Save the JSON file as `gcp-oauth.keys.json`

## Step 3: Setup Authentication Directory

Create a dedicated directory for Gmail MCP credentials:

```powershell
# Create the directory
mkdir C:\Users\YOUR_USERNAME\.gmail-mcp

# Move the downloaded JSON file to this directory
# File should be named: gcp-oauth.keys.json
```

**File location should be**: `C:\Users\YOUR_USERNAME\.gmail-mcp\gcp-oauth.keys.json`

## Step 4: Authenticate with Gmail

Run the authentication process to get your OAuth tokens:

```cmd
# Navigate to your home directory or any location
cd C:\Users\YOUR_USERNAME

# Set environment variable pointing to your credentials (optional for auto-auth version)
set GOOGLE_OAUTH_CREDENTIALS=C:\Users\YOUR_USERNAME\.gmail-mcp\gcp-oauth.keys.json

# Run authentication (this will open your browser)
npx @gongrzhe/server-gmail-autoauth-mcp auth
```

**What happens during authentication**:
1. A browser window opens with Google OAuth login
2. Sign in with your Gmail account
3. Grant permissions for Gmail access (read, send, modify)
4. Review and accept the requested permissions
5. Browser shows "Authentication successful" message
6. Close browser - authentication tokens are now saved locally

## Step 5: Claude Desktop Configuration

Add this configuration to your `claude_desktop_config.json`:

```json
"gmail": {
  "command": "npx",
  "args": ["@gongrzhe/server-gmail-autoauth-mcp"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\"
  }
}
```

**Replace**:
- `YOUR_USERNAME` with your actual Windows username

**Note**: This MCP server automatically handles OAuth credentials without needing to specify the credentials file path in the configuration.

## Available Gmail Capabilities

After setup, Claude can provide comprehensive email management:

### Email Reading & Searching
- **Read Emails**: Access email content, headers, and metadata
  - "Show me my latest emails"
  - "Read the email from John about the project"
- **Search Messages**: Find emails by sender, subject, content, or date
  - "Find all emails from my manager this week"
  - "Search for emails containing 'invoice' in the last month"
  - "Show me unread emails with attachments"

### Email Composition & Sending
- **Send Emails**: Compose and send new messages
  - "Send an email to sarah@company.com about tomorrow's meeting"
  - "Email the team about the project update"
- **Reply to Messages**: Respond to existing email threads
  - "Reply to that email from Mike with confirmation"
- **Forward Messages**: Forward emails to other recipients
  - "Forward John's email to the entire team"

### Email Organization
- **Label Management**: Create, apply, and remove Gmail labels
  - "Create a new label called 'Urgent Projects'"
  - "Add the 'Important' label to emails from my boss"
  - "Remove the 'Inbox' label from processed emails"
- **Archive/Delete**: Organize mailbox by archiving or deleting messages
  - "Archive all emails older than 3 months"
  - "Delete promotional emails from last week"

### Attachment Handling
- **Download Attachments**: Save email attachments to local storage
  - "Download the PDF attachment from yesterday's contract email"
  - "Save all images from Sarah's email to my Documents folder"
- **Send with Attachments**: Include files when sending emails
  - "Email the report with the Excel file attached"

### Advanced Email Operations
- **Batch Operations**: Process multiple emails at once
  - "Mark all emails from newsletters as read"
  - "Move all emails from 'Promotions' to a custom label"
- **Email Analysis**: Get insights about email patterns
  - "How many emails did I receive this week?"
  - "Who are my most frequent email contacts?"

## Usage Examples

### Basic Email Management
```
"Show me my inbox with the latest 10 emails"
"Read the most recent email from my manager"
"Send a thank you email to jane@company.com"
"Archive all read emails from last month"
"Create a new label for client communications"
```

### Advanced Email Operations
```
"Find all emails with 'budget' in the subject from the last quarter"
"Download all PDF attachments from emails this week"
"Reply to the email thread about the project timeline"
"Forward the meeting agenda to all team members"
"Search for emails from domain '@supplier.com' with attachments"
```

### Email Productivity
```
"Mark all promotional emails as read and archive them"
"Find emails I need to respond to (starred or flagged)"
"Create a summary of all project-related emails from this week"
"Set up labels for different clients and organize recent emails"
"Find the latest email with contract documents"
```

## Authentication & Security

### OAuth2 Flow Details
- **One-time setup**: Authentication persists until tokens expire
- **Secure storage**: Tokens stored locally in system credential store
- **Auto-refresh**: Automatic token refresh when needed
- **Scope limitations**: Only Gmail permissions granted (no access to other Google services)

### Gmail Permissions
The OAuth process grants these specific permissions:
- **Read access**: View email messages, labels, and metadata
- **Send access**: Compose and send emails on your behalf
- **Modify access**: Change labels, archive/delete messages, mark as read/unread
- **Attachment access**: Download and upload email attachments

## Troubleshooting

### Common Issues

**"Authentication failed" or browser doesn't open**:
- Try running the auth command from a standard command prompt (not PowerShell)
- Ensure no browser pop-up blockers are interfering
- Check that the `gcp-oauth.keys.json` file is in the correct location

**"Invalid credentials" or "Access denied"**:
- Verify Gmail API is enabled in Google Cloud Console
- Check OAuth consent screen configuration
- Ensure proper scopes are added (gmail.readonly, gmail.send, gmail.modify)
- Confirm the credentials file isn't corrupted (re-download if needed)

**"Quota exceeded" or rate limiting**:
- Gmail API has usage limits (typically generous for personal use)
- Check quota usage in Google Cloud Console → APIs & Services → Quotas
- Implement delays between batch operations if hitting limits

**Emails not showing up or incomplete results**:
- Verify you're authenticated with the correct Gmail account
- Check if emails might be in different labels/folders
- Some emails might be filtered out by Gmail's API (spam, etc.)

### Re-authentication Process

If you need to re-authenticate (tokens expired or corrupted):

```cmd
# Clear existing authentication
del C:\Users\YOUR_USERNAME\.gmail-mcp\*.token
del C:\Users\YOUR_USERNAME\.gmail-mcp\token.json

# Run authentication again
npx @gongrzhe/server-gmail-autoauth-mcp auth
```

### Testing Your Setup

After configuration, test with simple commands:

```
"Show me my latest 5 emails"
"What's in my inbox right now?"
"Send a test email to myself"
```

## Security Best Practices

### OAuth Credential Security
1. **Protect credential files**: Keep `gcp-oauth.keys.json` secure and never share
2. **Monitor app permissions**: Regularly review connected apps in your Google Account settings
3. **Rotate credentials**: Consider regenerating OAuth credentials periodically
4. **Revoke access**: Can revoke access anytime through Google Account → Security → Third-party access

### Email Privacy
- **Local processing**: All operations happen locally between Claude and Gmail
- **No data storage**: Claude doesn't store your emails permanently
- **Request-based access**: Emails only accessed when you explicitly ask Claude
- **Secure transmission**: All communication with Gmail uses HTTPS encryption

## Advanced Configuration

### Custom Scopes (Advanced Users)
If you need different permission levels, you can modify the OAuth scopes in Google Cloud Console:
- `gmail.readonly` - Read-only access
- `gmail.send` - Send emails only
- `gmail.modify` - Full access except delete permanently
- `gmail.compose` - Create drafts and send

### Multiple Gmail Accounts
For multiple Gmail accounts, you'll need separate MCP server configurations with different credential files and authentication processes.

**Support**: For OAuth or API issues, consult [Gmail API Documentation](https://developers.google.com/gmail/api) or Google Cloud Support.
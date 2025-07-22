# Google Calendar OAuth2 MCP Setup

**Requirements:** Google Cloud Console account + OAuth2 credentials + browser authentication + API activation

The Google Calendar MCP server provides full calendar management capabilities through Claude Desktop, including creating, editing, and deleting events, finding free time, and managing multiple calendars.

## Prerequisites

- Google Cloud Console account (free)
- Gmail/Google account for calendar access
- Browser access for one-time OAuth authentication

## Step 1: Enable Google Calendar API

1. **Open Google Cloud Console**: Go to [console.cloud.google.com](https://console.cloud.google.com)

2. **Select or Create Project**:
   - Click the project dropdown at the top
   - Select existing project or click "New Project"
   - Give it a name like "Claude Calendar Integration"

3. **Enable Calendar API**:
   - Go to **APIs & Services** → **Library**
   - Search for "Google Calendar API"
   - Click on "Google Calendar API" from results
   - Click **"Enable"** button

## Step 2: Create OAuth2 Credentials

1. **Navigate to Credentials**:
   - Go to **APIs & Services** → **Credentials**

2. **Configure OAuth Consent Screen** (if first time):
   - Click **"OAuth consent screen"** tab
   - Choose **"External"** (unless you have Google Workspace)
   - Fill required fields:
     - App name: "Claude Calendar MCP"
     - User support email: Your email
     - Developer contact information: Your email
   - Click **"Save and Continue"** through all steps

3. **Create OAuth2 Client ID**:
   - Go back to **"Credentials"** tab
   - Click **"CREATE CREDENTIALS"** → **"OAuth client ID"**
   - Choose **"Desktop application"**
   - Name: "Claude Calendar MCP"
   - Click **"CREATE"**

4. **Download Credentials File**:
   - Click the **download button** (⬇) next to your new OAuth client
   - Save the JSON file as `gcp-oauth.keys.json`

## Step 3: Setup Authentication Directory

Create a dedicated directory for Google Calendar credentials:

```powershell
# Create the directory
mkdir C:\Users\YOUR_USERNAME\.google-calendar

# Move the downloaded JSON file to this directory
# Rename it to: gcp-oauth.keys.json if needed
```

**File location should be**: `C:\Users\YOUR_USERNAME\.google-calendar\gcp-oauth.keys.json`

## Step 4: Authenticate with Google Calendar

Run the authentication process to get your OAuth tokens:

```cmd
# Navigate to the credentials directory
cd C:\Users\YOUR_USERNAME\.google-calendar

# Set environment variable pointing to your credentials
set GOOGLE_OAUTH_CREDENTIALS=C:\Users\YOUR_USERNAME\.google-calendar\gcp-oauth.keys.json

# Run authentication (this will open your browser)
npx @cocal/google-calendar-mcp auth
```

**What happens during authentication**:
1. A browser window opens with Google OAuth login
2. Sign in with your Google/Gmail account
3. Grant permissions for calendar access
4. Browser shows "Authentication successful" message
5. Close browser - authentication tokens are now saved locally

## Step 5: Claude Desktop Configuration

Add this configuration to your `claude_desktop_config.json`:

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

**Replace**:
- `YOUR_USERNAME` with your actual Windows username (in both places)

## Available Calendar Capabilities

After setup, Claude can provide comprehensive calendar management:

### Event Management
- **Create Events**: Schedule meetings, appointments, reminders
  - "Schedule a team meeting tomorrow at 2 PM for 1 hour"
  - "Create a dentist appointment on Friday at 10 AM"
- **Edit Events**: Modify existing events (time, date, attendees, description)
  - "Move my Monday morning meeting to Tuesday at the same time"
  - "Add John and Sarah to the project kickoff meeting"
- **Delete Events**: Remove events from calendar
  - "Cancel the meeting scheduled for Thursday afternoon"

### Calendar Browsing
- **List Events**: View upcoming or past events
  - "What meetings do I have this week?"
  - "Show me all events for next Monday"
- **Search Events**: Find events by title, description, or attendee
  - "Find all meetings with 'budget' in the title"
  - "When is my next meeting with Sarah?"

### Free Time Management
- **Find Free Time**: Locate available time slots for scheduling
  - "When am I free for a 1-hour meeting this week?"
  - "Find a 2-hour block next Tuesday afternoon"
- **Check Availability**: Verify if specific times are free
  - "Am I free tomorrow at 3 PM?"
  - "Is Friday morning available for a long meeting?"

### Multiple Calendar Support
- **Multi-Calendar Management**: Work with multiple Google calendars
  - Personal calendar, work calendar, shared calendars
  - Calendar-specific event creation and viewing
- **Calendar Selection**: Specify which calendar for operations
  - "Add this to my work calendar"
  - "Check my personal calendar for conflicts"

## Usage Examples

### Basic Event Operations
```
"Create a lunch meeting with Mike tomorrow at 12:30 PM"
"What meetings do I have this afternoon?"
"Move my 3 PM meeting to 4 PM"
"Cancel the project review meeting on Friday"
"Add a reminder to call mom on Sunday at 2 PM"
```

### Advanced Scheduling
```
"Find a 2-hour free block this week for project planning"
"Schedule a recurring weekly team standup every Monday at 10 AM"
"What's my availability between 2-5 PM next Tuesday?"
"Create an all-day event for vacation next Friday"
"Find the next available hour-long slot for a client call"
```

### Calendar Analysis
```
"How many meetings do I have scheduled this week?"
"What's my busiest day next month?"
"List all meetings with 'quarterly' in the title"
"Show me my schedule for the first week of next month"
"Find conflicts in my calendar for the next two weeks"
```

## Authentication & Security

### OAuth2 Flow Details
- **One-time setup**: Authentication persists until tokens expire
- **Secure storage**: Tokens stored locally, never sent to external servers
- **Refresh handling**: Automatic token refresh when needed
- **Scope limitations**: Only calendar permissions granted (no access to other Google services)

### Token Management
- **Token location**: Stored in `.google-calendar` directory
- **Token refresh**: Automatic when tokens expire
- **Re-authentication**: Required if tokens become invalid (rare)

## Troubleshooting

### Common Issues

**"Authentication failed" or browser doesn't open**:
- Ensure `GOOGLE_OAUTH_CREDENTIALS` environment variable is set correctly
- Verify the `gcp-oauth.keys.json` file path is correct
- Try running authentication command with full path

**"Invalid credentials" error**:
- Check that Google Calendar API is enabled in Google Cloud Console
- Verify OAuth consent screen is properly configured
- Ensure the credentials file isn't corrupted (re-download if needed)

**"Permission denied" or "Access blocked"**:
- Make sure OAuth consent screen is published (not in testing mode)
- Check that your Google account has calendar access
- Verify no organizational restrictions on your Google account

**Events not showing up**:
- Confirm you're authenticated with the correct Google account
- Check if you have multiple Google calendars and specify the right one
- Ensure calendar permissions are properly granted during OAuth flow

### Re-authentication Process

If you need to re-authenticate (tokens expired or corrupted):

```cmd
# Delete existing tokens (if any)
del C:\Users\YOUR_USERNAME\.google-calendar\*.token

# Run authentication again
cd C:\Users\YOUR_USERNAME\.google-calendar
set GOOGLE_OAUTH_CREDENTIALS=C:\Users\YOUR_USERNAME\.google-calendar\gcp-oauth.keys.json
npx @cocal/google-calendar-mcp auth
```

### Testing Your Setup

After configuration, test with simple commands:

```
"List my events for today"
"What's on my calendar tomorrow?"
"Create a test event for next hour"
```

## Privacy & Data Handling

### Local Processing
- **No cloud storage**: All operations happen locally between Claude and Google Calendar
- **Direct API access**: Claude communicates directly with Google Calendar APIs
- **Token security**: OAuth tokens stored locally, never transmitted to third parties

### Permissions Granted
The OAuth process grants these specific permissions:
- **Read calendar events**: View your calendar events and details
- **Write calendar events**: Create, modify, and delete calendar events
- **Calendar metadata**: Access calendar names, colors, and settings
- **Free/busy information**: Check availability across calendars

**Note**: No access to other Google services (Gmail, Drive, etc.) unless explicitly configured separately.

**Support**: For OAuth or API issues, consult [Google Calendar API Documentation](https://developers.google.com/calendar) or Google Cloud Support.
# Google Maps API MCP Setup

**Requirements:** Google Cloud Console account + Multiple API activations + API key

The Google Maps MCP server provides comprehensive location and mapping services through Claude Desktop, including geocoding, directions, place search, and elevation data.

## Prerequisites

- Google Cloud Console account (free tier available)
- Billing account enabled (required for Maps APIs, even with free usage)
- Credit card for account verification (Google provides free credits)

## Step 1: Create API Key

1. **Open Google Cloud Console**: Go to [console.cloud.google.com](https://console.cloud.google.com)

2. **Select or Create Project**:
   - Click the project dropdown at the top
   - Select existing project or click "New Project"
   - Give it a name like "Claude MCP Integration"

3. **Navigate to API Credentials**:
   - Go to **APIs & Services** → **Credentials**
   - Click **"Create Credentials"** → **"API key"**

4. **Copy and Secure Your API Key**:
   - Copy the generated API key immediately
   - Click **"Restrict Key"** for security (recommended)

5. **Optional - Restrict API Key** (Recommended):
   - Under "Application restrictions": Choose appropriate restriction
   - Under "API restrictions": Select "Restrict key" and choose the APIs you enabled
   - Click **"Save"**

## Step 2: Enable Required APIs

**Critical**: You must enable ALL these APIs in Google Cloud Console → **APIs & Services** → **Library**:

### Required APIs to Enable:
1. **Maps JavaScript API** - For interactive map functionality
2. **Places API** - For location search and place details
3. **Geocoding API** - For converting addresses to coordinates and vice versa
4. **Routes API** - For directions and route planning
5. **Elevation API** - For elevation data at specific locations
6. **Distance Matrix API** - For travel time and distance calculations

### How to Enable Each API:
1. In Google Cloud Console, go to **APIs & Services** → **Library**
2. Search for the API name (e.g., "Maps JavaScript API")
3. Click on the API from search results
4. Click **"Enable"** button
5. Repeat for all 6 APIs listed above

**Important**: Enabling all APIs is crucial - missing APIs will cause specific features to fail.

## Step 3: Configure Billing (Required)

Google Maps APIs require a billing account even for free usage:

1. Go to **Billing** in Google Cloud Console
2. Link a billing account or create a new one
3. **Don't worry**: Google provides free monthly usage credits for Maps APIs
4. Set up billing alerts to monitor usage (recommended)

## Step 4: Claude Desktop Configuration

Add this configuration to your `claude_desktop_config.json`:

```json
"google-maps": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-google-maps"],
  "env": {
    "APPDATA": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\",
    "GOOGLE_MAPS_API_KEY": "your-google-maps-api-key-here"
  }
}
```

**Replace**:
- `YOUR_USERNAME` with your actual Windows username
- `your-google-maps-api-key-here` with the API key you copied from Step 1

## Available Capabilities

After setup, Claude can provide these Google Maps services:

### Location Services
- **Geocoding**: Convert addresses to coordinates
  - "What are the coordinates of Times Square, NYC?"
- **Reverse Geocoding**: Convert coordinates to addresses
  - "What address is at latitude 40.7589, longitude -73.9851?"

### Place Search & Details
- **Place Search**: Find businesses, landmarks, points of interest
  - "Find coffee shops near Central Park"
  - "Search for gas stations within 5 miles of my location"
- **Place Details**: Get comprehensive information about specific places
  - "Tell me about the Empire State Building - hours, reviews, contact info"

### Directions & Routes
- **Route Planning**: Get step-by-step directions between locations
  - "Give me driving directions from Brooklyn to Manhattan"
  - "What's the walking route from my hotel to the museum?"
- **Travel Modes**: Support for driving, walking, bicycling, transit
- **Route Alternatives**: Multiple route options with time and distance

### Distance & Travel Time
- **Distance Matrix**: Calculate distances and travel times between multiple points
  - "How long does it take to drive between these 5 locations?"
  - "What are the distances between all major airports in California?"

### Elevation Data
- **Elevation Lookup**: Get elevation data for specific coordinates
  - "What's the elevation of Mount Whitney?"
  - "Show me elevation changes along this hiking trail"

## Usage Examples

```
"Find the nearest Italian restaurants to Times Square"
"Get driving directions from LAX to Hollywood Boulevard"
"What's the elevation at the coordinates 39.7392, -104.9903?"
"Calculate driving time between San Francisco and Los Angeles"
"Search for hotels within 2 miles of Golden Gate Bridge"
"Convert the address '1600 Pennsylvaniā Avenue NW, Washington, DC' to coordinates"
```

## Free Usage Limits

Google provides generous free monthly quotas:
- **Maps JavaScript API**: 28,500 map loads per month
- **Geocoding API**: 40,000 requests per month  
- **Places API**: Varies by request type (typically 2,500-100,000 per month)
- **Directions API**: 2,500 requests per month
- **Distance Matrix API**: 100 elements per month
- **Elevation API**: 2,500 requests per month

**Monitor Usage**: Set up billing alerts in Google Cloud Console to track usage.

## Security Best Practices

### API Key Security
1. **Restrict Your API Key**: 
   - Limit to specific APIs you're using
   - Set application restrictions if possible
   - Never commit API keys to version control

2. **Monitor Usage**:
   - Regular check usage in Google Cloud Console
   - Set up billing alerts
   - Review API key usage logs

3. **Rotate Keys Periodically**:
   - Generate new API keys every few months
   - Delete old/unused keys
   - Update configuration when rotating

## Troubleshooting

### Common Issues

**"API key not found" or "Invalid API key"**:
- Verify the API key is correctly copied (no extra spaces)
- Ensure the API key hasn't been deleted or restricted
- Check that billing is enabled on your Google Cloud project

**"This API has not been enabled"**:
- Return to Google Cloud Console → APIs & Services → Library
- Verify ALL 6 required APIs are enabled
- Wait 5-10 minutes after enabling APIs for changes to propagate

**"Quota exceeded" or "Rate limit exceeded"**:
- Check usage in Google Cloud Console → APIs & Services → Quotas
- Consider upgrading billing plan if exceeding free limits
- Implement request throttling in your usage patterns

**No results for place searches**:
- Try broader search terms
- Increase search radius
- Verify the location you're searching actually exists in Google Maps

### Performance Tips

1. **Cache Results**: For repeated queries, cache responses locally
2. **Batch Requests**: Use Distance Matrix API for multiple distance calculations
3. **Optimize Searches**: Use specific, targeted search terms for better results

**Support**: For API-specific issues, consult [Google Maps Platform Documentation](https://developers.google.com/maps/documentation) or Google Cloud Support.
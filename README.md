## Install as a skill

```bash
npx skills add postnifyhq/postnify-agent
```

# Postnify CLI

**Social media automation CLI for AI agents** - Schedule posts across 28+ platforms programmatically.

The Postnify CLI provides a command-line interface to the Postnify API, enabling developers and AI agents to automate social media posting, manage content, and handle media uploads across platforms like Twitter/X, LinkedIn, Reddit, YouTube, TikTok, Instagram, Facebook, and more.

---

## Installation

```bash
npm install -g postnify
# or
pnpm install -g postnify
```

---

## Authentication

### Option 1: OAuth2 (Recommended)

Authenticate using the device flow — no client ID or secret needed:

```bash
postnify auth:login
```

This will:
1. Display a one-time code in your terminal
2. Open your browser to authorize
3. Automatically save credentials to `~/.postnify/credentials.json`

```bash
# Check current auth status (verifies credentials are still valid)
postnify auth:status

# Remove stored credentials
postnify auth:logout
```

### Option 2: API Key

```bash
export POSTNIFY_API_KEY=your_api_key_here
```

**Optional:** Custom API endpoint

```bash
export POSTNIFY_API_URL=https://your-custom-api.com
```

> **Note:** OAuth2 credentials take priority over the API key when both are present.

---

## Commands

### Discovery & Settings

**List all connected integrations**
```bash
postnify integrations:list
```

Returns integration IDs, provider names, and metadata.

**Get integration settings schema**
```bash
postnify integrations:settings <integration-id>
```

Returns character limits, required settings, and available tools for fetching dynamic data.

**Trigger integration tools**
```bash
postnify integrations:trigger <integration-id> <method-name>
postnify integrations:trigger <integration-id> <method-name> -d '{"key":"value"}'
```

Fetch dynamic data like Reddit flairs, YouTube playlists, LinkedIn companies, etc.

**Examples:**
```bash
# Get Reddit flairs
postnify integrations:trigger reddit-123 getFlairs -d '{"subreddit":"programming"}'

# Get YouTube playlists
postnify integrations:trigger youtube-456 getPlaylists

# Get LinkedIn companies
postnify integrations:trigger linkedin-789 getCompanies
```

---

### Creating Posts

**Simple scheduled post**
```bash
postnify posts:create -c "Content" -s "2024-12-31T12:00:00Z" -i "integration-id"
```

**Draft post**
```bash
postnify posts:create -c "Content" -s "2024-12-31T12:00:00Z" -t draft -i "integration-id"
```

**Post with media**
```bash
postnify posts:create -c "Content" -m "img1.jpg,img2.jpg" -s "2024-12-31T12:00:00Z" -i "integration-id"
```

**Post with comments** (each comment can have its own media)
```bash
postnify posts:create \
  -c "Main post" -m "main.jpg" \
  -c "First comment" -m "comment1.jpg" \
  -c "Second comment" -m "comment2.jpg,comment3.jpg" \
  -s "2024-12-31T12:00:00Z" \
  -i "integration-id"
```

**Multi-platform post**
```bash
postnify posts:create -c "Content" -s "2024-12-31T12:00:00Z" -i "twitter-id,linkedin-id,facebook-id"
```

**Platform-specific settings**
```bash
postnify posts:create \
  -c "Content" \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"subreddit":[{"value":{"subreddit":"programming","title":"Post Title","type":"text"}}]}' \
  -i "reddit-id"
```

**Complex post from JSON file**
```bash
postnify posts:create --json post.json
```

**Options:**
- `-c, --content` - Post/comment content (use multiple times for posts with comments)
- `-s, --date` - Schedule date in ISO 8601 format (REQUIRED)
- `-t, --type` - Post type: "schedule" or "draft" (default: "schedule")
- `-m, --media` - Comma-separated media URLs for corresponding `-c`
- `-i, --integrations` - Comma-separated integration IDs (required)
- `-d, --delay` - Delay between comments in minutes (default: 0)
- `--settings` - Platform-specific settings as JSON string
- `-j, --json` - Path to JSON file with full post structure
- `--shortLink` - Use short links (default: true)

---

### Managing Posts

**List posts**
```bash
postnify posts:list
postnify posts:list --startDate "2024-01-01T00:00:00Z" --endDate "2024-12-31T23:59:59Z"
postnify posts:list --customer "customer-id"
```

Defaults to last 30 days to next 30 days if dates not specified.

**Delete post**
```bash
postnify posts:delete <post-id>
```

**Change post status (draft ↔ schedule)**
```bash
postiz posts:status <post-id> --status draft
postiz posts:status <post-id> --status schedule
```

Move a scheduled post back to a draft, or promote a draft into the publishing queue. Switching to `draft` also terminates any workflow that's already running for the post, so it won't publish. Switching to `schedule` queues the post for publishing at its stored date.

---

### Analytics

**Get platform analytics**
```bash
postnify analytics:platform <integration-id>
postnify analytics:platform <integration-id> -d 30
```

Returns metrics like followers, impressions, and engagement over time. The `-d` flag specifies days to look back (default: 7).

**Get post analytics**
```bash
postnify analytics:post <post-id>
postnify analytics:post <post-id> -d 30
```

Returns metrics like likes, comments, shares, and impressions for a specific published post.

**⚠️ If `analytics:post` returns `{"missing": true}`**, the post was published but the platform didn't return a usable post ID. Resolve it:

```bash
# 1. List available content from the provider
postnify posts:missing <post-id>

# 2. Connect the correct content to the post
postnify posts:connect <post-id> --release-id "7321456789012345678"

# 3. Analytics will now work
postnify analytics:post <post-id>
```

---

### Connecting Missing Posts

Some platforms (e.g. TikTok) don't return a post ID immediately after publishing. The post's `releaseId` is set to `"missing"` and analytics won't work until resolved.

**List available content from the provider**
```bash
postnify posts:missing <post-id>
```

**Connect a post to its published content**
```bash
postnify posts:connect <post-id> --release-id "<content-id>"
```

---

### Media Upload

**Upload file and get URL**
```bash
postnify upload <file-path>
```

**⚠️ IMPORTANT: Upload Files Before Posting**

You **must** upload media files to Postnify before using them in posts. Many platforms (especially TikTok, Instagram, and YouTube) require verified/trusted URLs and will reject external links.

**Workflow:**
1. Upload your file using `postnify upload`
2. Extract the returned URL
3. Use that URL in your post's `-m` parameter

**Supported formats:**
- **Images:** PNG, JPG, JPEG, GIF
- **Videos:** MP4

**Example:**
```bash
# 1. Upload the file first
RESULT=$(postnify upload video.mp4)
PATH=$(echo "$RESULT" | jq -r '.path')

# 2. Use the returned URL in your post
postnify posts:create -c "Check out my video!" -s "2024-12-31T12:00:00Z" -m "$PATH" -i "tiktok-id"
```

---

## Platform-Specific Features

### Reddit
```bash
# Get available flairs
postnify integrations:trigger reddit-id getFlairs -d '{"subreddit":"programming"}'

# Post with subreddit and flair
postnify posts:create \
  -c "Content" \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"subreddit":[{"value":{"subreddit":"programming","title":"My Post","type":"text","is_flair_required":true,"flair":{"id":"flair-123","name":"Discussion"}}}]}' \
  -i "reddit-id"
```

### YouTube
```bash
# Get playlists
postnify integrations:trigger youtube-id getPlaylists

# Upload video FIRST (required!)
VIDEO=$(postnify upload video.mp4)
VIDEO_URL=$(echo "$VIDEO" | jq -r '.path')

# Post with uploaded video URL
postnify posts:create \
  -c "Video description" \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"title":"Video Title","type":"public","tags":[{"value":"tech","label":"Tech"}],"playlistId":"playlist-id"}' \
  -m "$VIDEO_URL" \
  -i "youtube-id"
```

### TikTok
```bash
# Upload video FIRST (TikTok only accepts verified URLs!)
VIDEO=$(postnify upload video.mp4)
VIDEO_URL=$(echo "$VIDEO" | jq -r '.path')

# Post with uploaded video URL
postnify posts:create \
  -c "Video caption #fyp" \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"privacy":"PUBLIC_TO_EVERYONE","duet":true,"stitch":true}' \
  -m "$VIDEO_URL" \
  -i "tiktok-id"
```

### LinkedIn
```bash
# Get companies you can post to
postnify integrations:trigger linkedin-id getCompanies

# Post as company
postnify posts:create \
  -c "Company announcement" \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"companyId":"company-123"}' \
  -i "linkedin-id"
```

### X (Twitter)
```bash
# Create thread
postnify posts:create \
  -c "Thread 1/3 🧵" \
  -c "Thread 2/3" \
  -c "Thread 3/3" \
  -s "2024-12-31T12:00:00Z" \
  -d 2000 \
  -i "twitter-id"

# With reply settings
postnify posts:create \
  -c "Tweet content" \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"who_can_reply_post":"everyone"}' \
  -i "twitter-id"
```

### Instagram
```bash
# Upload image FIRST (Instagram requires verified URLs!)
IMAGE=$(postnify upload image.jpg)
IMAGE_URL=$(echo "$IMAGE" | jq -r '.path')

# Regular post
postnify posts:create \
  -c "Caption #hashtag" \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"post_type":"post"}' \
  -m "$IMAGE_URL" \
  -i "instagram-id"

# Story
STORY=$(postnify upload story.jpg)
STORY_URL=$(echo "$STORY" | jq -r '.path')

postnify posts:create \
  -c "" \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"post_type":"story"}' \
  -m "$STORY_URL" \
  -i "instagram-id"
```

**See [PROVIDER_SETTINGS.md](./PROVIDER_SETTINGS.md) for all 28+ platforms.**

---

## Features for AI Agents

### Discovery Workflow

1. **List integrations** - Get available social media accounts
2. **Get settings** - Retrieve character limits, required fields, and available tools
3. **Trigger tools** - Fetch dynamic data (flairs, playlists, boards, etc.)
4. **Create posts** - Use discovered data in posts
5. **Analyze** - Get post analytics; if `{"missing": true}` is returned, resolve with `posts:missing` + `posts:connect`

### JSON Mode
For complex posts with multiple platforms and settings:

```bash
postnify posts:create --json complex-post.json
```

JSON structure:
```json
{
  "integrations": ["twitter-123", "linkedin-456"],
  "posts": [
    {
      "provider": "twitter",
      "post": [{ "content": "Tweet version", "image": ["twitter-image.jpg"] }]
    },
    {
      "provider": "linkedin",
      "post": [{ "content": "LinkedIn version with more context...", "image": ["linkedin-image.jpg"] }],
      "settings": { "__type": "linkedin", "companyId": "company-123" }
    }
  ]
}
```

### All Output is JSON
Every command outputs JSON for easy parsing:

```bash
INTEGRATIONS=$(postnify integrations:list | jq -r '.')
REDDIT_ID=$(echo "$INTEGRATIONS" | jq -r '.[] | select(.identifier=="reddit") | .id')
```

---

## Common Workflows

### Reddit Post with Flair
```bash
#!/bin/bash
REDDIT_ID=$(postnify integrations:list | jq -r '.[] | select(.identifier=="reddit") | .id')
FLAIRS=$(postnify integrations:trigger "$REDDIT_ID" getFlairs -d '{"subreddit":"programming"}')
FLAIR_ID=$(echo "$FLAIRS" | jq -r '.output[0].id')

postnify posts:create \
  -c "My post content" \
  -s "2024-12-31T12:00:00Z" \
  --settings "{\"subreddit\":[{\"value\":{\"subreddit\":\"programming\",\"title\":\"Post Title\",\"type\":\"text\",\"is_flair_required\":true,\"flair\":{\"id\":\"$FLAIR_ID\",\"name\":\"Discussion\"}}}]}" \
  -i "$REDDIT_ID"
```

### YouTube Video Upload
```bash
#!/bin/bash
VIDEO=$(postnify upload video.mp4)
VIDEO_PATH=$(echo "$VIDEO" | jq -r '.path')

postnify posts:create \
  -c "Video description..." \
  -s "2024-12-31T12:00:00Z" \
  --settings '{"title":"My Video","type":"public","tags":[{"value":"tech","label":"Tech"}]}' \
  -m "$VIDEO_PATH" \
  -i "youtube-id"
```

### Multi-Platform Campaign
```bash
#!/bin/bash
postnify posts:create \
  -c "Same content everywhere" \
  -s "2024-12-31T12:00:00Z" \
  -m "image.jpg" \
  -i "twitter-id,linkedin-id,facebook-id"
```

### Batch Scheduling
```bash
#!/bin/bash
DATES=("2024-02-14T09:00:00Z" "2024-02-15T09:00:00Z" "2024-02-16T09:00:00Z")
CONTENT=("Monday motivation 💪" "Tuesday tips 💡" "Wednesday wisdom 🧠")

for i in "${!DATES[@]}"; do
  postnify posts:create \
    -c "${CONTENT[$i]}" \
    -s "${DATES[$i]}" \
    -i "twitter-id"
done
```

---

## API Endpoints

The CLI interacts with these Postnify API endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/public/v1/posts` | POST | Create a post |
| `/public/v1/posts` | GET | List posts |
| `/public/v1/posts/:id` | DELETE | Delete a post |
| `/public/v1/posts/:id/missing` | GET | Get missing content from provider |
| `/public/v1/posts/:id/release-id` | PUT | Update release ID for a post |
| `/public/v1/integrations` | GET | List integrations |
| `/public/v1/integration-settings/:id` | GET | Get integration settings |
| `/public/v1/integration-trigger/:id` | POST | Trigger integration tool |
| `/public/v1/analytics/:integration` | GET | Get platform analytics |
| `/public/v1/analytics/post/:postId` | GET | Get post analytics |
| `/public/v1/upload` | POST | Upload media |

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `POSTNIFY_API_KEY` | No* | - | Your Postnify API key |
| `POSTNIFY_API_URL` | No | `https://api.postnify.com` | Custom API endpoint |
| `POSTNIFY_AUTH_SERVER` | No | `https://cli-auth.postnify.com` | Custom auth server URL |

*Either OAuth2 (via `postnify auth:login`) or `POSTNIFY_API_KEY` is required.

---

## Error Handling

| Error | Solution |
|-------|----------|
| `Not authenticated` | Run `postnify auth:login` or set `POSTNIFY_API_KEY` |
| `Integration not found` | Run `integrations:list` to get valid IDs |
| `startDate/endDate required` | Use ISO 8601 format: `"2024-12-31T12:00:00Z"` |
| `Invalid settings` | Check `integrations:settings` for required fields |
| `Tool not found` | Check available tools in `integrations:settings` output |
| `Upload failed` | Verify file exists and format is supported |
| `analytics:post` returns `{"missing": true}` | Run `posts:missing <id>` then `posts:connect <id> --release-id "<rid>"` |

---

## Development

### Project Structure

```
src/
├── index.ts              # CLI entry point with yargs
├── api.ts                # Postnify API client class
├── config.ts             # Configuration (OAuth2 + API key)
└── commands/
    ├── auth.ts           # OAuth2 authentication (login/logout/status)
    ├── posts.ts          # Post management commands
    ├── integrations.ts   # Integration commands
    ├── analytics.ts      # Analytics commands
    └── upload.ts         # Media upload command
examples/                 # Example scripts and JSON files
package.json
tsconfig.json
tsup.config.ts            # Build configuration
README.md                 # This file
SKILL.md                  # AI agent reference
```

### Scripts

```bash
pnpm run dev       # Watch mode for development
pnpm run build     # Build the CLI
pnpm run start     # Run the built CLI
```

---

## Quick Reference

```bash
# Authentication
postnify auth:login                                              # OAuth2 device flow
postnify auth:status                                             # Check auth
postnify auth:logout                                             # Remove credentials
export POSTNIFY_API_KEY=your_key                                 # Or use API key

# Discovery
postnify integrations:list                           # List integrations
postnify integrations:settings <id>                  # Get settings
postnify integrations:trigger <id> <method> -d '{}'  # Fetch data

# Posting (date is required)
postnify posts:create -c "text" -s "2024-12-31T12:00:00Z" -i "id"                    # Simple
postnify posts:create -c "text" -s "2024-12-31T12:00:00Z" -t draft -i "id"          # Draft
postnify posts:create -c "text" -m "img.jpg" -s "2024-12-31T12:00:00Z" -i "id"      # With media
postnify posts:create -c "main" -c "comment" -s "2024-12-31T12:00:00Z" -i "id"      # With comment
postnify posts:create -c "text" -s "2024-12-31T12:00:00Z" --settings '{}' -i "id"   # Platform-specific
postnify posts:create --json file.json                                               # Complex

# Management
postnify posts:list                                  # List posts
postnify posts:delete <id>                          # Delete post
postnify posts:status <id> --status draft           # Move to draft (stops workflow)
postnify posts:status <id> --status schedule        # Queue draft for publishing
postnify upload <file>                              # Upload media

# Analytics
postnify analytics:platform <id>                    # Platform analytics (7 days)
postnify analytics:platform <id> -d 30             # Platform analytics (30 days)
postnify analytics:post <id>                        # Post analytics (7 days)
postnify analytics:post <id> -d 30                 # Post analytics (30 days)
# If analytics:post returns {"missing": true}, resolve it:
postnify posts:missing <id>                         # List provider content
postnify posts:connect <id> --release-id "<rid>"    # Connect content to post

# Help
postnify --help                                     # Show help
postnify posts:create --help                        # Command help
```

---

## License

AGPL-3.0

---

## Links

- **Website:** [postnify.com](https://postnify.com)
- **API Docs:** [docs.postnify.com](https://docs.postnify.com)
- **GitHub:** [postnifyhq/postnify-agent](https://github.com/postnifyhq/postnify-agent)
- **Issues:** [Report bugs](https://github.com/postnifyhq/postnify-agent/issues)

---

## Supported Platforms

28+ platforms including:

| Platform | Integration Tools | Settings |
|----------|------------------|----------|
| Twitter/X | getLists, getCommunities | who_can_reply_post |
| LinkedIn | getCompanies | companyId, carousel |
| Reddit | getFlairs, searchSubreddits | subreddit, title, flair |
| YouTube | getPlaylists, getCategories | title, type, tags, playlistId |
| TikTok | - | privacy, duet, stitch |
| Instagram | - | post_type (post/story) |
| Facebook | getPages | - |
| Pinterest | getBoards, getBoardSections | - |
| Discord | getChannels | - |
| Slack | getChannels | - |
| And 18+ more... | | |

**See [PROVIDER_SETTINGS.md](./PROVIDER_SETTINGS.md) for complete documentation.**

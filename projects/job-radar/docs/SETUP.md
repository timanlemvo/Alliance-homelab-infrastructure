# Detailed Setup Guide

This guide walks you through setting up Job Radar step-by-step. Estimated time: 15-20 minutes.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: RapidAPI Setup](#step-1-rapidapi-setup)
3. [Step 2: Discord Webhook](#step-2-discord-webhook)
4. [Step 3: n8n Configuration](#step-3-n8n-configuration)
5. [Step 4: Import and Configure](#step-4-import-and-configure)
6. [Step 5: Test and Activate](#step-5-test-and-activate)
7. [Troubleshooting](#troubleshooting)

## Prerequisites

### n8n Installation

You need n8n running. Options:

**Option A: n8n Cloud (Easiest)**
- Sign up at [n8n.io](https://n8n.io)
- Free trial available

**Option B: Self-Hosted with Docker**
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

**Option C: Docker Compose**
```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_HOST=0.0.0.0
      - TZ=America/Los_Angeles

volumes:
  n8n_data:
```

### If Using a Reverse Proxy

If n8n is behind Nginx, Traefik, or similar:

1. **Enable WebSocket support** in your proxy
2. Add this environment variable to n8n:
   ```
   N8N_PROXY_HOPS=1
   ```

This prevents "Lost connection to server" errors.

## Step 1: RapidAPI Setup

### 1.1 Create Account

1. Go to [rapidapi.com](https://rapidapi.com)
2. Click **Sign Up** (free)
3. Verify your email

### 1.2 Subscribe to JSearch

1. Go to [JSearch API](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch)
2. Click **Subscribe to Test**
3. Select the **Basic** plan (free, 500 requests/month)
4. Complete subscription

### 1.3 Get Your API Key

1. On the JSearch page, look at the code examples on the right
2. Find `X-RapidAPI-Key` in the headers
3. Copy this value (it looks like: `a1b2c3d4e5f6...`)
4. Save it somewhere secure

**Important:** Never share this key publicly or commit it to Git.

## Step 2: Discord Webhook

### 2.1 Create a Channel

1. Open Discord
2. In your server, create a new channel (e.g., `#job-radar`)
3. Or use an existing channel

### 2.2 Create Webhook

1. Right-click the channel → **Edit Channel**
2. Go to **Integrations** → **Webhooks**
3. Click **New Webhook**
4. Set the name: `Job Radar`
5. (Optional) Upload an icon
6. Click **Copy Webhook URL**
7. Save this URL

**Webhook URL format:**
```
https://discord.com/api/webhooks/123456789/abcdefghijklmnop...
```

## Step 3: n8n Configuration

### 3.1 Access n8n

Open your n8n instance:
- Cloud: `https://your-instance.app.n8n.cloud`
- Self-hosted: `http://localhost:5678` or your domain

### 3.2 Create RapidAPI Credential

1. Click **Credentials** in the left sidebar
2. Click **Add Credential**
3. Search for **Header Auth**
4. Configure:
   - **Name:** `RapidAPI Key`
   - **Header Name:** `X-RapidAPI-Key`
   - **Header Value:** (paste your API key from Step 1.3)
5. Click **Save**

## Step 4: Import and Configure

### 4.1 Import Workflow

1. Download `job-radar-workflow.json` from this repository
2. In n8n, click **Workflows** → **Import from File**
3. Select the JSON file
4. Click **Import**

### 4.2 Configure JSearch API Node

1. Click on the **JSearch API** node
2. Under **Credential to connect with**, select `RapidAPI Key`
3. Click somewhere else to save

### 4.3 Configure Discord Webhook

1. Click on the **Post to Discord** node
2. Find the **URL** field
3. Replace `YOUR_DISCORD_WEBHOOK_URL` with your actual webhook URL
4. Click somewhere else to save

### 4.4 Customize Search (Optional)

Click on the **Search Config** node to modify:

**Job Titles:**
```javascript
=["Software Engineer", "DevOps Engineer", "Cloud Engineer"]
```

**Location:**
```
San Francisco, CA
```

**Radius (miles):**
```
25
```

### 4.5 Save Workflow

Click **Save** in the top right.

## Step 5: Test and Activate

### 5.1 Run a Test

1. Click **Execute Workflow** (orange button at bottom)
2. Watch the workflow run through each node
3. Check your Discord channel for job posts

**Expected runtime:** 3-5 minutes (due to rate limiting delays)

### 5.2 Troubleshoot if Needed

If any node turns red:
1. Click on the red node
2. Read the error message
3. Check the [Troubleshooting](#troubleshooting) section

### 5.3 Activate

Once the test succeeds:
1. Toggle **Active** (top right)
2. The workflow will now run automatically every 6 hours

## Troubleshooting

### "Lost connection to the server"

**Cause:** WebSocket connection issues

**Fix:**
1. If using a reverse proxy, enable WebSocket support
2. Add `N8N_PROXY_HOPS=1` to n8n environment variables
3. Restart n8n

### "The service is receiving too many requests" (JSearch)

**Cause:** RapidAPI rate limit

**Fix:**
1. Wait a few minutes before testing again
2. Reduce the number of job titles
3. Increase the Wait node time (try 15 seconds)

### "The service is receiving too many requests" (Discord)

**Cause:** Discord webhook rate limit

**Fix:**
1. The batching settings should handle this
2. If it persists, increase `batchInterval` to 5000 (5 seconds)

### "The value in the JSON Body field is not valid JSON"

**Cause:** Special characters in job data

**Fix:** The workflow uses `JSON.stringify()` to escape special characters. If you see this error:
1. Check the Format Job Data node
2. Ensure values have fallbacks: `$json.company || "N/A"`

### No Jobs Found

**Possible causes:**
1. `date_posted: "today"` is too restrictive
2. Location format is wrong

**Fix:**
1. In JSearch API node, change `date_posted` to `week`
2. Use format: `City, State` (e.g., `Los Angeles, CA`)

### Workflow Takes Too Long

**Expected behavior:** With 15 job titles and rate limiting, the workflow takes 4-5 minutes.

**To speed up:**
1. Reduce job titles to 5-8
2. Reduce Wait node times (may cause rate limit errors)

## Next Steps

Once your workflow is running:

1. **Monitor for a few days** to see job quality
2. **Adjust job titles** based on what's most relevant
3. **Consider upgrading** RapidAPI if you need more requests
4. **Add persistence** to avoid duplicate posts across runs

See [CUSTOMIZATION.md](CUSTOMIZATION.md) for advanced modifications.

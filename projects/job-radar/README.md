# Job Radar: Automated Job Search Pipeline

An n8n workflow that automatically searches multiple job boards and delivers matching positions directly to your Discord server. Built as a beginner-friendly introduction to workflow automation and API integration.

![n8n](https://img.shields.io/badge/n8n-Workflow%20Automation-orange)
![Discord](https://img.shields.io/badge/Discord-Webhook%20Integration-5865F2)
![RapidAPI](https://img.shields.io/badge/RapidAPI-JSearch-blue)
![License](https://img.shields.io/badge/License-MIT-green)

## What It Does

Job Radar runs on a schedule (default: every 6 hours) and:

1. Searches 15 job titles across Indeed, LinkedIn, Glassdoor, and ZipRecruiter
2. Filters results by location and radius
3. Removes duplicate listings
4. Posts formatted job cards to a Discord channel

No more manually checking multiple job boards. Wake up to fresh opportunities in your Discord.

## Sample Output

Jobs appear in Discord as rich embeds:

```
┌─────────────────────────────────────────────┐
│ Senior DevOps Engineer                      │
├─────────────────────────────────────────────┤
│ 🏢 Company   │ 📍 Location │ 💰 Salary     │
│ Acme Corp    │ Los Angeles, CA│ $150-180K   │ 
├─────────────────────────────────────────────┤
│ Job Radar                                   │
└─────────────────────────────────────────────┘
```

## Prerequisites

Before you begin, you'll need:

- **n8n** (self-hosted or cloud): [Installation Guide](https://docs.n8n.io/hosting/)
- **RapidAPI Account**: Free tier available at [rapidapi.com](https://rapidapi.com)
- **Discord Server**: With permission to create webhooks

## Quick Start

### Step 1: Get Your API Key

1. Create a free account at [rapidapi.com](https://rapidapi.com)
2. Subscribe to [JSearch API](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch) (free tier: 500 requests/month)
3. Copy your `X-RapidAPI-Key` from the code snippets section

### Step 2: Create Discord Webhook

1. Open your Discord server
2. Go to the channel where you want job alerts
3. Click the gear icon → **Integrations** → **Webhooks**
4. Click **New Webhook**
5. Name it "Job Radar" and copy the webhook URL

### Step 3: Import the Workflow

1. Download `job-radar-workflow.json` from this repo
2. Open n8n
3. Go to **Workflows** → **Import from File**
4. Select the downloaded JSON file

### Step 4: Configure Credentials

**Add RapidAPI Credential:**
1. Go to **Credentials** → **Add Credential**
2. Search for "Header Auth"
3. Configure:
   - Name: `RapidAPI Key`
   - Header Name: `X-RapidAPI-Key`
   - Header Value: `your-api-key-here`

**Update Discord Webhook:**
1. Open the imported workflow
2. Click on the **Post to Discord** node
3. Replace the URL with your webhook URL

### Step 5: Customize Search Parameters

Click on the **Search Config** node to modify:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `jobTitles` | 15 titles | Array of job titles to search |
| `location` | Los Angeles, CA | Search location |
| `radius` | 30 | Search radius in miles |

### Step 6: Activate

1. Click **Save**
2. Toggle the workflow to **Active**
3. Wait for the next scheduled run, or click **Execute Workflow** to test

## Workflow Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Schedule   │────▶│ Split Titles │────▶│    Wait     │
│  (6 hours)   │     │  (15 items)  │     │ (10 seconds) │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
                                                 ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Filter &   │◀────│  Split Jobs  │◀────│  JSearch    │
│   Dedupe     │     │  (per title) │     │    API       │
└──────────────┘     └──────────────┘     └──────────────┘
        │
        ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Format     │────▶│    Wait      │────▶│   Discord   │
│   Job Data   │     │ (5 seconds)  │     │   Webhook    │
└──────────────┘     └──────────────┘     └──────────────┘
```

## Understanding the Workflow

This project introduces several key concepts for beginners:

### 1. Schedule Triggers
The workflow runs automatically on a schedule using n8n's Schedule Trigger node. You can adjust the frequency based on your needs.

### 2. API Integration
We use the JSearch API (via RapidAPI) to aggregate job listings from multiple sources. This demonstrates:
- HTTP requests with authentication
- Query parameters
- Response handling

### 3. Data Transformation
Raw API responses are processed through several steps:
- **Split nodes**: Break arrays into individual items
- **Filter nodes**: Remove invalid entries
- **Set nodes**: Reshape data into the format we need

### 4. Rate Limiting
Both JSearch and Discord have rate limits. We handle this with:
- Wait nodes between API calls
- Batch settings on the Discord node

### 5. Webhook Integration
Discord webhooks provide a simple way to post messages programmatically. The workflow constructs a JSON payload matching Discord's embed format.

## Customization

### Change Job Titles

Edit the `Search Config` node and modify the `jobTitles` array:

```javascript
=["Software Engineer", "Backend Developer", "Full Stack Engineer"]
```

### Add Remote-Only Filter

In the `JSearch API` node, change:
```
remote_jobs_only: "true"
```

### Change Search Frequency

Click on `Run Every 6 Hours` and adjust the interval. Options:
- Hours: 1-24
- Days: 1-7
- Cron expressions for advanced scheduling

### Filter by Salary

Add a Filter node after `Format Job Data`:
- Field: `salary`
- Operation: `does not equal`
- Value: `Not listed`

## Troubleshooting

### "Lost connection to the server"

If using n8n behind a reverse proxy (like Nginx Proxy Manager):

1. Enable **Websockets Support** in your proxy settings
2. Add this environment variable to n8n:
   ```
   N8N_PROXY_HOPS=1
   ```

### Rate Limit Errors (429)

**JSearch API:**
- Free tier allows 500 requests/month
- Each job title = 1 request
- 15 titles × 4 runs/day = 60 requests/day

**Discord:**
- Webhooks are limited to ~30 messages/minute
- The workflow includes batching (1 message per 3 seconds)

### No Jobs Found

- Check if `date_posted` is set to `today` (might be too restrictive)
- Try `week` for more results initially
- Verify your location format: "City, State" (e.g., "Los Angeles, CA")

## API Usage Notes

### JSearch Free Tier Limits
- 500 requests/month
- This workflow uses ~60 requests/day at default settings (15 titles × 4 runs)

### Staying Within Limits
- Reduce job titles to your top priorities
- Run twice daily instead of four times
- Use `date_posted: "week"` and run once daily

## Project Structure

```
job-radar/
├── README.md                    # This file
├── job-radar-workflow.json      # n8n workflow (import this)
├── docs/
│   ├── SETUP.md                 # Detailed setup guide
│   └── CUSTOMIZATION.md         # Advanced customization
└── examples/
    └── discord-embed-format.json # Discord message structure
```

## Contributing

Contributions are welcome! Some ideas:

- [ ] Add persistent deduplication (SQLite/Redis)
- [ ] Keyword scoring (rank by relevance)
- [ ] Email digest option
- [ ] Slack integration
- [ ] Application tracking

## License

MIT License. See [LICENSE](LICENSE) for details.

## Acknowledgments

- [n8n](https://n8n.io/) for the workflow automation platform
- [JSearch API](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch) for job data aggregation
- Built as part of my homelab automation journey

---

**Questions?** Open an issue or reach out on [LinkedIn](https://www.linkedin.com/in/timanlemvo/).

# Implementation

How Job Radar was built, step by step.

---

## Overview

Job Radar is an n8n workflow that:
1. Runs on a schedule (every 6 hours)
2. Searches 15 job titles via JSearch API
3. Filters and deduplicates results
4. Posts formatted job cards to Discord

Total nodes: 10
Average runtime: 4-5 minutes

---

## Workflow Nodes

### 1. Schedule Trigger

**Node:** `n8n-nodes-base.scheduleTrigger`

**Purpose:** Starts the workflow automatically.

**Configuration:**
- Interval: Every 6 hours

---

### 2. Search Config

**Node:** `n8n-nodes-base.set`

**Purpose:** Define search parameters in one place for easy customization.

**Output:**
```json
{
  "jobTitles": ["Cloud Engineer", "DevOps Engineer", ...],
  "location": "Los Angeles, CA",
  "radius": "30"
}
```

**Why:** Centralizes configuration. Change job titles or location in one node instead of editing the API call directly.

---

### 3. Split Job Titles

**Node:** `n8n-nodes-base.splitOut`

**Purpose:** Convert the jobTitles array into individual items for sequential processing.

**Input:** 1 item with array of 15 titles
**Output:** 15 items, one per title

**Why:** JSearch API searches one query at a time. Splitting allows us to search each title separately.

---

### 4. Wait (10 seconds)

**Node:** `n8n-nodes-base.wait`

**Purpose:** Rate limiting between API calls.

**Configuration:**
- Amount: 10
- Unit: Seconds

**Why:** Prevents 429 errors from JSearch API. Free tier has per-minute request limits.

---

### 5. JSearch API

**Node:** `n8n-nodes-base.httpRequest`

**Purpose:** Fetch job listings from aggregated job boards.

**Configuration:**
- Method: GET
- URL: `https://jsearch.p.rapidapi.com/search`
- Authentication: Header Auth (X-RapidAPI-Key)

**Query Parameters:**
| Parameter | Value |
|-----------|-------|
| query | `{{ $json.jobTitles }} in {{ $('Search Config').item.json.location }}` |
| page | 1 |
| num_pages | 1 |
| date_posted | today |
| radius | `{{ $('Search Config').item.json.radius }}` |
| remote_jobs_only | false |

**Headers:**
- X-RapidAPI-Host: jsearch.p.rapidapi.com

**Output:** JSON response with `data` array containing job listings.

---

### 6. Split Jobs

**Node:** `n8n-nodes-base.splitOut`

**Purpose:** Convert the `data` array from API response into individual job items.

**Configuration:**
- Field to Split Out: `data`

**Input:** 15 API responses (one per job title)
**Output:** ~100 individual job items

---

### 7. Filter Valid Jobs

**Node:** `n8n-nodes-base.filter`

**Purpose:** Remove empty or invalid job entries.

**Configuration:**
- Condition: `job_id` is not empty

**Why:** Some API responses include null entries. Filtering prevents errors in downstream nodes.

---

### 8. Remove Duplicates

**Node:** `n8n-nodes-base.itemLists`

**Purpose:** Deduplicate jobs that appear under multiple search titles.

**Configuration:**
- Operation: Remove Duplicates
- Compare: Selected Fields
- Fields to Compare: `job_id`

**Input:** ~100 jobs
**Output:** ~87 unique jobs

**Why:** A "Cloud Security Engineer" job might appear in both "Cloud Engineer" and "Security Engineer" searches.

---

### 9. Format Job Data

**Node:** `n8n-nodes-base.set`

**Purpose:** Extract and format only the fields needed for Discord.

**Field Mappings:**
| Output Field | Expression |
|--------------|------------|
| job_id | `{{ $json.job_id }}` |
| title | `{{ $json.job_title }}` |
| company | `{{ $json.employer_name }}` |
| location | `{{ $json.job_city }}, {{ $json.job_state }}` |
| remote | `{{ $json.job_is_remote ? 'Remote' : 'On-site' }}` |
| salary | `{{ $json.job_min_salary && $json.job_max_salary ? '$' + $json.job_min_salary.toLocaleString() + ' - $' + $json.job_max_salary.toLocaleString() : 'Not listed' }}` |
| url | `{{ $json.job_apply_link }}` |

**Why:** Raw API response has 50+ fields per job. Discord only needs 6-7.

---

### 10. Wait (5 seconds)

**Node:** `n8n-nodes-base.wait`

**Purpose:** Rate limiting before Discord posts.

**Configuration:**
- Amount: 5
- Unit: Seconds

**Why:** Additional buffer before hitting Discord webhook.

---

### 11. Post to Discord

**Node:** `n8n-nodes-base.httpRequest`

**Purpose:** Send formatted job cards to Discord channel.

**Configuration:**
- Method: POST
- URL: Discord webhook URL
- Body Type: JSON

**JSON Body:**
```javascript
={{ JSON.stringify({
  embeds: [{
    title: $json.title,
    url: $json.url,
    color: 36863,
    fields: [
      { name: "🏢 Company", value: $json.company || "N/A", inline: true },
      { name: "📍 Location", value: $json.location || "N/A", inline: true },
      { name: "💰 Salary", value: $json.salary || "Not listed", inline: true }
    ],
    footer: { text: "Job Radar" }
  }]
}) }}
```

**Batching Options:**
- Items per Batch: 1
- Batch Interval: 3000ms

**Why:** `JSON.stringify()` escapes special characters. Batching prevents Discord rate limits.

---

## Data Flow

```
Schedule (1 trigger)
    │
    ▼
Search Config (1 item)
    │
    ▼
Split Job Titles (15 items)
    │
    ▼
Wait 10s ──────────────┐
    │                  │
    ▼                  │ (loops for each title)
JSearch API ───────────┘
    │
    ▼
Split Jobs (~100 items)
    │
    ▼
Filter Valid (~100 items)
    │
    ▼
Remove Duplicates (~87 items)
    │
    ▼
Format Job Data (~87 items)
    │
    ▼
Wait 5s
    │
    ▼
Post to Discord (batched, 1 per 3s)
```

---

## Credentials Required

### RapidAPI Key (Header Auth)

- Type: Header Auth
- Name: `RapidAPI Key`
- Header Name: `X-RapidAPI-Key`
- Header Value: Your API key from RapidAPI

### Discord Webhook

- No credential needed
- Webhook URL pasted directly in node

---

## Environment

**Platform:** n8n (self-hosted)
**Version:** 2.13.2
**Container:** Docker
**Host:** Proxmox LXC (Phoenix-Nest)
**Reverse Proxy:** Nginx Proxy Manager with SSL

**Required n8n Settings:**
```yaml
environment:
  N8N_HOST: 0.0.0.0
  N8N_PROTOCOL: https
  WEBHOOK_URL: https://n8n.yourdomain.com/
  N8N_PROXY_HOPS: 1
  TZ: America/Los_Angeles
```

---

## Testing

### Test Individual Nodes

1. Click on a node
2. Click "Test step" (play button)
3. Inspect output in the panel

### Test Full Workflow

1. Click "Execute Workflow"
2. Watch nodes light up as they complete
3. Check Executions tab for logs

### Verify Discord Output

1. Open Discord channel
2. Confirm job cards appear
3. Click title links to verify URLs work

---

## Deployment

1. Save workflow
2. Toggle "Active" (top right)
3. Workflow runs automatically on schedule

### Verify Active

- Check Executions tab after next scheduled time
- Confirm new jobs appearing in Discord

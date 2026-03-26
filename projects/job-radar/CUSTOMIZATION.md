# Customization Guide

This guide covers advanced modifications to Job Radar.

## Changing Job Titles

### For Different Roles

Edit the **Search Config** node and replace the `jobTitles` array:

**Frontend Development:**
```javascript
=["Frontend Developer", "React Developer", "Vue.js Developer", "UI Engineer", "JavaScript Developer"]
```

**Data Engineering:**
```javascript
=["Data Engineer", "Data Analyst", "Analytics Engineer", "ETL Developer", "Business Intelligence"]
```

**Security:**
```javascript
=["Security Engineer", "SOC Analyst", "Penetration Tester", "Security Architect", "CISO"]
```

**Management:**
```javascript
=["Engineering Manager", "VP Engineering", "CTO", "Technical Director", "IT Director"]
```

## Location Options

### Different Cities

Change the `location` value in Search Config:

```
New York, NY
San Francisco, CA
Austin, TX
Seattle, WA
Chicago, IL
Denver, CO
Boston, MA
```

### Multiple Locations

To search multiple locations, duplicate the workflow or create a loop:

1. Add locations to an array in Search Config
2. Add another Split node after the location array
3. Modify the JSearch API query to use the split location

## Remote-Only Jobs

In the **JSearch API** node, change:

```
remote_jobs_only: "true"
```

## Date Range

In the **JSearch API** node, change `date_posted`:

| Value | Description |
|-------|-------------|
| `today` | Last 24 hours |
| `3days` | Last 3 days |
| `week` | Last 7 days |
| `month` | Last 30 days |
| `all` | All time |

For initial setup, use `week` to see more results. Then switch to `today` for daily monitoring.

## Salary Filtering

Add a Filter node after **Format Job Data**:

1. Add node: **Filter**
2. Configure:
   - Field: `salary`
   - Operation: `does not equal`
   - Value: `Not listed`

This only posts jobs with listed salaries.

## Keyword Filtering

To filter for specific technologies, add a Filter node:

**Example: Only Azure Jobs**

1. Add a **Filter** node after Split Jobs
2. Access the raw job description: `{{ $json.job_description }}`
3. Configure:
   - Operation: `contains`
   - Value: `Azure`

**Multiple Keywords (any match):**
```javascript
={{ $json.job_description.toLowerCase().includes('azure') || 
   $json.job_description.toLowerCase().includes('aws') ||
   $json.job_description.toLowerCase().includes('gcp') }}
```

## Change Schedule

Click **Run Every 6 Hours** and modify:

**More Frequent:**
- Every 3 hours
- Every hour (watch your API limits!)

**Less Frequent:**
- Every 12 hours
- Once daily at 8 AM

**Custom Cron:**
```
0 8 * * 1-5
```
(Runs at 8 AM, Monday through Friday)

## Discord Formatting

### Add Company Logo

Modify the JSON body in **Post to Discord**:

```javascript
={{ JSON.stringify({
  embeds: [{
    title: $json.title,
    url: $json.url,
    color: 36863,
    thumbnail: {
      url: $json.employer_logo || ""
    },
    fields: [
      { name: "🏢 Company", value: $json.company || "N/A", inline: true },
      { name: "📍 Location", value: $json.location || "N/A", inline: true },
      { name: "💰 Salary", value: $json.salary || "Not listed", inline: true }
    ],
    footer: { text: "Job Radar" }
  }]
}) }}
```

### Add Job Description

```javascript
fields: [
  { name: "🏢 Company", value: $json.company || "N/A", inline: true },
  { name: "📍 Location", value: $json.location || "N/A", inline: true },
  { name: "💰 Salary", value: $json.salary || "Not listed", inline: true },
  { name: "📝 Description", value: ($json.description_snippet || "No description").substring(0, 200) + "...", inline: false }
]
```

### Change Embed Color

The `color` value is a decimal number. Some options:

| Color | Decimal |
|-------|---------|
| Blue | 36863 |
| Green | 65280 |
| Red | 16711680 |
| Purple | 10181046 |
| Orange | 16744448 |
| Gold | 15844367 |

## Slack Integration

Replace the Discord webhook with Slack:

1. Create a Slack app and webhook
2. Change the **Post to Discord** node URL
3. Modify the JSON body:

```javascript
={{ JSON.stringify({
  blocks: [
    {
      type: "section",
      text: {
        type: "mrkdwn",
        text: "*<" + $json.url + "|" + $json.title + ">*\n" +
              "🏢 " + ($json.company || "N/A") + " | " +
              "📍 " + ($json.location || "N/A") + " | " +
              "💰 " + ($json.salary || "Not listed")
      }
    }
  ]
}) }}
```

## Email Digest

Instead of individual posts, send a daily summary:

1. Remove the batching/wait logic
2. Add an **Aggregate** node to combine all jobs
3. Add an **Email Send** node with a formatted list

## Persistent Deduplication

To avoid posting the same job across runs, you need persistent storage.

### Option 1: n8n Data Tables (Easiest)

1. Create a Data Table in n8n
2. Before posting, check if `job_id` exists
3. After posting, add `job_id` to the table

### Option 2: External Database

Add nodes to:
1. Query a database (PostgreSQL, MySQL, SQLite)
2. Check if job exists
3. Insert new jobs after posting

### Option 3: File-Based

1. Store job IDs in a JSON file
2. Read file at start of workflow
3. Filter out already-seen IDs
4. Write updated file at end

## Adding More Job Sources

JSearch aggregates multiple sources, but you can add more:

### LinkedIn API (Requires Partner Access)

Not publicly available, but you can scrape with caution.

### Indeed Publisher API

Requires approval, provides direct access.

### Company Career Pages

Use the HTTP Request node to fetch RSS feeds or JSON APIs from specific companies you're interested in.

# Problems Encountered

Issues hit during the Job Radar build and how they were resolved.

---

## 1. WebSocket Disconnection

**Error:** "Lost connection to the server"

**Cause:** n8n uses WebSockets for real-time updates. Running behind a reverse proxy (Nginx Proxy Manager) without WebSocket support caused connections to drop.

**Fix:**
1. Enable WebSocket support in NPM proxy host settings
2. Add environment variable to n8n container:
   ```yaml
   environment:
     N8N_PROXY_HOPS: 1
   ```

---

## 2. SQLite Node Not Available

**Error:** "Unrecognized node type: n8n-nodes-base.sqlite"

**Cause:** SQLite node is not included in default self-hosted n8n. Requires installing community nodes.

**Fix:** Removed SQLite dependency. Used built-in "Remove Duplicates" node for per-run deduplication instead.

**Note:** Persistent deduplication moved to Phase 2 roadmap.

---

## 3. JSearch API Rate Limiting

**Error:** 429 "The service is receiving too many requests from you"

**Cause:** Sending 15 API requests too quickly. Free tier has per-minute limits.

**Fix:** Added a 10-second Wait node between Split Job Titles and JSearch API.

```
Split Job Titles → Wait (10s) → JSearch API
```

---

## 4. Discord Webhook Rate Limiting

**Error:** 429 "You are being rate limited" (failed on item 4-5)

**Cause:** Discord webhooks limit to ~30 messages per minute. Posting 80+ jobs instantly triggers the limit.

**Fix:** Configured batching on the Post to Discord node:
- Items per Batch: 1
- Batch Interval: 3000ms (3 seconds)

---

## 5. Invalid JSON in Discord Payload

**Error:** "The value in the JSON Body field is not valid JSON"

**Cause:** Job titles and descriptions containing special characters (quotes, newlines, backslashes) broke the JSON structure.

**Fix:** Wrapped entire payload in `JSON.stringify()`:

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

---

## 6. Remove Duplicates Node Configuration

**Error:** "The 'Fields To Compare' parameter must be a string of fields separated by commas or an array of strings."

**Cause:** Used `[job_id]` format instead of plain string.

**Fix:** Changed fieldsToCompare value to just `job_id` (no brackets).

---

## 7. n8n Data Volume Disconnect

**Error:** n8n prompted for fresh setup after container restart. All workflows and credentials gone.

**Cause:** Modified docker-compose.yml and accidentally changed volume mount from named volume (`apps_n8n-data`) to bind mount (`/opt/n8n`), which was empty.

**Fix:** Reverted to named volume:
```yaml
volumes:
  - apps_n8n-data:/home/node/.n8n
```

**Lesson:** Always verify volume configuration before restarting containers. Named volumes persist data independently of the compose file.

---

## 8. Mixed YAML Syntax in Docker Compose

**Error:** Container failed to start after adding environment variable.

**Cause:** Mixed dict and list syntax in environment block:
```yaml
# Wrong
environment:
  N8N_HOST: 0.0.0.0
   - N8N_PROXY_HOPS=1  # Mixed syntax breaks YAML
```

**Fix:** Use consistent dict syntax:
```yaml
# Correct
environment:
  N8N_HOST: 0.0.0.0
  N8N_PROXY_HOPS: 1
```

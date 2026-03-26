# Tradeoffs

Design decisions made during the Job Radar build and the reasoning behind them.

---

## 1. Rate Limiting vs Speed

**Decision:** Added 10-second delays between API calls and 3-second delays between Discord posts.

**Reasoning:** A workflow that takes 5 minutes but completes reliably beats one that fails halfway through. Job searching is not time-sensitive to the minute.

**Impact:** Full workflow takes ~5 minutes for 15 job titles and 80+ results.

**Alternative:** Could reduce delays and accept occasional failures, then retry on next scheduled run.

---

## 2. Per-Run vs Persistent Deduplication

**Decision:** Used n8n's built-in Remove Duplicates node (memory only, resets each run).

**Reasoning:** Avoids external dependencies. Works immediately with no setup.

**Impact:** Same job may post again on subsequent runs if still listed on job boards.

**Alternative:** Add SQLite, Redis, or n8n Data Tables for persistent tracking. Planned for Phase 2.

---

## 3. 15 Job Titles vs API Quota

**Decision:** Search 15 different job titles per run.

**Reasoning:** Cast a wide net during active job search. Better to see too many relevant jobs than miss opportunities.

**Impact:** 
- 15 requests per run
- 4 runs per day = 60 requests/day
- ~1,800 requests/month
- Exceeds free tier (500/month)

**Options:**
- Reduce to 5-8 highest priority titles (stays within free tier)
- Run twice daily instead of four times
- Upgrade to RapidAPI Basic plan ($10/month)

---

## 4. Rich Embeds vs Simple Messages

**Decision:** Used Discord embeds with structured fields (company, location, salary).

**Reasoning:** 
- Cleaner visual presentation
- Clickable title links directly to application
- Easy to scan multiple jobs quickly
- Professional appearance

**Impact:** More complex JSON payload, more prone to escaping issues with special characters.

**Alternative:** Plain text messages (`"DevOps Engineer at Acme Corp - Los Angeles - $150K"`) would be simpler but harder to read at scale.

---

## 5. Self-Hosted vs Cloud n8n

**Decision:** Self-hosted on existing Proxmox homelab.

**Reasoning:**
- Free (no monthly cost)
- Full control over configuration
- Learning opportunity
- Fits existing infrastructure
- No workflow execution limits

**Impact:** Responsible for uptime, updates, backups, and troubleshooting.

**Alternative:** n8n Cloud ($20/month starter) handles infrastructure but has execution limits and less customization.

---

## 6. Single Workflow vs Modular Design

**Decision:** Built as one linear workflow.

**Reasoning:** Simpler to understand, debug, and share. Good for a v1 and for beginners learning n8n.

**Impact:** All logic in one place. Changes require editing the main workflow.

**Alternative:** Could split into sub-workflows (search, filter, notify) for better modularity. Consider for Phase 2 with BD-1 integration.

---

## 7. Discord vs Other Notification Methods

**Decision:** Post to Discord via webhook.

**Reasoning:**
- Already have Alliance Fleet Discord server
- Webhooks are simple (no bot hosting required)
- Rich embed support
- Mobile notifications built-in
- Can share channel with others

**Impact:** Requires Discord. Jobs only visible in that channel.

**Alternative:** 
- Email digest (daily summary instead of real-time)
- Slack webhook (same approach, different platform)
- Push notifications via Pushover/Ntfy
- Database + web dashboard

---

## 8. Today vs Week for Date Filter

**Decision:** Used `date_posted: "today"` for scheduled runs.

**Reasoning:** Reduces duplicate results across runs. Only see genuinely new postings.

**Impact:** Fewer results per run. May miss jobs posted late in the day that roll off by next run.

**Alternative:** Use `"week"` for initial backfill, then switch to `"today"` for ongoing monitoring.

---

## Summary

Most tradeoffs favor reliability and simplicity over speed and features. This is intentional for a v1 built quickly during a job search. Phase 2 can add persistence, modularity, and additional integrations once the core workflow is proven.

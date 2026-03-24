# BD-1 Job Application Tracker: Implementation Guide
*Phase 1: SQLite + Discord Commands (Ship Now)*
*Phase 2: n8n Application Tracker Workflow (Build Later)*

---

## Phase 1 Overview

Add a `job_applications` table to BD-1's SQLite database and three new commands:

| Command | Purpose |
|---------|---------|
| `!bd1 applied <company> <role> [url] [notes]` | Log a new application |
| `!bd1 apps [status]` | List applications, optionally filtered by status |
| `!bd1 appstatus <id> <status>` | Update an application's status |

Statuses: `applied`, `screening`, `interview`, `offer`, `rejected`, `withdrawn`, `ghosted`

Job commands (`applied`, `apps`, `appstatus`) work in both **#command-bridge** and **#job-search**.

---

## Step 1: Add Database Schema + Functions to db.js

SSH into Stinger Mantis, then:

```bash
nano /opt/bd1/db.js
```

**Edit 1: Add the table.** Find the `reminders` table creation block (the last `db.exec` in the `init` function). Add this right AFTER it, before `console.log('[DB] Initialized at', dbPath);`:

```javascript
  // Job application tracker
  db.exec(`
    CREATE TABLE IF NOT EXISTS job_applications (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      company TEXT NOT NULL,
      role TEXT NOT NULL,
      url TEXT,
      status TEXT NOT NULL DEFAULT 'applied',
      notes TEXT,
      applied_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  `);
```

**Edit 2: Add CRUD functions.** Find the `module.exports` block at the very bottom. Add these functions right BEFORE `module.exports`:

```javascript
// ── Job Applications ──

function addApplication(company, role, url = null, notes = null) {
  const stmt = db.prepare(`
    INSERT INTO job_applications (company, role, url, notes)
    VALUES (?, ?, ?, ?)
  `);
  return stmt.run(company, role, url, notes);
}

function getApplications(status = null) {
  if (status) {
    const stmt = db.prepare(`
      SELECT * FROM job_applications
      WHERE status = ?
      ORDER BY applied_at DESC
    `);
    return stmt.all(status);
  }
  const stmt = db.prepare(`
    SELECT * FROM job_applications
    ORDER BY applied_at DESC
  `);
  return stmt.all();
}

function getApplicationById(id) {
  const stmt = db.prepare(`SELECT * FROM job_applications WHERE id = ?`);
  return stmt.get(id);
}

function updateApplicationStatus(id, status) {
  const stmt = db.prepare(`
    UPDATE job_applications
    SET status = ?, updated_at = datetime('now')
    WHERE id = ?
  `);
  return stmt.run(status, id);
}

function getApplicationStats() {
  const stmt = db.prepare(`
    SELECT status, COUNT(*) as count
    FROM job_applications
    GROUP BY status
  `);
  return stmt.all();
}

function getPendingFollowUps(days = 5) {
  const stmt = db.prepare(`
    SELECT * FROM job_applications
    WHERE status = 'applied'
      AND applied_at <= datetime('now', '-' || ? || ' days')
    ORDER BY applied_at ASC
  `);
  return stmt.all(days);
}
```

**Edit 3: Export the new functions.** In the `module.exports` block, add these to the existing exports:

```javascript
  addApplication, getApplications, getApplicationById,
  updateApplicationStatus, getApplicationStats, getPendingFollowUps,
```

Save: `Ctrl+O` > Enter > `Ctrl+X`

---

## Step 2: Add Command Handlers to index.js

```bash
nano /opt/bd1/index.js
```

**Edit 1: Register commands.** Find the COMMANDS object. Add these three entries before the closing `};`:

```javascript
  applied: {
    description: 'Log a job application (e.g., !bd1 applied "Google" "Cloud Engineer" https://url "Referral from John")',
    handler: handleApplied,
  },
  apps: {
    description: 'List job applications (e.g., !bd1 apps, !bd1 apps interview)',
    handler: handleApps,
  },
  appstatus: {
    description: 'Update application status (e.g., !bd1 appstatus 3 interview)',
    handler: handleAppStatus,
  },
```

**Edit 2: Add handler functions.** Add these functions near the other handlers (after `handleReload` is fine):

```javascript
// ── Job Application Handlers ──

const APP_STATUS_EMOJI = {
  applied: '📨',
  screening: '📞',
  interview: '🎯',
  offer: '🎉',
  rejected: '❌',
  withdrawn: '🚪',
  ghosted: '👻',
};

const VALID_STATUSES = Object.keys(APP_STATUS_EMOJI);

function parseQuotedArgs(text) {
  const args = [];
  const regex = /"([^"]+)"|(\S+)/g;
  let match;
  while ((match = regex.exec(text)) !== null) {
    args.push(match[1] || match[2]);
  }
  return args;
}

async function handleApplied(message, args) {
  // Re-parse from raw content to handle quoted strings
  const rawContent = message.content;
  const afterCommand = rawContent.replace(/^!bd1\s+applied\s*/i, '');
  const parsed = parseQuotedArgs(afterCommand);

  if (parsed.length < 2) {
    await message.reply(
      '**Usage:** `!bd1 applied "Company" "Role" [url] ["notes"]`\n' +
      '**Example:** `!bd1 applied "Google" "Cloud Engineer" https://careers.google.com/123 "Referral from John"`\n' +
      'Company and Role are required. URL and notes are optional.'
    );
    return;
  }

  const company = parsed[0];
  const role = parsed[1];
  const url = parsed[2] && parsed[2].startsWith('http') ? parsed[2] : null;
  const notes = url ? (parsed[3] || null) : (parsed[2] || null);

  try {
    const result = db.addApplication(company, role, url, notes);
    const id = result.lastInsertRowid;

    let reply = `📨 **Application #${id} Logged**\n`;
    reply += `**Company:** ${company}\n`;
    reply += `**Role:** ${role}\n`;
    reply += `**Status:** Applied\n`;
    reply += `**Date:** ${new Date().toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}`;
    if (url) reply += `\n**URL:** ${url}`;
    if (notes) reply += `\n**Notes:** ${notes}`;

    await message.reply(reply);
  } catch (error) {
    console.error('[BD-1] Application logging error:', error.message);
    await message.reply(`Error logging application: ${error.message}`);
  }
}

async function handleApps(message, args) {
  const statusFilter = args[0] ? args[0].toLowerCase() : null;

  if (statusFilter && !VALID_STATUSES.includes(statusFilter)) {
    await message.reply(
      `Invalid status. Valid options: ${VALID_STATUSES.map(s => `\`${s}\``).join(', ')}`
    );
    return;
  }

  const apps = db.getApplications(statusFilter);

  if (apps.length === 0) {
    await message.reply(
      statusFilter
        ? `No applications with status "${statusFilter}".`
        : 'No applications logged yet. Use `!bd1 applied` to start tracking.'
    );
    return;
  }

  // Stats header (only when showing all)
  let reply = '';
  if (!statusFilter) {
    const stats = db.getApplicationStats();
    const statLine = stats.map(s => `${APP_STATUS_EMOJI[s.status] || '❓'} ${s.status}: ${s.count}`).join(' | ');
    reply += `**📊 Application Pipeline** (${apps.length} total)\n${statLine}\n\n`;
  } else {
    reply += `**${APP_STATUS_EMOJI[statusFilter] || '📋'} Applications: ${statusFilter}** (${apps.length})\n\n`;
  }

  // List applications (cap at 15 to stay within Discord limits)
  const display = apps.slice(0, 15);
  for (const app of display) {
    const date = new Date(app.applied_at).toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
    const emoji = APP_STATUS_EMOJI[app.status] || '❓';
    reply += `${emoji} **#${app.id}** ${app.company} — ${app.role} (${date})`;
    if (app.notes) reply += ` — _${app.notes}_`;
    reply += '\n';
  }

  if (apps.length > 15) {
    reply += `\n...and ${apps.length - 15} more. Filter by status to narrow: \`!bd1 apps interview\``;
  }

  // Follow-up nudge
  const overdue = db.getPendingFollowUps(5);
  if (overdue.length > 0 && !statusFilter) {
    reply += `\n⚠️ **${overdue.length} app(s) need follow-up** (5+ days, no status change)`;
  }

  await message.reply(reply);
}

async function handleAppStatus(message, args) {
  if (args.length < 2) {
    await message.reply(
      '**Usage:** `!bd1 appstatus <id> <status>`\n' +
      `**Statuses:** ${VALID_STATUSES.map(s => `\`${s}\``).join(', ')}\n` +
      '**Example:** `!bd1 appstatus 3 interview`'
    );
    return;
  }

  const id = parseInt(args[0]);
  const newStatus = args[1].toLowerCase();

  if (isNaN(id)) {
    await message.reply('Application ID must be a number. Use `!bd1 apps` to see IDs.');
    return;
  }

  if (!VALID_STATUSES.includes(newStatus)) {
    await message.reply(
      `Invalid status "${newStatus}". Valid: ${VALID_STATUSES.map(s => `\`${s}\``).join(', ')}`
    );
    return;
  }

  const app = db.getApplicationById(id);
  if (!app) {
    await message.reply(`No application found with ID #${id}. Use \`!bd1 apps\` to see all.`);
    return;
  }

  const oldStatus = app.status;
  db.updateApplicationStatus(id, newStatus);

  const emoji = APP_STATUS_EMOJI[newStatus];
  await message.reply(
    `${emoji} **Application #${id} Updated**\n` +
    `**${app.company}** — ${app.role}\n` +
    `${APP_STATUS_EMOJI[oldStatus]} ${oldStatus} → ${emoji} ${newStatus}`
  );
}
```

Save: `Ctrl+O` > Enter > `Ctrl+X`

---

## Step 2b: Add #job-search Channel Support to index.js

BD-1 currently only listens in #command-bridge. This edit adds #job-search as a second active channel for job commands, while keeping all commands available in #command-bridge.

**Edit 1: Add channel ID to .env.** First, get the #job-search channel ID from Discord (right-click the channel > Copy Channel ID). Then:

```bash
nano /opt/bd1/.env
```

Add this line:

```
JOB_SEARCH_CHANNEL_ID=your_channel_id_here
```

Save: `Ctrl+O` > Enter > `Ctrl+X`

**Edit 2: Update the message handler in index.js.** Find the `messageCreate` event handler where BD-1 checks which channel to respond in. There should be a line that checks if the message is in the command channel (something like `if (message.channel.id !== CONFIG.channels.command)`). Update the logic so BD-1 also responds in the job search channel.

```bash
nano /opt/bd1/index.js
```

Find where `CONFIG` loads channel IDs (near the top). Add the job search channel:

```javascript
  jobSearch: process.env.JOB_SEARCH_CHANNEL_ID || '',
```

Then find the channel gate in the `messageCreate` handler. It currently looks something like:

```javascript
  if (message.channel.id !== CONFIG.channels.command) return;
```

Replace it with:

```javascript
  const JOB_COMMANDS = ['applied', 'apps', 'appstatus'];
  const isCommandChannel = message.channel.id === CONFIG.channels.command;
  const isJobChannel = message.channel.id === CONFIG.channels.jobSearch;

  // Ignore messages outside active channels
  if (!isCommandChannel && !isJobChannel) return;

  // In #job-search, only respond to job commands and natural conversation
  if (isJobChannel && content.startsWith('!bd1 ')) {
    const cmd = content.split(/\s+/)[1]?.toLowerCase();
    if (!JOB_COMMANDS.includes(cmd)) {
      await message.reply('Job commands only in this channel: `!bd1 applied`, `!bd1 apps`, `!bd1 appstatus`');
      return;
    }
  }
```

This way #command-bridge keeps full access to every command, while #job-search only accepts the three job tracker commands. Natural conversation (messages without the `!bd1` prefix) still routes to Claude in both channels.

Save: `Ctrl+O` > Enter > `Ctrl+X`

---

## Step 3: Set Ownership and Restart

```bash
chown bd1:bd1 /opt/bd1/db.js /opt/bd1/index.js /opt/bd1/.env
su - bd1 -c "pm2 restart bd1"
su - bd1 -c "pm2 logs bd1 --lines 15"
```

Confirm `[BD-1] Online` with no errors. The new table auto-creates on startup.

---

## Step 4: Test in Discord

### In #command-bridge:

**Test 1: Help shows new commands**
```
!bd1 help
```
Verify `applied`, `apps`, `appstatus` appear in the list.

**Test 2: Log an application**
```
!bd1 applied "Riot Games" "Senior Systems Engineer" https://example.com/job123 "Found on LinkedIn"
```

**Test 3: Log another (no URL, no notes)**
```
!bd1 applied "CrowdStrike" "Cloud Security Engineer"
```

**Test 4: View all applications**
```
!bd1 apps
```

**Test 5: Update status**
```
!bd1 appstatus 1 screening
```

**Test 6: Filter by status**
```
!bd1 apps applied
```

**Test 7: Ghost an application**
```
!bd1 appstatus 2 ghosted
```

### In #job-search:

**Test 8: Job commands work here**
```
!bd1 apps
```

**Test 9: Non-job commands are blocked**
```
!bd1 status
```
Should reply with "Job commands only in this channel."

---

## Step 5: Update Knowledge Base

Add this section to the knowledge export in the `bd1-knowledge` GitHub repo (KNOWLEDGE_EXPORT.md). Place it after the existing commands documentation:

```markdown
## Job Application Tracker

BD-1 tracks job applications in SQLite with full pipeline visibility.

### Commands
- `!bd1 applied "Company" "Role" [url] ["notes"]` — Log a new application
- `!bd1 apps [status]` — List all applications or filter by status
- `!bd1 appstatus <id> <status>` — Update an application's pipeline status

### Statuses
applied → screening → interview → offer (or rejected/withdrawn/ghosted at any stage)

### Active Channels
Job commands work in both #command-bridge (full access) and #job-search (job commands only).

### Follow-Up Logic
Applications in "applied" status for 5+ days trigger a follow-up nudge in `!bd1 apps` output. This threshold is also available to the Monday briefing via `db.getPendingFollowUps(5)`.

### Integration Points (Phase 2, not yet built)
- n8n "Application Tracker" workflow with PostgreSQL backend
- Webhook trigger for automated logging
- #job-search Discord embeds with color-coded status
- Daily "Follow-Up Reminders" scheduled workflow
```

Then sync:

```bash
# On your local machine: commit and push to GitHub
cd bd1-knowledge
git add KNOWLEDGE_EXPORT.md
git commit -m "Add job application tracker with ghosted status, #job-search channel support (v3.3)"
git push

# On Stinger Mantis: reload
# Or in Discord: !bd1 reload
```

---

## Phase 2: n8n Application Tracker (Later)

When you're ready to build the full pipeline, the architecture from the Discord runbook applies:

1. **n8n Workflow: "Application Tracker"** — Webhook trigger receives new applications, logs to PostgreSQL, posts Discord embed to #job-search
2. **n8n Workflow: "Follow-Up Reminders"** — Daily scheduled query for stale "applied" entries, posts nudges to #job-search
3. **BD-1 migration** — Once PostgreSQL is live, BD-1's SQLite `job_applications` table becomes a local cache; n8n becomes the source of truth
4. **Color coding** — Green (new), Yellow (follow-up due), Purple (interview), Red (rejected)

The SQLite implementation you're shipping now is fully compatible with this migration path. The table schema matches the n8n design, so moving data over is a straight SQL export/import.

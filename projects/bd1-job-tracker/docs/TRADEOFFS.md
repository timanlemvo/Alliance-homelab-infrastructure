# Design Tradeoffs

## Decision 1: SQLite First, PostgreSQL Later

**Chosen:** Phase 1 ships with SQLite (BD-1's existing database). Phase 2 migrates to PostgreSQL via n8n.

**Alternatives considered:**

| Option | Pros | Cons |
|--------|------|------|
| SQLite only (forever) | Zero new dependencies, ships in minutes, queryable locally | No n8n integration, no automated webhooks, single-node |
| PostgreSQL from day one | Production-grade, n8n-native, supports concurrent access | Requires PostgreSQL setup, overkill for initial command set, delays shipping |
| Google Sheets API | Visual, shareable, familiar | External dependency, API auth overhead, latency, doesn't live in Discord |
| Notion API | Rich UI, kanban views | External dependency, API complexity, overkill |

**Rationale:** The job search is time-sensitive. SQLite ships in an hour because BD-1 already uses better-sqlite3. The schema is designed to match the planned PostgreSQL table structure, so Phase 2 migration is a straight SQL export/import with no schema translation. Shipping fast now, scaling later.

**Risk:** If Phase 2 is delayed indefinitely, SQLite remains the permanent store. This is acceptable because the data volume is small (hundreds of rows at most) and the query patterns are simple.

---

## Decision 2: Channel Scoping (Both #command-bridge and #job-search)

**Chosen:** Job commands (`applied`, `apps`, `appstatus`) work in both #command-bridge and #job-search. Non-job commands are blocked in #job-search.

**Alternatives considered:**

| Option | Pros | Cons |
|--------|------|------|
| Job commands in #command-bridge only | No code changes to channel handler | Job data buried in general ops channel, no separation of concerns |
| Job commands in #job-search only | Clean separation | Forces channel switch to log an application mid-conversation in #command-bridge |
| Full BD-1 access in #job-search | Maximum flexibility | Pollutes a focused channel with unrelated commands, blurs channel purpose |

**Rationale:** The dual-channel approach respects the Discord server's channel architecture (ops channels have defined purposes) while avoiding friction. If you're in #command-bridge talking to BD-1 about everything and want to log an application, you can. If you're in #job-search focused on the pipeline, you can. The gate on non-job commands in #job-search keeps that channel clean for its intended purpose.

---

## Decision 3: Quoted-String Argument Parsing

**Chosen:** Custom `parseQuotedArgs()` function that handles `"multi word arguments"` alongside single-word arguments.

**Alternatives considered:**

| Option | Pros | Cons |
|--------|------|------|
| Positional args (space-separated) | Simplest parsing | Breaks on "Riot Games" or "Cloud Security Engineer" |
| Slash commands (Discord interactions API) | Native Discord UX, modals, autocomplete | Requires bot application command registration, more complex codebase, slower iteration |
| JSON input | Unambiguous parsing | Terrible UX for typing in Discord on mobile |
| Delimiter-based (pipes or commas) | Simple split | Easy to forget the delimiter, fragile |

**Rationale:** Quoted strings are intuitive. `!bd1 applied "Riot Games" "Senior Systems Engineer"` reads naturally and handles multi-word company and role names without ambiguity. The regex is 3 lines. Slash commands are the better long-term UX (autocomplete, modals for notes), but they require Discord application command registration and a more complex handler pattern. That's a Phase 2 consideration.

---

## Decision 4: Seven Pipeline Statuses

**Chosen:** `applied`, `screening`, `interview`, `offer`, `rejected`, `withdrawn`, `ghosted`

**Alternatives considered:**

| Option | Pros | Cons |
|--------|------|------|
| Fewer statuses (applied/interview/closed) | Simpler | Loses visibility into screening vs. interview, no distinction between rejection and ghosting |
| More statuses (add "phone screen", "onsite", "negotiation", "accepted") | Granular tracking | Overhead of status management for a single-user system |
| Free-text status | Maximum flexibility | Impossible to query or aggregate, no emoji mapping |

**Rationale:** Seven statuses cover the real-world job search pipeline without overcomplicating it. The key distinction is `rejected` vs. `ghosted`: rejection is a definitive outcome, while ghosted (👻) signals a company that never responded. This matters for follow-up strategy. `withdrawn` captures the applicant's side of the equation. Adding more granular interview stages (phone screen, onsite, panel) can be handled in the `notes` field rather than multiplying statuses.

---

## Decision 5: Follow-Up Threshold at 5 Days

**Chosen:** Applications in `applied` status for 5+ days trigger a follow-up nudge.

**Rationale:** Industry convention for follow-up timing is typically 5 to 7 business days. Five calendar days is aggressive enough to stay top-of-mind without being premature. The threshold is a constant in the code (`getPendingFollowUps(5)`) and easy to adjust. Phase 2's n8n scheduled workflow will use the same threshold for automated daily nudges posted to #job-search.

---

## Decision 6: No Dedicated Database for Job Data

**Chosen:** Job applications live in BD-1's existing SQLite database (`/opt/bd1/data/bd1.sqlite`) alongside messages, projects, study sessions, and reminders.

**Alternatives considered:**

| Option | Pros | Cons |
|--------|------|------|
| Separate SQLite file | Isolation, portable backup | Extra file management, no benefit at this scale |
| Shared BD-1 database | Single backup target, single connection, consistent with existing pattern | All data in one file |

**Rationale:** BD-1's database is already the single source of truth for all bot-managed data. Adding a table to it is consistent with the existing architecture. The data is backed up when the VM is backed up. Phase 2 moves job data to PostgreSQL anyway, at which point the SQLite table becomes either a local cache or gets dropped.

---

## What Phase 2 Changes

| Concern | Phase 1 | Phase 2 |
|---------|---------|---------|
| Data store | SQLite (local) | PostgreSQL (n8n-managed) |
| Logging trigger | `!bd1 applied` command | n8n webhook (form, command, or API POST) |
| Follow-up nudges | Manual via `!bd1 apps` | Automated daily workflow posting to #job-search |
| Discord output | Bot reply in channel | Color-coded embeds via Admiral Ackbar webhook |
| Command interface | Text commands with quoted args | Slash commands with modals (optional) |

The Phase 1 SQLite schema, status set, and command UX all carry forward. Phase 2 adds automation on top; it does not replace the interactive commands.

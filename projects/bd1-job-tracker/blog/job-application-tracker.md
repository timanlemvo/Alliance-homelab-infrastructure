# Building a Job Application Tracker Inside a Discord Bot

*How I added pipeline visibility to BD-1 during an active job search.*

---

## The Problem

I got laid off. The job search started immediately, and within the first week I realized my personal ops bot (BD-1) had a gap: it tracked projects, study sessions, and infrastructure health, but knew nothing about the most important workstream happening in my life.

Every time I told BD-1 "I applied to CrowdStrike for a Cloud Security Engineer role," that context died when the conversation ended. The next session, BD-1 had no memory of it. My applications were scattered across LinkedIn bookmarks, email threads, and mental notes. No pipeline visibility, no follow-up tracking, no integration with my Monday morning briefings.

## The Approach

I broke the solution into two phases:

**Phase 1** ships a SQLite-backed command set directly into BD-1's existing codebase. Three commands: `!bd1 applied` to log applications, `!bd1 apps` to view the pipeline, and `!bd1 appstatus` to update status. This ships in about an hour because BD-1 already uses better-sqlite3.

**Phase 2** adds n8n automation on top: a PostgreSQL backend, webhook-triggered logging, color-coded Discord embeds in #job-search, and a daily scheduled workflow that posts follow-up reminders for applications that have gone silent.

The key design constraint: Phase 1's schema, status set, and command UX must carry forward into Phase 2 without a rewrite. SQLite becomes either a local cache or gets dropped when PostgreSQL takes over as the source of truth.

## Design Decisions Worth Noting

**Seven statuses, not three.** `applied`, `screening`, `interview`, `offer`, `rejected`, `withdrawn`, and `ghosted`. The distinction between `rejected` (they said no) and `ghosted` (they said nothing) matters for follow-up strategy. The 👻 emoji makes pipeline reviews a little less painful.

**Dual-channel scoping.** Job commands work in both #command-bridge (where BD-1 lives full-time) and #job-search (the dedicated career channel). Non-job commands are blocked in #job-search to keep it focused. This avoids the friction of switching channels to log an application mid-conversation.

**Quoted-string parsing over slash commands.** `!bd1 applied "Riot Games" "Senior Systems Engineer"` is less polished than a Discord slash command with autocomplete, but it ships today with a 3-line regex. Slash commands are a Phase 2 refinement.

**5-day follow-up threshold.** Applications stuck in `applied` status for 5+ days trigger a nudge in the `!bd1 apps` output. The same threshold feeds into Monday briefings. Aggressive enough to stay responsive, conservative enough to avoid premature follow-ups.

## What I Learned

Building tools for yourself during a stressful time is a double-edged sword. On one hand, the job tracker solved a real problem and reduced the cognitive load of managing 20+ active applications. On the other, there is a gravitational pull toward building infrastructure instead of using it. The two-phase approach helped with this: Phase 1 is deliberately minimal. Ship it, use it, then decide if Phase 2 is worth the investment based on actual usage patterns.

The other takeaway: your homelab projects are your portfolio. This tracker is a clean example of database schema design, command parsing, channel-scoped access control, and a migration-friendly architecture. Every one of those maps to a real interview question for the Cloud/DevOps/IAM roles I am targeting.

## Links

- [GitHub: bd1-job-tracker](https://github.com/timanlemvo/bd1-job-tracker)
- [BD-1 Knowledge Base](https://github.com/timanlemvo/bd1-knowledge)
- [Alliance Fleet Overview](https://tima.dev)

---

*This is post 034 in the Holocron Labs series documenting the Alliance Fleet homelab.*

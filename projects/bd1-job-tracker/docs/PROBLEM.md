# Problem Statement

## Context

BD-1 is a Claude-powered Discord bot that serves as a personal operations assistant for the Alliance Fleet homelab. It tracks projects, nudges on stalled work, logs study sessions, and delivers Monday morning briefings. It runs on the Stinger Mantis VM (Node-B, VLAN 20) under a dedicated `bd1` service user with PM2 process management.

BD-1's knowledge comes from a GitHub-synced knowledge base (`bd1-knowledge` repo) that contains a comprehensive export of infrastructure state, project status, and operational context. This file is loaded into Claude's system prompt on every conversation, giving BD-1 persistent awareness of the Alliance Fleet.

## The Problem

During an active job search, tracking applications becomes a critical operational concern. The typical workflow looks like this:

1. Find a role on LinkedIn, a company careers page, or through a referral
2. Tailor the resume and apply
3. Wait for a response
4. Follow up if no response after several days
5. Track status changes (screening, interview, offer, rejection)
6. Repeat across dozens of applications simultaneously

Without structured tracking, several failure modes emerge:

**Lost context.** BD-1 uses Claude's conversational memory, which resets every session. Telling BD-1 "I applied to Google for a Cloud Engineer role" in one conversation provides zero recall in the next. The information dies with the session.

**No pipeline visibility.** There is no way to answer basic questions like "how many applications are in interview stage?" or "which companies have I not heard back from in over a week?" without maintaining a separate spreadsheet or document.

**Missed follow-ups.** Applications that go silent for 5+ days need follow-up. Without automated tracking, these fall through the cracks, especially when juggling multiple active applications.

**Scattered data.** Job search activity lives across LinkedIn, email, bookmarks, and mental notes. There is no single source of truth.

**No integration with existing ops.** BD-1 already delivers Monday morning briefings with project status and study progress. The job search, arguably the highest-priority workstream, has no representation in these briefings.

## Requirements

The solution needs to:

1. **Persist application data** across BD-1 sessions in a queryable format
2. **Track full pipeline status** from applied through offer/rejection/ghosted
3. **Store metadata** per application: company, role, date applied, URL, notes (recruiter name, referral source, etc.)
4. **Surface follow-up nudges** for applications that have gone silent
5. **Integrate with Monday briefings** so the job search appears alongside project and study status
6. **Live where the work happens** (Discord, specifically #job-search) rather than requiring a context switch to another tool
7. **Scale later** to n8n-automated workflows with PostgreSQL, color-coded embeds, and daily follow-up reminders without requiring a rewrite

## Who This Is For

This is a single-user system built for a job seeker who already runs their operational life through Discord and BD-1. It is not a general-purpose ATS. The design prioritizes speed of implementation, integration with existing infrastructure, and a clean migration path to the n8n automation layer.

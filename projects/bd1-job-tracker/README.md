# BD-1 Job Tracker

**Automated job application pipeline for Discord, powered by SQLite and n8n.**

Part of the [Alliance Fleet](https://tima.dev) homelab ecosystem. BD-1 is a Claude-powered Discord bot running on the Stinger Mantis VM (Node-B, VLAN 20) that serves as a personal operations assistant. This module adds structured job application tracking with pipeline visibility, follow-up automation, and channel-scoped command access.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Discord Server                        │
│                                                          │
│   #command-bridge          #job-search                   │
│   (all BD-1 commands)      (job commands only)           │
│         │                        │                       │
│         └────────┬───────────────┘                       │
│                  ▼                                        │
│           ┌─────────────┐                                │
│           │    BD-1     │  Phase 1: SQLite                │
│           │  (Node.js)  │  Phase 2: n8n + PostgreSQL      │
│           └──────┬──────┘                                │
│                  │                                        │
│         ┌────────┴────────┐                              │
│         ▼                 ▼                               │
│   ┌──────────┐    ┌──────────────┐                       │
│   │  SQLite  │    │ n8n Webhooks │ (Phase 2)             │
│   │ (local)  │    │  PostgreSQL  │                       │
│   └──────────┘    └──────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

## Commands

| Command | Description | Example |
|---------|-------------|---------|
| `!bd1 applied` | Log a new application | `!bd1 applied "Riot Games" "Cloud Engineer" https://url "Referral"` |
| `!bd1 apps` | List applications (optional status filter) | `!bd1 apps interview` |
| `!bd1 appstatus` | Update pipeline status | `!bd1 appstatus 3 screening` |

## Pipeline Statuses

| Status | Emoji | Description |
|--------|-------|-------------|
| `applied` | 📨 | Application submitted |
| `screening` | 📞 | Recruiter screen scheduled or completed |
| `interview` | 🎯 | Technical or panel interview stage |
| `offer` | 🎉 | Offer received |
| `rejected` | ❌ | Application rejected |
| `withdrawn` | 🚪 | Withdrawn by applicant |
| `ghosted` | 👻 | No response after 14+ days |

## Phases

| Phase | Scope | Time Estimate | Status |
|-------|-------|---------------|--------|
| Phase 1 | SQLite + Discord commands | ~1.5 hours | 🔲 Ready to ship |
| Phase 2 | n8n workflows + PostgreSQL + automated follow-ups | ~3-4 hours | 🔲 Planned |

## Documentation

| Document | Purpose |
|----------|---------|
| [PROBLEM.md](docs/PROBLEM.md) | Why this exists, what it solves |
| [TRADEOFFS.md](docs/TRADEOFFS.md) | Design decisions and alternatives considered |
| [IMPLEMENTATION.md](docs/IMPLEMENTATION.md) | Step-by-step build guide with all code |

## Tech Stack

- **Runtime:** Node.js 20 + discord.js + better-sqlite3
- **Host:** Stinger Mantis VM (192.168.20.80, Node-B, VLAN 20)
- **Process Manager:** PM2 (runs under `bd1` service user)
- **Phase 2:** n8n (Phoenix-Nest, Node-B), PostgreSQL, Discord webhooks

## Related Projects

- [Alliance Fleet Infrastructure](https://tima.dev) (parent project)
- [BD-1 Knowledge Base](https://github.com/timanlemvo/bd1-knowledge) (persistent memory system)
- [Holocron Labs Blog](https://holocron-labs.tima.dev) (technical writeups)

---

*Built by [Tima Nlemvo](https://tima.dev) as part of the Alliance Fleet homelab.*

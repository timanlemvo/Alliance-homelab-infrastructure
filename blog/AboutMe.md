# Welcome to the Holocron

**Post 01: Transmission Origin**
**Title:** Welcome to the Holocron: Who I Am and Why This Exists
**Date:** February 2026
**Category:** Introduction / Career / Infrastructure Philosophy

---

## The Short Version

I'm Tima Nlemvo an IT Engineer with 7+ years in IT operations, now leveling up through infrastructure I build, break, and document myself.

This blog is the **Alliance Holocron** a technical archive of everything I'm learning, building, and solving across a production-grade home-lab environment. Writeups, incident forensics, project deep-dives, and the occasional hard-earned lesson.

---

## Where I've Been

My career started on service desks and escalated from there, literally.

**Creative Artists Agency (CAA)**  I managed asset lifecycle and procurement for, maintained Active Directory infrastructure, and led service desk operations while optimizing identity workflows and escalation paths.

**Stagwell**  I sustained high reliability across enterprise IT systems and cloud applications. Managed macOS and Windows fleets using JAMF Pro and Intune. Worked closely with security teams to enforce policy compliance across a distributed agency network.

**Team Liquid**  Currently operating and supporting production systems for competitive gaming and corporate environments. Tier III escalation across identity, endpoints, and core infrastructure. Access governance for 200+ users.

Every role taught me the same thing from a different angle: **what production resilience looks like when it's present — and what breaks when it's missing.**

---

## Why the Home-lab

Operational experience tells you *what* matters. It doesn't always let you build it yourself.

The Alliance Fleet is how I'm closing that gap. It's a 3-node Proxmox cluster running 25+ services — SIEM, identity management, observability pipelines, network segmentation, GPU-accelerated AI workloads — designed to mirror the standards I followed (and sometimes lacked) in enterprise environments.

This isn't a tutorial follow-along. Every component exists because I wanted to understand *why* it works, not just *how* to configure it.

**The Fleet at a Glance:**

| Node | Codename | Role |
|------|----------|------|
| **Node A** | Millennium Falcon | AI/ML Compute — RTX 4000 Ada, Ollama, VFIO passthrough |
| **Node B** | CR90 Corvette | Data & Operations — PostgreSQL, InfluxDB, Authentik, Grafana |
| **Node C** | Gozanti Cruiser | Network & Security — Wazuh SIEM, AdGuard, Nginx Proxy Manager |

Full architecture documentation lives in the [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) repo.

---

## What You'll Find Here

The Holocron publishes three types of content, each with a different purpose and voice:

**Writeups** — Incident forensics and postmortems. These document real problems I encountered, investigated, and resolved on live infrastructure. Full methodology, CLI evidence, root cause analysis, and lessons learned. Written to the standard expected in production postmortems.

**Projects** — Architecture and implementation deep-dives. How I designed and built specific systems SIEM pipelines, identity platforms, AI stacks with the tradeoffs and decisions explained.

**Blog** — The broader picture. Career reflections, infrastructure philosophy, and the process of leveling up from IT operations into systems engineering.

---

## The Operating Principles

These aren't abstract ideals. They're lessons from environments where outages had immediate consequences, applied daily in the Fleet:

- Design fault domains before deploying workloads
- Treat identity and network boundaries as foundational controls
- Favor explicit trust and default-deny over convenience
- Make observability and documentation first-class components
- Automate to reduce cognitive load, not just manual effort

---

## Connect

- **Portfolio:** [tima.dev](https://tima.dev)
- **GitHub:** [github.com/timanlemvo](https://github.com/timanlemvo)
- **LinkedIn:** [linkedin.com/in/timanlemvo](https://linkedin.com/in/timanlemvo)
- **Twitter/X:** [@tee_ma3](https://twitter.com/tee_ma3)
- **Email:** timanlemvo@gmail.com

---

*The Holocron is open. Transmissions inbound.*
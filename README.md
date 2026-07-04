# Threat Intelligence: MISP Deployment

## Overview

This document summarizes a MISP (Malware Information Sharing Platform) deployment used as the threat intelligence hub in a home lab security stack. It is written for a portfolio audience, focusing on architecture, feed strategy, and integration with downstream log enrichment.

MISP aggregates indicators of compromise (IOCs) from open-source threat feeds, normalizes them into a structured taxonomy, and exports actionable data (IPs, domains, hashes) to the log routing layer for real-time enrichment of network events.

## Environment

| Field | Value |
|-------|-------|
| System | VMCTI01 |
| IP Address | (internal) |
| OS | Ubuntu Server 24.04 (minimal) |
| Platform | MISP (self-hosted) |
| Web UI | https://(internal)/ |
| Key Ports | 443 (HTTPS), 22 (SSH) |
| Feeds | OSINT threat intelligence feeds |
| Export Target | Cribl Stream (IOC lookup tables) |

## Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│  External OSINT Feeds                                        │
│  • abuse.ch (URLhaus, MalwareBazaar, ThreatFox)              │
│  • CIRCL OSINT Feed                                          │
│  • Botvrij                                                   │
│  • Custom feed sources                                       │
└─────────────────────────┬───────────────────────────────────┘
                          │ Scheduled pull (hourly/daily)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  VMCTI01 — MISP                                              │
│                                                              │
│  ┌───────────┐  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Feed Mgr  │──│ Event Store  │──│ Export / API         │  │
│  │ (pull)    │  │ (MySQL)      │  │ (restSearch, CSV)    │  │
│  └───────────┘  └──────────────┘  └─────────────────────┘  │
│                                              │               │
│  Taxonomies, warninglists, correlation       │               │
└──────────────────────────────────────────────┼───────────────┘
                                               │ IOC export
                                               ▼
┌─────────────────────────────────────────────────────────────┐
│  VMCRIB01 — Cribl Stream                                     │
│  Lookup tables: malicious IPs, domains                       │
│  Enriches network events in-flight                           │
└─────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────┐
│  VMGRAY01 — Graylog                                          │
│  IOC-enriched events searchable and alertable                │
└─────────────────────────────────────────────────────────────┘
```

## Design Decisions

- **Self-hosted MISP over commercial TIP:** MISP is open-source, well-documented, and widely used in the security community. For a home lab, it provides enterprise-grade threat intelligence workflows without licensing costs.
- **OSINT feed focus:** The lab consumes freely available community feeds. This demonstrates feed curation, scheduling, and lifecycle management without requiring paid subscriptions.
- **Export to Cribl as lookup tables:** Rather than building complex Graylog correlation rules, IOCs are exported as flat files that Cribl loads for in-flight enrichment. This keeps enrichment fast and decoupled from the SIEM's processing pipeline.
- **Warninglists for false positive reduction:** MISP warninglists filter out known-good infrastructure (CDNs, cloud providers, DNS resolvers) from IOC exports, reducing alert noise in the SIEM.

## Quirks and Gotchas

- MISP's web installer requires careful attention to PHP version compatibility and Apache configuration.
- Feed scheduling should be staggered to avoid overwhelming the instance with concurrent pulls.
- Export automation (cron-based `restSearch` calls) needs API key management and error handling.
- Warninglists must be enabled explicitly — they are not active by default after installation.

## What This Demonstrates

This deployment shows practical threat intelligence operations: feed curation, IOC lifecycle management, structured export for downstream enrichment, and integration into a security monitoring pipeline. The value is in the workflow — from raw feed data through normalization, correlation, and actionable enrichment — not just in having a MISP instance running.

---

Sanitized for public portfolio use.

For step-by-step deployment instructions, see [OPERATIONS.md](./OPERATIONS.md).
For initial configuration (feeds, orgs, schedules), see the [MISP-Initial-Configuration](https://github.com/jared-labs/MISP-Initial-Configuration) repo.
For IOC export to Cribl, see the [MISP-to-Cribl-IOC-Export-Enrichment](https://github.com/jared-labs/MISP-to-Cribl-IOC-Export-Enrichment-Runbook) repo.

---
title: Reading-list index — pointer entries to canonical sources
kind: meta
fetched_at: 2026-05-18
tags: [index, meta]
---

# `reference/index/` — what lives here

This directory holds **curated reading-list pointers**, not full-text
imports. Each file is a topic-grouped list of links to canonical external
sources whose license forbids redistribution (Google SRE Book: CC-BY-NC-ND)
or whose content is hosted authoritatively elsewhere and we'd rather
operators read in source.

Files in this directory:

- [`sre-methodology.md`](sre-methodology.md) — Google SRE Book (41 chapters) + SRE Workbook (24 chapters)
- [`linux-kernel-internals.md`](linux-kernel-internals.md) — Gustavo Duarte's memory/syscall/boot series + packagecloud + Ardan Labs
- [`networking.md`](networking.md) — Cloudflare engineering blog (TCP, BPF, XDP) + packagecloud networking stack tuning
- [`observability-and-tracing.md`](observability-and-tracing.md) — Cilium troubleshooting (28 sub-sections) + performance debugging
- [`databases-and-storage.md`](databases-and-storage.md) — antirez Redis blog + PostgreSQL official docs + Kubernetes ops

# How the AIOps assistant uses these

The RAG indexer treats index files as `kind: meta` — they're retrieved
when the operator's question is about *learning a domain*, not about
fixing a specific incident. The model is expected to suggest a link
from these files when the in-house `concepts/`, `diagnostics/`, or
`systems/` material is thinner than the question warrants.

Full-text imports (under `reference/external/<topic>/`) are the
primary semantic-search target; this index is the secondary "go read"
recommendation.

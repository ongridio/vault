# ongridio vault

The platform-vendor knowledge base shipped with Ongrid. The manager
auto-registers this tree at boot under the `ongridio` org so its contents
are available to RAG retrieval, agent skills (`query_knowledge`), and the
Knowledge UI out of the box.

This repo will live at **`github.com/ongridio/vault`** (private). The
ongrid manager clones it on first run and re-syncs periodically. Operators
can override the source URL via the standard `knowledge_repos` mechanism.

## Layout

Top-level directories are organized by **the intent that brings a reader
(human or agent) to a doc**, not by the doc's literary form. RAG queries
typically carry intent (what is X / X is broken / X alert fired), so the
folder name aligns with the query and improves recall.

| Dir | Intent | Typical title shape |
|-----|--------|--------------------|
| `concepts/`   | "What is X / how do these signals relate" | Observability primer, alerting model, incident SOP |
| `systems/`    | "How does this system normally work" | Linux memory model, TCP state machine, Prometheus TSDB |
| `diagnostics/`| "X is broken — walk me through it" | Network connectivity, disk pressure, slow DB |
| `alerts/`     | "Alert ABC fired — what now" | Linked from rule `runbook_url`; per-alert handler |
| `reference/`  | "Give me the command / threshold quickly" | Linux cheatsheet, PromQL snippets, default thresholds |

## Language policy

**Single source of truth in English.** The frontend / agent layer
translates LLM output to the user's locale at presentation time (the
embedding model is multilingual; mixing zh+en in the same corpus hurts
ranking, and maintaining two mirrors guarantees drift).

If a doc genuinely must ship in Chinese (compliance excerpts, vendor
contracts, regulator text), name it `<slug>.zh.md` as a sibling — never
maintain parallel translations of the same doc.

## File conventions

- Markdown with GitHub-flavored extensions
- Front matter (YAML) optional; if present, supported keys:
  ```yaml
  ---
  title: <human title>
  tags: [linux, memory]
  alert_keys: [mem_high, swap_high]   # alerts/ only — drives rule linking
  applies_to: [edge, manager]         # optional scope hint
  ---
  ```
- Code blocks should include a language tag so the UI can syntax-highlight
- Inline commands use single backticks; full procedures use fenced blocks

## How the manager picks this up

On startup the manager:

1. Ensures the `ongridio` org exists
2. Ensures a `knowledge_repos` row points at this vault, owner_org = `ongridio`, kind = `builtin`
3. Pulls + walks + embeds every `.md` under the tracked directories
4. Tags chunks with their `path` (e.g. `diagnostics/network-connectivity`) so
   the Knowledge UI breadcrumb works and skills can scope retrieval

Operators can extend this vault by forking and pointing
`KNOWLEDGE_BUILTIN_VAULT_URL` at their fork. Built-in chunks remain
read-only in the UI (locked icon) to prevent local edits from being
clobbered by the next sync.

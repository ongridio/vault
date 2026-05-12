---
title: Incident Response SOP
tags: [incident, sop, postmortem]
---

# Incident Response SOP

The lifecycle Ongrid expects from alert-fired to postmortem-published.
The agent follows this shape and the UI's Incident Workspace mirrors it.

## Phases

### 1. Detect

An alert fires. The incident is created automatically when a `critical`
fires, or when a `warning` re-fires twice within 15 minutes. The incident
inherits the alert's labels (edge, service, rule_key) as its initial
scope.

### 2. Triage (target: < 5 min)

Determine: is this real, what's the blast radius, who needs to be paged?

The agent's `correlate_incident` skill auto-runs at this point: pulls the
metric, the matching traces, and the surrounding logs into the workspace.
Read those before you go investigating from scratch.

### 3. Mitigate (target: stop the bleed first)

Restore service. **Mitigation is not the same as root cause.** A
rollback, traffic shift, or rate-limit is a valid mitigation; you'll
diagnose the cause after the page closes.

Capture mitigation actions in the incident timeline as you take them — the
postmortem needs the actual sequence, not your reconstruction of it.

### 4. Diagnose

Now that the page is closed, find root cause. Use the correlated signals,
the timeline, and any diagnostic playbooks in `diagnostics/`. The
`investigator` agent persona is specifically tuned for this phase.

### 5. Postmortem

Blameless. Required for any `critical` incident that exceeded 30 minutes
of customer impact. Template lives in the incident workspace.

The three questions a postmortem must answer:

1. **What happened** — the actual sequence, including detection delay
2. **Why it happened** — root cause + contributing factors, separately
3. **What changes prevent recurrence** — concrete, owned, dated action
   items

## Roles during an active incident

- **Incident commander (IC)** — owns the response, makes calls, talks to
  stakeholders. Not necessarily the most senior person; whoever the IC
  designates is in charge until handoff.
- **Investigator** — drives diagnosis. Reports to IC.
- **Communicator** — owns the stakeholder updates and customer comms.
  Frees the IC from typing-while-thinking.
- **Scribe** — keeps the timeline current. Often automated by the agent.

For small incidents, IC and investigator are the same person. Don't
inflate the role count past what's useful.

## Severity → response

| Severity | Acknowledge | Mitigation owner | Communication cadence |
|----------|-------------|------------------|----------------------|
| SEV1 | < 5 min  | On-call + IC named | Every 15 min until mitigated |
| SEV2 | < 15 min | On-call            | Hourly |
| SEV3 | Next business day | Service owner | On resolution |

## Anti-patterns

- **Skipping the timeline** — without it, the postmortem becomes
  collective memory, which is wrong by definition
- **Diagnosing before mitigating** — never block mitigation on root cause
- **Blame the human** — if a human action caused the incident, the
  postmortem question is "what about the system made that action
  possible / easy / unrecoverable?"

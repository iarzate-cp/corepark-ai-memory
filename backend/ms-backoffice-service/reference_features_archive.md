---
name: Feature archive lives at ~/Dev/Features
description: One MD per feature at ~/Dev/Features/YYYY_MM_DD_<feature>.md — Jira-ticket-shaped write-ups, filename is date + snake_case feature name
type: reference
originSessionId: b094aa38-29e6-42a7-bdf0-66c9d0e65af9
---
# Feature archive — ~/Dev/Features

Per-feature Jira-ticket-shaped write-ups live at:

```
~/Dev/Features/YYYY_MM_DD_<feature>.md
```

- **Date first, underscore-separated:** `YYYY_MM_DD` (e.g. `2026_07_17`). Sorts chronologically in `ls`.
- **Feature name in snake_case:** e.g. `vehicle_info_links`, not `vehicle-info-links` or `vehicleInfoLinks`.
- Full path: `~/Dev/Features/2026_07_17_vehicle_info_links.md`.

## When to write one

When a feature ships (or is being scoped for a Jira ticket) and would benefit from a durable, shareable write-up: acceptance criteria, wire contracts, deploy state, follow-ups, non-obvious design decisions.

## Structure (Jira-ticket-shaped)

- Summary
- Business context
- Acceptance criteria (per persona / flow)
- Technical details: endpoints, wire contracts, envelopes, non-obvious decisions
- Testing
- Branches / deploy state
- Merge-conflict resolutions (if any)
- Follow-ups

## Distinction from `~/Documents`

- `~/Documents` — ad-hoc shareables (operator guides, HTML walkthroughs, one-off deliverables). See `feedback_shareable_docs_location.md`.
- `~/Dev/Features` — the per-feature archive. Structured, dated, one MD per shipped feature.

## Existing entries

- `2026_07_17_vehicle_info_links.md` — see `project_vehicle_info_ecosystem.md`.

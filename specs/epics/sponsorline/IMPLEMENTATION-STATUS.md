# sponsorline — Implementation status

As-built record. This file tracks what actually shipped, kept deliberately
distinct from `design.md` (intent) and `implementation-plan.md` (plan).

## Summary

| Field | Value |
|-------|-------|
| Epic status | **Draft — not started** |
| Product repo | not yet bootstrapped |
| Branch | `epic/deals` (to be created after `flows/phases/00-all`) |
| Milestones shipped | none |
| Live on `stage` | no |
| Live on `prod` | no |
| Demo URL | `sponsorline.orun.dev` (not yet provisioned) |

Nothing has been implemented. This epic carries the charter, the technical
design, and the SL0–SL8 milestone ladder; the first code lands with SL0, after
the product repo is born from the baseline.

## Bootstrap prerequisites

Before SL0 can start, the product must exist:

| Step | What it lands | Time |
|---|---|---|
| `flows/phases/00-all` | the full platform layer — 13 workers, console, infra, CI — live on `stage` and `prod` | ~73 min, unattended |
| `tooling/rebrand/rebrand.mjs --values sponsorline-brand.json --verify` | repo slug, product name, domain, CLI bin, env prefix, worker names, service bindings | minutes |
| `CONSOLE_CUSTOM_DOMAIN` | `sponsorline.orun.dev` on the zone we already control, so phase 07 is not a blocker | — |

## Baseline this epic starts from

Recorded so that "what did the product layer add" stays answerable later. Taken
from the Lumen baseline as of 5 Aug 2026.

| Layer | State at epic open |
|-------|--------------------|
| Workers | 13: `api-edge`, `identity`, `membership`, `projects`, `policy`, `events`, `config`, `metering`, `billing`, `notifications`, `webhooks`, `admin`, `integrations` |
| Console | `web-console-next` live; org surfaces: `api-keys`, `audit`, `billing`, `config`, `invitations`, `members`, `projects`, `settings`, `usage`, `webhooks` |
| Packages | `contracts`, `policy-engine`, `db`, `sdk`, `cli`, `shared`, `testing`, `notifications-client`, `webhook-verifier` |
| Migrations | `000_control` → `190_integrations_delivery_attribution` — product context starts at `200` |
| Policy actions | `ORGANIZATION_ACTIONS` platform-only; no product actions |
| Object storage | **none** — this epic adds the first R2 bucket |
| Environments | `dev` (verify-only), `stage`, `prod`, converging through Orun |
| Composition stack | `oci://ghcr.io/sourceplane/stack-tectonic:0.18.2` (pinned) |

## Milestone log

| ID | Milestone | Status | PR | Verified on | Notes |
|----|-----------|--------|----|-------------|-------|
| SL0 | Contract + data model + R2 | Draft | — | — | |
| SL1 | Opportunity lifecycle | Draft | — | — | |
| SL2 | Cross-org visibility & grants | Draft | — | — | |
| SL3 | Deal rooms + documents | Draft | — | — | |
| SL4 | NDA gate | Draft | — | — | |
| SL5 | Leads | Draft | — | — | |
| SL6 | Edge + SDK + CLI | Draft | — | — | |
| SL7 | Console (4 surfaces) | Draft | — | — | |
| SL8 | Evidence + demo tenant | Draft | — | — | |

## Deviations from design

None recorded. Any implementation that diverges from `design.md` is noted here
with the reason, rather than by silently editing the design.

# Epic: sponsorline

**Collapse "four interconnected applications" into one multi-tenant system with
four surfaces.** A two-sided investment marketplace: sponsors publish
opportunities, an admin spine reviews and approves them, investors discover them
and — after an NDA they actually have to accept — enter a deal room and read the
documents. Every document view is recorded.

Everything below the product line — users, organizations, RBAC, audit,
entitlements, billing, notifications, webhooks, console shell, CI/CD, IaC,
migrations — is already live per environment. This epic adds the one bounded
context that makes the platform a marketplace, and nothing else.

## Status

| Field | Value |
|-------|-------|
| Status | **Draft — not started** |
| Cluster | **SL** (SL0–SL8) — the product bounded context for sponsorline |
| Owner(s) | `apps/deals-worker` (new), `apps/api-edge`, `packages/{contracts,policy-engine,db,sdk,cli}`, `apps/web-console-next` |
| Target branch | `epic/deals` → `main` |
| Builds on | identity/membership, `policy-worker` + `packages/policy-engine`, `@saas/db` + Hyperdrive, `events-worker`, `notifications-worker`, `webhooks-worker`, `admin-worker`, the console foundation |
| New infrastructure | one Cloudflare R2 bucket for deal-room documents (`infra/terraform/cloudflare-r2`) |
| Origin | Upwork job **"Senior Full-Stack Developer / Team — Investment Marketplace Platform (Next.js + Supabase + Sanity)"** — see [`market-context.md`](./market-context.md) |
| Decisions locked | See below |

### Decisions locked

1. **One bounded context, one worker.** `apps/deals-worker` owns opportunities,
   deal rooms, documents, NDA acceptances, access grants, and leads. The four
   "applications" in the brief are four *surfaces* over one system.
2. **Sponsor and investor are role bindings on one org model.** A sponsor org and
   an investor org are both `membership` organizations; what differs is the role
   set and which side of an opportunity they stand on. This single decision is
   what turns three portals into one build.
3. **Cross-org reads are grant-scoped, never unscoped.** A marketplace breaks the
   naive "every query filters on your own `org_id`" invariant, so we replace it
   with a stronger one: *every read is either owner-scoped or grant-scoped.*
   §4 of `design.md` is the whole design.
4. **NDA gating is an authorization decision, not a UI state.** Document access
   requires a recorded NDA acceptance; the check runs in the policy path with
   deny-as-404. A hidden download button is not a gate.
5. **Documents live in R2 behind short-TTL signed URLs** minted per request
   *after* the policy check. No durable public URLs. Every mint is logged with
   who, what, and when.
6. **The opportunity lifecycle is an explicit state machine** —
   `draft → in_review → approved → live → closed`, plus `rejected` and
   `withdrawn` — with every transition recorded append-only. Illegal transitions
   are rejected by a pure engine, not by convention.
7. **The admin portal is `admin-worker` plus console surfaces**, not a fourth
   application.
8. **No CMS dependency in v1.** The console already renders catalog-driven
   markdown; Sanity is a recurring cost and a second content model for no gain
   at this scale. Offered as an optional integration, priced separately.
9. **Audited from day one.** `deals.*` events emit to `events-worker` on every
   mutation and every document access, because "who saw which document, when" is
   the product's trust story rather than a nice-to-have.
10. **Leads are a light sub-domain inside `deals`, not a CRM build.** Capture,
    assign, note, convert. Anything more is an integration with a real CRM.

## Thesis

The brief asks for four interconnected applications — Public Website, Investor
Portal, Sponsor Portal, Admin Portal — over a shared backend, database, and auth,
with deal rooms, NDA-gated documents, an approval workflow, leads, a CMS,
reports, and roles.

Read structurally, three of those four "applications" are the same multi-tenant
system with different role bindings:

| The brief's application | What it actually is |
|---|---|
| Investor Portal | the console, with investor role bindings |
| Sponsor Portal | the console, with sponsor role bindings |
| Admin Portal | `admin-worker` + audited console surfaces |
| Public Website | one storefront route group + lead capture |

And the platform layer under all of it — auth, orgs, seats, roles, audit,
notifications, billing, deploy pipeline — is already running:

| Brief requirement | Where it lives |
|---|---|
| Shared auth across four apps | `identity-worker` — shipped |
| Sponsors and investors as tenants | `membership-worker` — shipped |
| Roles & permissions | `policy-worker` + `packages/policy-engine` — shipped |
| Audit trail | `events-worker` — shipped |
| Email notifications | `notifications-worker` — shipped |
| Admin/support workflows | `admin-worker` — shipped |
| Console shell, org switching, settings | `web-console-next` — shipped |
| **Opportunities, deal rooms, NDA gates, document grants, leads** | **this epic** |

So the commercial move is not to bid a lower price than the agencies. It is to
**collapse the comparison**: show that three of his four applications already
exist as one running system, so his SOW is mostly configuration, not
construction. That reframes the quote from "how many developers × how many
weeks" into "how much of this is already done" — a question we can answer with a
live URL.

Two design commitments carry disproportionate weight:

- **The grant model.** Every other bidder will build a marketplace by loosening
  tenant isolation until cross-org reads work, and will not be able to explain
  what is now readable by whom. We do the opposite: cross-org visibility is a
  first-class, auditable grant, and the test suite proves that an investor
  without a grant gets a 404 rather than a leak.
- **The NDA as a gate you can trip.** The demo ships an opportunity whose deal
  room you can actually fail to enter. That single interaction communicates more
  about the build's seriousness than any architecture diagram.

## Read order

1. `README.md` (this file) — charter and locked decisions.
2. `design.md` — the bounded context, the grant model, data model, lifecycle
   engine, document handling, RBAC, and the explicit non-goals.
3. `implementation-plan.md` — SL0–SL8 with acceptance criteria.
4. `market-context.md` — the originating job, what the client is actually buying,
   and the two-project play.
5. `risks-and-open-questions.md`.
6. `IMPLEMENTATION-STATUS.md` — as-built record (empty until SL0 lands).

## Milestones at a glance

| ID | Milestone | Status |
|----|-----------|--------|
| SL0 | Contract + data model: `@saas/contracts/deals`, migrations `200`/`210`/`220`, manifest entries, policy actions, worker skeleton, R2 bucket | Draft |
| SL1 | Opportunity lifecycle: CRUD + pure state-machine engine + append-only transition record | Draft |
| SL2 | Cross-org visibility: teaser/full/confidential tiers, the grant model, and the "no unscoped read" test suite | Draft |
| SL3 | Deal rooms + documents: R2 storage, per-request signed URLs, append-only access log | Draft |
| SL4 | NDA gate: versioned NDA text, acceptance records, enforcement in the policy path | Draft |
| SL5 | Leads: public capture, assignment, activity timeline, conversion | Draft |
| SL6 | Edge + SDK + CLI: `deals-facade.ts`, rate-limit wiring, full contract parity | Draft |
| SL7 | Console: investor / sponsor / admin route groups + public storefront | Draft |
| SL8 | Evidence + demo tenant: 6 opportunities, 2 sponsors, 3 investors, one mid-approval, one trippable NDA gate, `08-docs` re-run | Draft |

## Scope boundary

| In scope | Out of scope |
|----------|--------------|
| One product bounded context (`deals`); opportunity lifecycle with recorded transitions; three-tier cross-org visibility mediated by explicit grants; deal rooms with R2 documents and per-request signed URLs; versioned NDA acceptance as an authorization gate; append-only document access log; lead capture and assignment; additive RBAC actions; full API/SDK/CLI parity; investor/sponsor/admin console surfaces plus a public storefront; a seeded demo tenant | Sanity or any external CMS (offered as a priced option); e-signature on the NDA (v1 records click-through acceptance with a versioned document — provider e-sign is a named follow-on); investor payments, subscriptions, or fund flows; secondary-market or transaction settlement; document viewer/watermarking beyond signed-URL delivery; portfolio analytics; multi-currency; automated investor accreditation checks |

## Relationship to the platform

- **`membership-worker`** — untouched. Sponsor and investor orgs are ordinary
  organizations; the epic only adds role bindings and actions.
- **`policy-worker`** — receives the new actions plus the NDA and grant
  predicates. This is the only place the marketplace's access rules live.
- **`events-worker`** — receives `deals.*` events including every document
  access. The existing audit console surface picks them up with no change.
- **`admin-worker`** — the approval queue's privileged operations run here so
  they are audited by construction.
- **`api-edge`** — one new facade plus a signed-URL mint route with a stricter
  rate-limit class. No change to existing facades.
- **New infrastructure** — one R2 bucket per environment, provisioned by
  Terraform through the composition stack like every other resource. This is the
  epic's only infrastructure addition.

## Verification bar

The lifecycle engine and the visibility resolver are unit-tested with no
database and no network — including every illegal transition and every
grant/tier combination. The headline suite is **cross-org denial**: an investor
org with no grant, and an investor org with a grant but no NDA acceptance, both
receive 404 on the document, and both attempts land in the access log. Backend
milestones are verified on `stage` via an authenticated CLI walkthrough; console
milestones with an authenticated Playwright walkthrough plus screenshots.

**"Implemented locally" is not a completion state.**

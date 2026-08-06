# sponsorline — Implementation plan

Milestones SL0–SL8. Each is independently shippable and independently verifiable
on `stage`. "Done when" is the acceptance bar; a milestone that cannot be
demonstrated on a deployed environment is not done.

**Demo** is the flagship demo build (seeded data, enough depth to survive a
click-through). **Client** is a delivered engagement with real documents, real
volumes, and handover documentation.

| ID | Milestone | Demo | Client |
|----|-----------|------|--------|
| SL0 | Contract + data model + R2 | 3 h | 1.5 d |
| SL1 | Opportunity lifecycle | 3 h | 1.5 d |
| SL2 | Cross-org visibility & the grant model | 4 h | 2 d |
| SL3 | Deal rooms + documents | 4 h | 2 d |
| SL4 | NDA gate | 2 h | 1 d |
| SL5 | Leads | 2 h | 1 d |
| SL6 | Edge + SDK + CLI | 2 h | 1 d |
| SL7 | Console (4 surfaces) | 8 h | 4.5 d |
| SL8 | Evidence + demo tenant | 3 h | 1.5 d |
| | **Total** | **~4 working days** | **~16 working days** |

---

## SL0 — Contract, data model, and storage

**Lands:** `packages/contracts/src/deals.ts` (types + validators for
opportunities, transitions, rooms, documents, grants, NDA acceptances, access
log, leads); migrations `200_deals_core`, `210_deals_rooms`, `220_deals_leads`
with `up.sql` per directory; three `packages/db/src/manifest.ts` entries with
sha256 checksums; `packages/db/src/deals/` repositories; the new actions
registered in `@saas/contracts/policy` and `@saas/policy-engine`;
`infra/terraform/cloudflare-r2` provisioning one bucket per environment; the
`apps/deals-worker` skeleton with the R2 binding and a health route.

**Done when:**

- `kiox -- orun validate --intent intent.yaml` passes with both new components
  discovered.
- `db-migrate` plans cleanly on the PR and applies on merge; the runner accepts
  all three manifest entries.
- Terraform provisions the R2 bucket on `stage` and `prod`; the worker can `PUT`
  and `HEAD` an object through its binding.
- `tests/policy-engine` proves every new action resolves for the sponsor and
  investor role matrices in `design.md` §7, and that an unregistered action
  denies with `unknown_action`.
- No existing component's behaviour changes.

---

## SL1 — Opportunity lifecycle

**Lands:** `engine/lifecycle.ts`; `GET`/`POST /opportunities`,
`GET`/`PATCH /opportunities/:id`, `POST /opportunities/:id/transitions`,
`GET /opportunities/:id/transitions`; the append-only transition record;
`deals.opportunity.*` events.

**Done when:**

- The engine is unit-tested over every legal transition and every illegal one,
  including that `reject` and `withdraw` require a reason.
- An illegal transition returns a typed `invalid_transition` error naming the
  current status and the legal set.
- Every transition writes a `opportunity_transitions` row **before** the
  denormalised `status` changes; a verifier test asserts the two always agree.
- The system close-at-`closes_at` path writes a transition with a null actor.
- A CLI walkthrough drives one opportunity `draft → in_review → approved → live`
  on stage and prints the transition history.

---

## SL2 — Cross-org visibility and the grant model

**The milestone the whole epic rests on.**

**Lands:** `engine/visibility.ts`; `GET /marketplace/opportunities` (the
grant-scoped cross-org listing); `GET`/`POST /opportunities/:id/grants` and
revoke; the teaser-tier read path used by the storefront; the cross-org denial
suite.

**Done when:**

- The resolver is unit-tested across the full tier × grant × NDA × status
  matrix, returning both a tier and a reason, with no database and no clock.
- **Every case in `design.md` §4.4 passes**, and each denial asserts both the
  404 and the access-log row.
- Tier-gated fields are **absent from the response payload**, not merely
  unrendered — a test inspects the raw JSON for `summary` on a teaser-tier read.
- A grant revoked mid-session removes marketplace visibility on the next read.
- Every query added in this milestone is annotated in review as owner-scoped or
  grant-scoped; a third shape fails review.

---

## SL3 — Deal rooms and documents

**Lands:** `storage.ts`; room creation; document upload via short-lived
pre-signed PUT with post-upload `HEAD` + checksum confirmation;
`GET …/documents`; `POST …/documents/:docId/url` with mint-time tier
re-resolution; archive; the append-only access log; the orphaned-object sweep;
`deals.document.*` events.

**Done when:**

- `storage_key` never appears in any API response — asserted by a
  response-shape test across every document route.
- A signed URL expires at the configured TTL; an expired URL returns the
  provider's denial, not the object.
- A grant revoked between `list` and `mint` yields 404 at mint.
- A client-supplied `storage_key` or a guessed key from another org is never
  honoured.
- An upload whose `HEAD` fails or whose checksum mismatches leaves no row, and
  the object is swept.
- Both `mint` and `denied` outcomes appear in the access log with the reason.

---

## SL4 — The NDA gate

**Lands:** versioned NDA text storage and rendering; `GET /opportunities/:id/nda`
returning text + version + digest; `POST …/nda/acceptances`; enforcement wired
into the visibility resolver; `deals.nda.accepted` events; the sponsor
notification.

**Done when:**

- Acceptance records the version **and** the sha256 digest of the exact rendered
  text, plus hashed IP and user agent — never raw.
- A room whose `nda_version` is bumped re-gates every prior acceptor until they
  accept again; a test proves an investor at v1 gets 404 after a bump to v2.
- Only owner/admin roles in an investor org can accept — a viewer attempting it
  is denied, because acceptance binds the organization.
- The denial reason surfaced to the console is `no_nda`, distinct from
  `no_grant`, so the UI can offer the right next action.

---

## SL5 — Leads

**Lands:** `GET`/`POST /leads`, `PATCH /leads/:id`; the unauthenticated
`POST /public/leads` with per-IP rate limiting; assignment and status changes;
the append-only activity timeline; conversion to an investor org;
`deals.lead.created` events; the owner notification.

**Done when:**

- A storefront submission creates a lead and notifies the owner; the endpoint is
  rate-limited per IP and rejects oversized bodies.
- Status and owner changes write activity rows with typed metadata.
- Converting a lead links `converted_org_id` and leaves the lead row intact.
- The public endpoint cannot read anything — it is write-only by construction.

---

## SL6 — Edge, SDK, CLI

**Lands:** `apps/api-edge/src/deals-facade.ts` wired into the dispatch chain;
`DEALS_WORKER` binding; strict rate-limit classes on document URL minting and
`POST /public/leads`; `packages/sdk` `deals.*`; `packages/cli` `opportunities` /
`documents` / `grants` / `leads` command groups.

**Done when:**

- Every route in `design.md` §6 is reachable through the public edge with
  tenancy resolution, idempotency, and rate limiting applied.
- The SDK surface is generated against the contracts module — no hand-written
  types that can drift.
- One authenticated CLI walkthrough on stage performs: create → submit →
  approve → publish → grant → accept NDA → mint URL → revoke → verify denial.
  That transcript is the milestone evidence.
- The facade stays under ~120 LOC; anything larger means logic leaked into the
  edge.

---

## SL7 — Console

**Lands:** the four surfaces in `design.md` §9 on the existing design system,
over the live API via the SDK.

**Done when:**

- Lifecycle buttons render from the engine's legal-transition set, not a
  hard-coded list — adding a transition server-side changes the UI with no
  console edit.
- The investor deal room shows the *reason* it is locked (`no_grant` vs `no_nda`)
  and offers the matching action.
- The NDA modal renders the exact versioned text whose digest gets recorded.
- The sponsor access log is filterable by investor org and shows denials.
- Every surface has empty, loading, error, and denied states.
- An authenticated Playwright walkthrough passes on stage, including tripping
  the NDA gate and then clearing it; screenshots attached.

---

## SL8 — Evidence and demo tenant

**Lands:** the seeded demo — **2 sponsor orgs, 3 investor orgs, 6
opportunities** spanning `draft`, `in_review` (one mid-approval, sitting in the
admin queue), `live`, and `closed`; one deal room with a **trippable NDA gate**;
one investor holding a `confidential` grant and one holding only `full`, so the
classification boundary is visible; a populated access log including denials;
`docs/{overview,architecture,runbook}.md`; `catalog.entities` enrichment; an
`08-docs` re-run.

**Done when:**

- `https://sponsorline.orun.dev` serves the storefront with teaser-tier
  opportunities and working lead capture.
- Three logins are in the proposal body — sponsor, investor, admin — and each
  lands somewhere immediately legible.
- The NDA gate can be tripped by the reviewer in under a minute from the
  investor login, and the resulting denial is visible in the sponsor's access
  log.
- `ai/context/deployment.md` and `operations.md` are regenerated against
  verified live state.
- **`docs/overview.md` is read with fresh eyes and describes *this* product** —
  the known failure mode is docs rebranded independently of the domain, and
  shipping that to a buyer costs exactly the credibility the docs were meant to
  buy.

---

## Sequencing notes

- SL0 blocks everything. SL1 → SL2 is a hard chain; SL3 needs SL2's resolver;
  SL4 modifies the resolver, so it must land before SL7's investor surface.
- SL5 is independent of SL1–SL4 and can run in parallel.
- SL6 should start once SL1 lands and grow with each milestone — the CLI
  walkthrough is the verification mechanism for SL1–SL5, so deferring it to the
  end removes the way we check the earlier work.
- SL7 needs SL6's SDK surface.
- **The demo cut, if time is short:** SL5 (leads) is the one milestone a
  click-through survives without. Everything else is load-bearing for the
  headline interaction.
- **Never** hand-deploy. A `wrangler deploy` or `terraform apply` by hand is
  drift the next plan will fight.

# sponsorline — Design

Status: Ready for implementation. This is the technical design for the `deals`
product bounded context.

## 1. The shape of the problem

The brief names four applications over a shared backend: Public Website,
Investor Portal, Sponsor Portal, Admin Portal — with deal rooms, NDA-gated
document access, an opportunity review and approval workflow, lead and CRM
management, a CMS, reports, and roles and permissions.

Decomposed, that is one system with four surfaces and exactly one genuinely new
problem:

| Capability | Owner |
|---|---|
| Who you are, which org you belong to, what you may do | platform — shipped |
| Sponsors and investors as separate tenants | platform — shipped |
| Approval workflow's audit trail, email, admin controls | platform — shipped |
| Publish an opportunity, move it through review, list it | **new** — routine CRUD plus a state machine |
| Let an investor from *another organization* read some of it | **new** — the actual problem |
| Gate documents behind an NDA they must accept | **new** — an authorization decision |
| Record who read which document, when | **new** — an append-only log |

Almost every capability in the brief is either already running or is
conventional. The one hard part is the second row, and it deserves the design
attention, because getting it wrong is how marketplaces leak.

**The tension.** The platform's tenancy invariant is *every row carries `org_id`,
every query filters on it.* That invariant is exactly what a marketplace has to
break: an investor in org B must read an opportunity owned by sponsor org A.
Most implementations resolve this by weakening the filter — adding an
`is_public` flag, or dropping the `org_id` predicate on "marketplace" queries —
and from that moment nobody can say precisely what is readable by whom.

**The resolution.** We do not weaken the invariant; we replace it with a
stronger, still-checkable one:

> **Every read is either owner-scoped or grant-scoped. There is no unscoped read.**

Owner-scoped means the reading org owns the row. Grant-scoped means a row exists
that names the reading org and what it may see. There is no third case, and
§4.4 describes the test suite that proves it.

## 2. Bounded context: `deals`

One new Cloudflare Worker, `apps/deals-worker`, owning one Postgres schema
(`deals`). It mirrors the `projects-worker` anatomy:

```
apps/deals-worker/
  component.yaml            spec.type: cloudflare-worker-turbo
                            dependsOn: [membership-worker, policy-worker,
                                        events-worker, notifications-worker]
                            providesApis: [deals-api]
                            consumesApis: [membership-api, policy-api,
                                           events-api, notifications-api]
                            subscribe.environments: dev/stage/prod, profile: verify
                            profileRules: profile deploy when triggerRef github-push-main
  wrangler.template.jsonc   + R2 binding: DEAL_DOCUMENTS
  wiring.fixture.json
  src/
    index.ts  router.ts  env.ts  http.ts  ids.ts  pagination.ts
    membership-client.ts  policy-client.ts  events-client.ts
    storage.ts              R2 put/head/delete + signed-URL minting
    handlers/*.ts           one file per operation
    engine/                 pure domain logic, no I/O — the testable core
      lifecycle.ts          the opportunity state machine
      visibility.ts         tier + grant + NDA → what this org may see
tests/deals-worker/         contract + verifier suites
```

**Why one worker.** Opportunities, rooms, documents, grants, NDA acceptances,
and leads all hang off the same opportunity aggregate, and the visibility
resolver needs all of them in one decision. Splitting would put a cross-worker
join on the authorization path — the worst possible place for one. The seam that
matters here is pure-engine vs I/O, enforced by `engine/`.

**Why `admin-worker` is not extended.** The approval *decision* is a `deals`
state transition and belongs here. What runs in `admin-worker` is the privileged
override — force-withdraw a live opportunity, revoke a grant on request — where
audited-by-construction matters more than convenience.

## 3. Data model

Three migrations, schema `deals`. The platform's last migration is
`190_integrations_delivery_attribution`, so the product block starts at `200`.
`org_id` is an opaque UUID owned by `membership`; there are no cross-context
foreign keys.

### `200_deals_core`

#### `deals.opportunities`

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `opp_<hex>` |
| `org_id` | `UUID NOT NULL` | **the sponsor org that owns it** |
| `slug` | `TEXT NOT NULL` | URL-safe, unique per org |
| `title` | `TEXT NOT NULL` | |
| `teaser` | `TEXT NOT NULL` | the always-visible summary — see §4.1 |
| `summary` | `TEXT` | full description, tier-gated |
| `sector` / `geography` | `TEXT` | facets for discovery |
| `target_raise_cents` / `min_ticket_cents` | `BIGINT` | integer cents, single currency in v1 |
| `currency` | `CHAR(3) NOT NULL` | ISO 4217 |
| `status` | `TEXT NOT NULL DEFAULT 'draft'` | `CHECK IN ('draft','in_review','approved','live','closed','rejected','withdrawn')` |
| `visibility` | `TEXT NOT NULL DEFAULT 'private'` | `CHECK IN ('private','teaser','full')` — the *maximum* tier any investor may reach without a grant |
| `published_at` / `closes_at` | `TIMESTAMPTZ` | |
| `created_by` | `UUID NOT NULL` | |
| `created_at` / `updated_at` | `TIMESTAMPTZ` | |

Unique `(org_id, slug)`. Index `(status, visibility, published_at DESC, id DESC)`
for the marketplace listing — note this index is deliberately *not*
`org_id`-leading, because the listing is a grant-scoped read across orgs (§4.2).

#### `deals.opportunity_transitions` — append-only

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `trn_<hex>` |
| `org_id` | `UUID NOT NULL` | the owning sponsor org |
| `opportunity_id` | `UUID NOT NULL` | |
| `from_status` / `to_status` | `TEXT NOT NULL` | |
| `actor_user_id` | `UUID` | null for system transitions (e.g. auto-close at `closes_at`) |
| `actor_org_id` | `UUID` | which org acted — sponsor submits, admin approves |
| `reason` | `TEXT` | required on `rejected` and `withdrawn` |
| `created_at` | `TIMESTAMPTZ NOT NULL` | |

Index `(org_id, opportunity_id, created_at DESC)`. **Never updated.** This table
*is* the approval workflow's audit; the current status on the opportunity is a
denormalised convenience, and a verifier test asserts the two agree.

### `210_deals_rooms`

#### `deals.deal_rooms`

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `room_<hex>` |
| `org_id` | `UUID NOT NULL` | owning sponsor org |
| `opportunity_id` | `UUID NOT NULL` | one room per opportunity in v1; the schema permits more |
| `name` | `TEXT NOT NULL` | |
| `nda_required` | `BOOLEAN NOT NULL DEFAULT true` | |
| `nda_version` | `INT` | which NDA text applies; null when not required |
| `created_at` | `TIMESTAMPTZ` | |

#### `deals.documents`

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `doc_<hex>` |
| `org_id` | `UUID NOT NULL` | owning sponsor org |
| `room_id` | `UUID NOT NULL` | |
| `name` | `TEXT NOT NULL` | display name |
| `storage_key` | `TEXT NOT NULL` | R2 object key — **never returned to a client** |
| `content_type` / `size_bytes` / `checksum` | | sha256 of the object |
| `classification` | `TEXT NOT NULL` | `CHECK IN ('teaser','full','confidential')` — the tier required to read it |
| `uploaded_by` | `UUID NOT NULL` | |
| `created_at` / `archived_at` | `TIMESTAMPTZ` | soft delete |

Index `(org_id, room_id, created_at DESC)` partial `WHERE archived_at IS NULL`.

#### `deals.nda_acceptances`

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `nda_<hex>` |
| `org_id` | `UUID NOT NULL` | the owning sponsor org (the NDA's counterparty) |
| `opportunity_id` | `UUID NOT NULL` | |
| `investor_org_id` | `UUID NOT NULL` | who accepted |
| `accepted_by_user_id` | `UUID NOT NULL` | which human clicked |
| `nda_version` | `INT NOT NULL` | the exact text version accepted |
| `nda_digest` | `TEXT NOT NULL` | sha256 of the rendered NDA text — proves *what* was agreed |
| `accepted_at` | `TIMESTAMPTZ NOT NULL` | |
| `ip_hash` / `user_agent_hash` | `TEXT` | hashed, never raw |

Unique `(opportunity_id, investor_org_id, nda_version)`. An NDA version bump
therefore re-gates every investor — deliberate, and the reason `nda_version`
exists.

#### `deals.access_grants`

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `grt_<hex>` |
| `org_id` | `UUID NOT NULL` | granting sponsor org |
| `opportunity_id` | `UUID NOT NULL` | |
| `investor_org_id` | `UUID NOT NULL` | grantee |
| `max_classification` | `TEXT NOT NULL` | `CHECK IN ('teaser','full','confidential')` |
| `granted_by` | `UUID NOT NULL` | |
| `granted_at` / `revoked_at` | `TIMESTAMPTZ` | |
| `revoked_by` / `revoke_reason` | | |

Partial unique `(opportunity_id, investor_org_id) WHERE revoked_at IS NULL`.
Revocation is a timestamp, not a delete — "who could see this in March" stays
answerable.

#### `deals.document_access_log` — append-only

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `acc_<hex>` |
| `org_id` | `UUID NOT NULL` | owning sponsor org |
| `document_id` | `UUID NOT NULL` | |
| `actor_user_id` / `actor_org_id` | `UUID NOT NULL` | who asked |
| `action` | `TEXT NOT NULL` | `CHECK IN ('list','mint','denied')` |
| `denial_reason` | `TEXT` | `no_grant`, `no_nda`, `classification`, `archived` |
| `url_expires_at` | `TIMESTAMPTZ` | on `mint` |
| `created_at` | `TIMESTAMPTZ NOT NULL` | |

Index `(org_id, document_id, created_at DESC)`; index
`(actor_org_id, created_at DESC)`.

**Denials are logged, not just successes.** A sponsor asking "did anyone try to
get at my documents" is a real question, and the answer is a query.

### `220_deals_leads`

#### `deals.leads`

| Column | Type | Notes |
|--------|------|-------|
| `id` | `UUID` PK | `led_<hex>` |
| `org_id` | `UUID NOT NULL` | the org the lead belongs to (sponsor, or the platform org for site-wide enquiries) |
| `opportunity_id` | `UUID` | nullable — a general enquiry has none |
| `name` / `email` / `company` / `phone` | `TEXT` | |
| `message` | `TEXT` | |
| `source` | `TEXT NOT NULL` | `storefront`, `opportunity_page`, `referral`, `manual` |
| `status` | `TEXT NOT NULL DEFAULT 'new'` | `CHECK IN ('new','contacted','qualified','converted','archived')` |
| `owner_user_id` | `UUID` | assignee |
| `converted_org_id` | `UUID` | set when the lead becomes an investor org |
| `created_at` / `updated_at` | | |

Index `(org_id, status, created_at DESC)`.

#### `deals.lead_activities` — append-only

`id` `led_act_<hex>`, `org_id`, `lead_id`, `kind`
(`note`/`status_change`/`owner_change`/`email_sent`), `actor_user_id`, `body`,
`metadata` JSONB, `created_at`.

Each migration adds an entry to `packages/db/src/manifest.ts` (id, context
`deals`, path, sha256, description). The runner refuses an unlisted or drifted
file.

## 4. The visibility model — the heart of the epic

### 4.1 Three tiers

Every readable thing carries a classification, and every reader resolves to a
maximum tier:

| Tier | What it exposes | Who reaches it |
|------|-----------------|----------------|
| `teaser` | title, teaser text, sector, geography, target raise | any authenticated member of any investor org, once the opportunity is `live` and `visibility ≥ teaser` |
| `full` | summary, financial detail, `full`-classified documents | requires an active grant **and**, if the room requires one, a current NDA acceptance |
| `confidential` | `confidential`-classified documents | requires an active grant whose `max_classification` is `confidential`, plus the NDA |

The owning sponsor org always reaches `confidential` on its own rows. Platform
admins reach it through `admin-worker`, audited.

### 4.2 The resolver

`engine/visibility.ts` is pure:

```ts
resolveTier(input: {
  readerOrgId: string
  ownerOrgId: string
  opportunityStatus: Status
  opportunityVisibility: Tier
  grant: { maxClassification: Tier; revokedAt: string | null } | null
  ndaRequired: boolean
  ndaVersionRequired: number | null
  ndaAcceptance: { ndaVersion: number } | null
}): { tier: Tier | 'none'; reason: string }
```

No clock, no I/O, no database. It returns the tier *and the reason*, and the
reason string is what lands in `denial_reason` on the access log and in the
console's explanation to the user. Every branch is unit-tested, including the
combinations that should be impossible.

Reads then work in exactly two shapes:

- **Owner-scoped:** `WHERE org_id = $readerOrg` — the sponsor's own dashboard.
- **Grant-scoped:** the marketplace listing joins
  `opportunities → access_grants` on `investor_org_id = $readerOrg`, or, for the
  teaser tier, filters `status = 'live' AND visibility >= 'teaser'` — a
  membership-gated read of deliberately public-facing columns only.

There is no third shape. A code review question for every new query in this
context is: *which of the two is this?*

### 4.3 Document delivery

`storage_key` never leaves the worker. Reading a document is a two-step dance:

1. `GET /v1/organizations/:orgId/opportunities/:oppId/documents` — returns
   metadata for documents the reader's tier permits. Logged as `list`.
2. `POST …/documents/:docId/url` — re-resolves the tier *at mint time*, then
   returns a signed R2 URL with a short TTL (default 120s, configurable per
   environment). Logged as `mint` with `url_expires_at`.

Re-resolving at mint time is the point: a grant revoked between the list and the
click does not yield a working URL. The TTL is short enough that a leaked URL
expires before it is useful and long enough for a real download to start.

Uploads are the mirror image: the worker checks
`organization.document.upload`, mints a short-lived pre-signed PUT, and records
the row only after a successful `HEAD` confirms the object and its checksum.
Orphaned objects are swept by a scheduled job.

### 4.4 The test suite that proves the invariant

`tests/deals-worker` carries a dedicated cross-org suite, and it is the
milestone gate for SL2:

| Case | Expected |
|---|---|
| Investor org, no grant, `live` + `teaser` opportunity | teaser fields only; `full` fields absent from the payload, not merely hidden |
| Investor org, no grant, `draft` opportunity | 404 |
| Investor org, grant, NDA required, no acceptance | 404 on documents; `denied/no_nda` in the access log |
| Investor org, grant `max=full`, valid NDA, `confidential` document | 404; `denied/classification` logged |
| Investor org, grant revoked after listing | mint returns 404 |
| Investor org, NDA accepted at v1, room bumped to v2 | 404 until re-acceptance |
| Any org, another org's `storage_key` guessed | 404 — keys are never client-supplied |
| Sponsor org, own rows | full access, no grant needed |

Every denial asserts **both** the 404 *and* the access-log row. A denial that
isn't recorded is a bug in this product.

## 5. The opportunity lifecycle

`engine/lifecycle.ts` is a pure state machine:

```
draft ──submit──▶ in_review ──approve──▶ approved ──publish──▶ live ──close──▶ closed
  ▲                   │                                          │
  └────reject─────────┘                                          │
  └────────────────────withdraw (from draft|in_review|approved|live)──▶ withdrawn
```

| Transition | Action | Who |
|---|---|---|
| `submit` | `organization.opportunity.submit` | sponsor admin |
| `approve` / `reject` | `organization.opportunity.approve` / `.reject` | platform admin |
| `publish` | `organization.opportunity.publish` | sponsor admin (only from `approved`) |
| `close` | `organization.opportunity.close` | sponsor admin, or system at `closes_at` |
| `withdraw` | `organization.opportunity.withdraw` | sponsor admin or platform admin |

`reject` and `withdraw` require a reason. Every transition writes a
`opportunity_transitions` row and emits a `deals.opportunity.*` event before the
denormalised `status` is updated, so the log can never be behind the state.
Illegal transitions return a typed `invalid_transition` error naming the
current status and the legal set — the console renders the legal set as the
available buttons rather than hard-coding them.

Publishing to `live` is also the moment `visibility` takes effect; a `live`
opportunity with `visibility = 'private'` is legal and means "approved, listed
for nobody" — useful for a sponsor staging a launch.

## 6. API surface

All routes under `/v1/organizations/:orgId/`, behind the standard three-step
gate (`fetchAuthorizationContext` → `authorizeViaPolicy` → tier resolution) with
deny-as-404.

| Method | Path | Action |
|--------|------|--------|
| `GET` / `POST` | `/opportunities` | `organization.opportunity.read` / `.write` |
| `GET` / `PATCH` | `/opportunities/:id` | `.read` / `.write` |
| `POST` | `/opportunities/:id/transitions` | the per-transition action from §5 |
| `GET` | `/opportunities/:id/transitions` | `.read` |
| `GET` | `/marketplace/opportunities` | `organization.marketplace.browse` — the grant-scoped cross-org listing |
| `GET` / `POST` | `/opportunities/:id/room/documents` | `organization.document.read` / `.upload` |
| `POST` | `/opportunities/:id/room/documents/:docId/url` | `organization.document.read` |
| `POST` | `/opportunities/:id/room/documents/:docId/archive` | `organization.document.archive` |
| `GET` | `/opportunities/:id/nda` | `organization.marketplace.browse` — fetch text + version |
| `POST` | `/opportunities/:id/nda/acceptances` | `organization.nda.accept` |
| `GET` / `POST` | `/opportunities/:id/grants` | `organization.grant.read` / `.write` |
| `DELETE` | `/opportunities/:id/grants/:grantId` | `organization.grant.write` (revoke) |
| `GET` | `/opportunities/:id/access-log` | `organization.access_log.read` |
| `GET` / `POST` | `/leads` · `PATCH /leads/:id` | `organization.lead.read` / `.write` |
| `POST` | `/public/leads` | unauthenticated storefront capture — rate-limited, see below |

**Edge.** One new facade, `apps/api-edge/src/deals-facade.ts`, registered in the
dispatch chain and bound to `DEALS_WORKER`, following `project-facade.ts`. Two
additions to `rate-limit.ts`: a strict class on document URL minting (the
expensive, security-sensitive route) and a strict per-IP class on
`POST /public/leads`, the only unauthenticated write in the product.

**SDK and CLI.** `packages/sdk` gains `deals.*` generated against the contracts
module; `packages/cli` gains `sponsorline opportunities`, `… documents`,
`… grants`, and `… leads`. The authenticated CLI walkthrough is the stage
verification for SL1–SL5.

## 7. RBAC

New actions in `@saas/contracts/policy` and `@saas/policy-engine`. An
unregistered action denies with `unknown_action`.

Two role families over the same membership model. **Sponsor org:**

| Action | owner | admin | member | viewer |
|---|:--:|:--:|:--:|:--:|
| `organization.opportunity.read` | ✓ | ✓ | ✓ | ✓ |
| `organization.opportunity.write` | ✓ | ✓ | ✓ | — |
| `organization.opportunity.submit` | ✓ | ✓ | — | — |
| `organization.opportunity.publish` / `.close` / `.withdraw` | ✓ | ✓ | — | — |
| `organization.document.upload` / `.archive` | ✓ | ✓ | ✓ | — |
| `organization.grant.read` | ✓ | ✓ | ✓ | ✓ |
| `organization.grant.write` | ✓ | ✓ | — | — |
| `organization.access_log.read` | ✓ | ✓ | — | — |
| `organization.lead.read` / `.write` | ✓ | ✓ | ✓ | read only |

**Investor org:** `organization.marketplace.browse` (all roles),
`organization.nda.accept` (owner/admin only — accepting an NDA binds the
organization, so it is not a viewer's decision), `organization.document.read`
(all roles, subject to tier).

**Platform admin:** `organization.opportunity.approve` / `.reject` plus
override withdraw and grant revocation, exercised through `admin-worker` so
every use is audited.

## 8. Events, notifications, and the storefront

**Events** (`events-worker` audit + `webhooks-worker` delivery):
`deals.opportunity.created` · `.submitted` · `.approved` · `.rejected` ·
`.published` · `.closed` · `.withdrawn` · `deals.nda.accepted` ·
`deals.grant.granted` · `.revoked` · `deals.document.uploaded` ·
`deals.document.accessed` · `deals.document.access_denied` · `deals.lead.created`

**Notifications** (`notifications-worker`, preference-controlled):

- to platform admins — an opportunity entered `in_review`
- to the sponsor — approved, rejected (with reason), or a new NDA acceptance
- to granted investors — an opportunity they can see went `live`, or new
  documents landed in a room they hold a grant on
- to the lead owner — a new lead on their opportunity

**Storefront.** A public route group (`src/app/sponsorline/`) with the marketing
surface, a teaser-tier opportunity index, individual teaser pages, and lead
capture. It is the only unauthenticated ingress and it reads exclusively
`teaser`-tier columns through a dedicated read path — the same resolver, with a
null reader org.

## 9. Console

| Surface | Route | What it is |
|---|---|---|
| Sponsor | `(app)/orgs/[orgSlug]/opportunities` | list, editor, submit-for-review, lifecycle buttons driven by the legal-transition set, room manager, grant manager, access log |
| Investor | `(app)/orgs/[orgSlug]/marketplace` | grant-scoped browse, teaser detail, NDA modal (the gate), deal room with document list and download |
| Admin | `(app)/orgs/[orgSlug]/review-queue` | approval queue with side-by-side opportunity detail, approve/reject with reason, full transition history |
| Public | `src/app/sponsorline/` | marketing, teaser index, lead capture |

Existing `audit` and `members` pages pick up `deals.*` events and the new role
bindings with no change.

**The demo's headline interaction:** log in as an investor who has *not* accepted
the NDA, open a deal room, and be refused — then accept and watch the same
document become available, with both events visible in the sponsor's access log.
That is the whole architecture in fifteen seconds.

## 10. Non-goals (v1)

- Sanity or any external CMS — the console renders catalog-driven markdown;
  offered as a priced option.
- Provider e-signature on the NDA. v1 records click-through acceptance with a
  versioned, digest-pinned document — legally meaningful and far cheaper. Real
  e-sign is a named follow-on and reuses the adapter pattern.
- Investor payments, commitments, fund flows, settlement.
- Document watermarking, in-browser DRM viewers, or download prevention. Signed
  short-TTL URLs plus a complete access log is the honest security posture; DRM
  theatre is not.
- Portfolio analytics, secondary market, automated accreditation checks.
- Multi-currency.

## 11. Verification bar

| Layer | How it is verified |
|-------|--------------------|
| `engine/visibility.ts` | unit tests over the full tier × grant × NDA × status matrix, no DB |
| `engine/lifecycle.ts` | unit tests over every legal and illegal transition, including reason requirements |
| Cross-org denial | the dedicated suite in §4.4 — 404 **and** an access-log row for every denial |
| Worker routes | contract suite including deny-as-404 for every action |
| Storage | signed URL expires as configured; a revoked grant between list and mint yields 404; a client-supplied `storage_key` is never honoured |
| Stage | authenticated CLI walkthrough: create → submit → approve → publish → grant → accept NDA → mint URL → revoke → verify denial |
| Console | authenticated Playwright walkthrough with screenshots, including the NDA gate trip |
| Prod | smoke check after promotion |

"Implemented locally" is not a completion state.

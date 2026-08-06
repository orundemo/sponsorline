# sponsorline — Risks and open questions

## Risks

### Product and technical

**A leaked document is the failure that ends the engagement.** This is a product
whose entire value is controlled disclosure; one investor seeing another
sponsor's confidential deck is not a bug report, it is the end of the client
relationship. *Mitigation:* the two-shape read rule (owner-scoped or
grant-scoped, never a third), tier re-resolution at mint time rather than list
time, short-TTL signed URLs, `storage_key` never crossing the API boundary, and
the §4.4 denial suite as a hard milestone gate rather than a nice-to-have.

**Denials that aren't logged.** A silent 404 is indistinguishable from a bug, and
a sponsor asking "did anyone try to open my documents" deserves an answer.
*Mitigation:* `denied` is a first-class `action` on the access log with a typed
reason; every denial test asserts the log row as well as the status code.

**NDA version bumps re-gate everyone.** This is intended, but it will surprise a
sponsor who edits a typo and locks out twelve investors. *Mitigation:* the
console warns explicitly at bump time and shows the count of acceptances that
will be invalidated; a future minor-edit path (digest changes, version doesn't)
is a named follow-on, deliberately not v1.

**Click-through NDA acceptance may not satisfy the client's counsel.** We record
version, digest, timestamp, hashed IP, and the accepting user — meaningful
evidence, but not a signed instrument. *Mitigation:* say so plainly in the
proposal rather than letting it surface in week four; the e-sign adapter is
scoped and priced as a follow-on that reuses the provider-seam pattern.

**Orphaned R2 objects.** An upload that fails after the pre-signed PUT but
before the row write leaves a paid-for object with no owner. *Mitigation:* the
post-upload `HEAD` + checksum confirmation, and a scheduled sweep of objects
with no corresponding row older than a short horizon.

**The marketplace listing index is not `org_id`-leading.** That is deliberate
(§4.2) but it is exactly the query a careless future change could turn into an
unscoped read. *Mitigation:* the query-shape review rule, and a test that runs
the listing as an org with zero grants and asserts only `live` + `teaser` rows
with teaser-tier columns come back.

### Delivery and commercial

**$3,500 is under-scoped for four portals and we should not pretend otherwise.**
*Mitigation:* quote a fixed Phase 1 (platform live + investor/sponsor portals +
auth/roles + admin shell) at his number, and Phase 2 (deal rooms, approval
workflow, CMS if he insists, reports) separately. Pretending the whole SOW fits
$3,500 is how a fixed-price engagement turns into an unpaid month.

**0% hire rate across 5 posts, 6 people interviewing.** He is collecting quotes,
possibly to price a build he intends to resell. *Mitigation:* cap exposure with
a small, demo-backed Phase 1; do not invest proposal effort beyond the demo we
have already built; treat the sibling travel-platform job as the real prize and
quote both.

**The Sanity pushback could read as scope-dodging.** We are declining a named
requirement. *Mitigation:* frame it as a cost decision with his interests in
front — a second content model and a recurring subscription for markdown the
console already renders — and offer it as a priced option rather than a refusal.
If he wants it, it is a small integration, not an argument.

**NDA before SOW.** He shares the SOW and designs only post-NDA, which means we
are quoting partly blind. *Mitigation:* sign fast — it filters out half the
competition — and treat anything discovered in the SOW that is not in this epic
as a Phase 2 line item, in writing, before Phase 1 starts.

**Docs that contradict the product.** The known failure mode from the reference
build: `docs/overview.md` describing one domain while the code implements
another, because docs were rebranded independently. *Mitigation:* SL8 re-runs
`08-docs` after the domain lands and the overview is read fresh before any
workspace link is shared.

**Repo hygiene.** `.orun/` is generated, gitignored, and can reach hundreds of
megabytes — never in a client clone. Composition contracts stay **pinned** to an
OCI tag, so upgrades are a one-line bump rather than a merge.

## Open questions

| # | Question | Owner | Needed by |
|---|----------|-------|-----------|
| 1 | One deal room per opportunity, or several (e.g. a public room and a diligence room)? The schema permits many; the console cost is a room switcher. | product | SL3 |
| 2 | Signed-URL TTL — 120s is the proposed default. Long enough for a 200 MB download on a poor connection? Consider TTL by size. | eng | SL3 |
| 3 | Does an investor org self-serve a grant request, or does the sponsor always initiate? A request flow adds a small state machine and is probably what a real marketplace needs. | product | SL2 |
| 4 | Who owns platform-wide leads — a dedicated platform org, or `org_id` nullable on `deals.leads`? Nullable weakens the tenancy invariant; a platform org keeps it. | eng | SL5 |
| 5 | Is the admin approval single-reviewer or two-person? The transition record supports either; two-person needs a second state. | product | SL1 |
| 6 | Single currency for v1 — which one, and does the sibling travel job change the answer? | product | SL0 |
| 7 | Does the storefront index every `live` + `teaser` opportunity across all sponsors, or only a curated set? Curation is another admin surface. | product | SL7 |
| 8 | Document virus scanning on upload — required for a client build with real diligence packs? Adds a provider and a quarantine state. | eng | SL3 (client-grade) |

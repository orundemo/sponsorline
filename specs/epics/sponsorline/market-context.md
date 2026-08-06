# sponsorline — Market context

Why this epic exists, what the buyer is actually buying, and what has to be in
the envelope alongside the code.

> Sourced from the shortlist analysis in `~/sourceplane/orun-upwork-top5.md`
> (prepared 3 Aug 2026). Upwork disallows automated fetching, so the figures
> below are as recorded at that time and should be re-checked in the browser
> before a proposal is sent.

## The originating job

**"Senior Full-Stack Developer / Team — Investment Marketplace Platform
(Next.js + Supabase + Sanity)"** — ranked #3 of five, and the **highest
structural fit in the shortlist**.

- **Link:** https://www.upwork.com/jobs/Senior-Full-Stack-Developer-Team-Investment-Marketplace-Platform-Next-span-class-highlight-Supabase-span-Sanity_~022083609979342656667/
- **Budget:** $3,500 fixed
- **Posted:** 1 Aug 2026 · 20–50 proposals · **6 interviewing** · last viewed yesterday
- **Client:** India; payment + phone verified; 5 jobs posted, **0% hire rate**; member since Jul 2023
- **Connects:** 19
- **Also open:** a second job — a **$5,000 travel booking platform** (B2C + B2B agent portal + admin ERP). Same shape again.

**What he says he wants:** four interconnected applications — Public Website,
Investor Portal, Sponsor Portal, Admin Portal — over a shared backend, database,
and auth. Deal rooms with NDA-gated document access, opportunity review and
approval workflow, lead and CRM management, CMS, reports, roles and permissions.
SOW and designs shared post-NDA. He asks for an "estimated fixed-price
quotation" and a "proposed team structure."

**What he actually wants.** The commercial read matters more than the technical
one. A 0% hire rate across 5 posts, two large open jobs, and 6 people
interviewing means he is **collecting quotes** — possibly to price a build he
intends to resell. He is comparing us against agencies on price and headcount,
which is a comparison nobody wins by being cheaper.

So the move is to **collapse the comparison**. Three of his four applications
already exist as one running system; his SOW is mostly configuration, not
construction. That reframes the question from "how many developers × how many
weeks" into "how much of this is already done" — and that question has a URL for
an answer.

## Why this shape wins

| Brief requirement | Status |
|---|---|
| Shared auth across four apps | shipped |
| Sponsors and investors as tenants | shipped |
| Roles and permissions | shipped |
| Approval workflow's audit trail | shipped |
| Admin/support workflows | shipped |
| Email notifications | shipped |
| Console shell, org switching, settings | shipped |
| CMS | **push back** — the console renders catalog-driven markdown; Sanity is a recurring cost he doesn't need. Offer it priced. |
| **Opportunities, deal rooms, NDA gates, document grants, leads** | **this epic** |

**Coverage: ~85%** — the highest in the shortlist, and the reason this demo is a
thin vertical (≈1 day) rather than a deep build.

## The demo

**Sponsorline** → `sponsorline.orun.dev`, with three logins in the proposal
body: sponsor, investor, admin.

Seeded: 6 opportunities, 2 sponsors, 3 investors, one opportunity mid-approval
sitting in the admin queue, and **one deal room with an NDA gate you can
actually trip**.

That last item is the whole pitch in fifteen seconds. Log in as an investor who
has not accepted the NDA, open the deal room, get refused with a reason — then
accept, watch the document appear, and see both events in the sponsor's access
log. No architecture diagram communicates as much.

## Proposal angle

Lead with the collapse:

> *"You listed four applications. Three of them — investor portal, sponsor
> portal, admin portal — are the same multi-tenant system with different role
> bindings, not three builds. Here's that system running: [URL]. Log in as
> investor / sponsor / admin: [3 logins]. What's left for your platform is the
> opportunity lifecycle and the deal room, which is where your SOW actually is.
> That's why my number is lower than the agencies' and my timeline is shorter."*

Then the artefacts, same four as every proposal we send:

1. **The live product** — URL plus seeded logins **in the proposal body**, not
   "happy to share on request." One line on what to try first: *trip the NDA
   gate.*
2. **The workspace** — a read-only Orun Cloud login at **app.orun.dev**: the
   component catalog with typed relations, real deployment history, docs pinned
   to a commit. *"This is the operations console you get on day one, not a status
   page I made for this proposal."*
3. **The onboarding pack** — `docs/{overview,architecture,runbook}.md`,
   generated `ai/context/*`, plus per-client `ONBOARDING.md` and
   `HANDOVER.md`/`EXIT.md`.
4. **The cover letter** — live URL and login in paragraph one, because Upwork
   truncates previews at about two lines.

**Sign the NDA fast.** It filters out half the competition and it is the only
way to see the SOW that the quote depends on.

## Pricing

$3,500 is under-scoped for four portals; do not pretend otherwise — pretending
is how this becomes an unpaid month.

- **Phase 1 — $3,500 fixed:** platform live in his accounts, investor and
  sponsor portals, auth and roles, admin shell.
- **Phase 2 — $6,000–8,000:** deal rooms, NDA gating and grants, approval
  workflow, reports, CMS if he insists on Sanity.

**Quote both his jobs.** The travel platform ($5,000: B2C + B2B agent portal +
admin ERP) is the same topology a third time — a marketplace with two role
families and an admin spine. Offering a two-project rate is how a quote-collector
becomes a client, and it doubles the value of a demo we build once.

## Risk

The 0% hire rate is the real risk, not the technology. Cap exposure by keeping
Phase 1 small and demo-backed, and do not invest proposal effort beyond the demo
already built for the shortlist.

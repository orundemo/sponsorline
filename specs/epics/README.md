# Epics

Status: Normative index

Work programs for sponsorline. Each epic is a folder carrying a canonical doc
set. Durable per-bounded-context behaviour ships as code under `apps/` and
`packages/`; epics are the programs that *introduce or evolve* a context.

## The epics

| Epic | Cluster | Status | Owner(s) | What it is |
|------|---------|--------|----------|------------|
| [`sponsorline/`](./sponsorline/) | **SL** (SL0–SL8) | Draft — not started | new `apps/deals-worker`, `apps/api-edge`, `packages/{contracts,policy-engine,db,sdk,cli}`, `apps/web-console-next` | The product bounded context: an investment marketplace — opportunity lifecycle, grant-scoped cross-org visibility, R2-backed deal rooms behind an NDA gate, and leads — audited on the platform baseline. |

## Lifecycle & conventions

- **Status legend:** `Draft → Ready → In progress → ✅ Shipped → ⛔ Blocked → Closed`.
- **As-built ≠ intent.** What actually shipped lives in each epic's
  `IMPLEMENTATION-STATUS.md`, kept distinct from the design and plan docs.
- **Milestone ✅, not archive.** A completed milestone inside an active epic is
  marked ✅ in `implementation-plan.md` and recorded in
  `IMPLEMENTATION-STATUS.md` — it is not deleted.
- **Doc set per epic:** `README.md` (charter: status, thesis, read order,
  milestones), `design.md` (technical design), `implementation-plan.md`
  (milestones with "done when"), `IMPLEMENTATION-STATUS.md` (as-built), plus
  `risks-and-open-questions.md` and any context docs that carry weight.
- **Layer discipline.** Epics never introduce standalone CI logic, environment
  behaviour hidden in shell scripts, or edits to generated `.orun/**`.
  Components are declared by `component.yaml`; execution contracts live in the
  pinned `stack-tectonic` catalog and are adopted by bumping the tag in
  `intent.yaml`.

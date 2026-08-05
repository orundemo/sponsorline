# web-console-next

Next.js 15 + opennextjs/cloudflare delivery of the SponsorLine web console (per-environment, Workers + Static Assets)

The sponsorline console UI — Next.js compiled to a Cloudflare Worker with
Static Assets, configured against the API edge. Public at
`https://sponsorline-web-console-next-{stage,prod}.orundemo.workers.dev`.

## Depends on

- **api-edge** — Cloudflare Worker for the API edge Runtime

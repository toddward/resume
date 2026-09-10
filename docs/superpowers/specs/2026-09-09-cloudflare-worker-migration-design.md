# Cloudflare Worker Migration — Design

**Date:** 2026-09-09
**Branch:** `updates/q4`
**Status:** Approved in brainstorming; awaiting implementation plan

## Goal

Serve the resume site at `wardzinski.info` and `www.wardzinski.info` from a
Cloudflare Worker with Static Assets, deployed by GitHub Actions on push to
`master`. Retire the homelab Kubernetes deployment and the Pangolin tunnel
path. Full cutover, not a parallel run.

## Current State

- Site is one static page: `app/index.html` (inline CSS), `app/print.css`,
  one image under `app/images/`, Google Fonts loaded from Google's CDN.
- Built into an nginx container by `.github/workflows/build-and-push.yml`,
  pushed to GHCR, deployed to a homelab Kubernetes cluster via
  `charts/resume/`, and exposed through a Pangolin tunnel (`newt` pod).
- Cloudflare zone `wardzinski.info` (id `410d16e3cc13ee121e99146cc7528322`)
  is already on Cloudflare DNS, Free plan.
- DNS today:
  - `wardzinski.info` and `www.wardzinski.info`: proxied placeholder A
    records (`192.0.2.1`). A redirect rule in the
    `http_request_dynamic_redirect` ruleset sends both to
    `https://resume.wardzinski.info` (301).
  - `resume.wardzinski.info`: unproxied CNAME to `pangolin.wardzinski.dev`.
  - `sheryl.wardzinski.info`: unrelated Cloudflare Tunnel CNAME. Untouched.
- Account already has static-assets Workers (`chestergappotter`,
  `thepourplan`) using custom domains. This migration follows that pattern.
- The old `cloudflare-updates` branch is from the previous template site and
  targeted Cloudflare Tunnel, not Workers. It is not a starting point.

## Decisions

| Decision | Choice | Rejected |
|---|---|---|
| Hosting model | Worker with Static Assets, no script (`main` omitted) | Worker + fetch handler (no request logic needed today); Cloudflare Pages (legacy) |
| Deploy trigger | GitHub Actions running `wrangler deploy` on push to `master` | Workers Builds (dashboard-only setup); manual deploy |
| Scope | Full cutover: DNS moves, k8s/container files removed | Keep k8s files as fallback; Worker-only with no DNS change |
| Worker name | `wardzinski-info` (matches domain-named Workers in the account) | `resume` |
| `resume.wardzinski.info` | 301 to `https://wardzinski.info` | Bind as third custom domain |
| www vs apex | Both serve the Worker directly; no canonical redirect | www -> apex redirect |

## Repo Changes

### Added

- `wrangler.jsonc`
  - `name`: `wardzinski-info`
  - `compatibility_date`: `2026-09-09`
  - `assets.directory`: `./app`
  - `assets.not_found_handling`: `none` (plain 404 for unknown paths)
  - `observability.enabled`: `true`
  - `routes`: two entries with `custom_domain: true` for
    `wardzinski.info` and `www.wardzinski.info`
  - No `main`. No bindings.
- `.github/workflows/deploy.yml`
  - Triggers: `push` to `master`, `workflow_dispatch`, `pull_request` to
    `master`.
  - Steps: checkout, setup-node 22, `npm ci`, then
    `cloudflare/wrangler-action` with `command: deploy` on push/dispatch and
    `command: deploy --dry-run` on pull requests.
  - Secrets: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`.
- `package.json` + `package-lock.json`
  - `wrangler` pinned as a devDependency.
  - Scripts: `dev` -> `wrangler dev`, `deploy` -> `wrangler deploy`.
- `.gitignore`: `node_modules/`, `.wrangler/`, `.DS_Store`,
  `.playwright-mcp/`.
- `Readme.md` rewritten: local preview, deploy flow, DNS/custom-domain
  notes, one-time secret setup.

### Removed

- `Dockerfile`
- `kube_deployments.yml`
- `wardzinskiinfo_ingress.yml`
- `issuer.yml`
- `charts/` (entire directory, including the pangolin template that
  contains a committed newt secret)
- `.github/workflows/build-and-push.yml`

### Untouched

- Everything under `app/`.
- `works/` and `ToddLinkedInResume.pdf` (never served; source artifacts).

## Cutover Runbook

Ordered so the old site serves until the last step and any failure leaves
DNS unchanged.

### Prerequisites (user)

1. Create a Cloudflare API token from the "Edit Cloudflare Workers"
   template, scoped to zone `wardzinski.info`, plus Zone > DNS > Edit.
2. Add it to the GitHub repo: `gh secret set CLOUDFLARE_API_TOKEN`.
3. `CLOUDFLARE_ACCOUNT_ID` is set by Claude (`8b91b6ce3fda30ddf8133756f18b590e`).

### Steps

1. Merge the PR to `master`. CI runs `wrangler deploy`, which uploads the
   assets and creates custom domains for apex and www, replacing the
   placeholder A records. The existing redirect rule still fires first, so
   apex/www continue to 301 to `resume.wardzinski.info` (homelab).
   - If wrangler refuses to overwrite existing DNS records from CI, create
     the custom domains via the Workers Domains API with
     `override_existing_dns_record: true` and re-run the workflow.
2. Edit the existing redirect rule: expression becomes
   `http.host eq "resume.wardzinski.info"`, target becomes
   `https://wardzinski.info`, status stays 301, preserve query string.
   Apex and www now serve the Worker.
3. Replace the `resume.wardzinski.info` CNAME with a proxied A record to
   `192.0.2.1` so the redirect rule can act on it.
4. Verify (see Testing).
5. User uninstalls the Helm release and deletes the Pangolin resource for
   the resume site. This also retires the committed newt secret. Do this
   only after step 4 passes.

Each Cloudflare-side change in steps 1-3 is confirmed with the user before
it is made.

### Rollback

- Restore the `resume.wardzinski.info` CNAME to `pangolin.wardzinski.dev`
  (unproxied).
- Restore the redirect rule expression to match apex/www and target
  `https://resume.wardzinski.info`.
- Homelab pods remain running until step 5, so rollback needs no redeploy.

## Testing

### Before merge

- `wrangler deploy --dry-run` succeeds (also runs in CI on the PR).
- `wrangler dev` locally, then curl:
  - `/` -> 200, body contains `W.T. Wardzinski — Resume`
  - `/print.css` -> 200, `content-type: text/css`
  - `/images/claude-certified-associate-foundations.png` -> 200,
    `content-type: image/png`
  - `/does-not-exist` -> 404

### After cutover

- `curl -sI https://wardzinski.info/` -> 200, `cf-ray` header present
- `curl -sI https://www.wardzinski.info/` -> 200
- `curl -sI https://resume.wardzinski.info/` -> 301,
  `location: https://wardzinski.info/`
- `curl -s https://wardzinski.info/ | grep -c 'Wardzinski'` -> non-zero

## Out of Scope

- Cache or security headers (`app/_headers`). Add later if wanted.
- Preview URLs per pull request.
- Any change to the site content or print stylesheet.
- Cleanup of `works/`, the PDF, or the stale `cloudflare-updates` branch.

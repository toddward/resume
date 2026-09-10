# wardzinski.info

Resume site for Wm. Todd Wardzinski. One static page served from a
Cloudflare Worker with Static Assets. No build step, no server code.

## Layout

- `app/` — everything that is served: `index.html`, `print.css`, `images/`.
- `wrangler.jsonc` — Worker config: asset directory and custom domains.
- `.github/workflows/deploy.yml` — deploys on push to `master`.

## Local preview

```bash
npm ci
npm run dev
```

Open http://localhost:8787.

## Deploy

Merging to `master` deploys automatically. Pull requests run
`wrangler deploy --dry-run` to validate the config without publishing.

To deploy from your own machine instead:

```bash
npx wrangler login
npm run deploy
```

## One-time setup

GitHub Actions needs two repository secrets:

- `CLOUDFLARE_API_TOKEN` — created from the "Edit Cloudflare Workers"
  template, scoped to the `wardzinski.info` zone, plus Zone > DNS > Edit.
- `CLOUDFLARE_ACCOUNT_ID` — the Cloudflare account id.

Set each with `gh secret set <NAME>`.

## DNS

The `wardzinski.info` zone is on Cloudflare. `wardzinski.info` and
`www.wardzinski.info` are Worker custom domains declared in
`wrangler.jsonc`; each deploy keeps them attached.
`resume.wardzinski.info` is a Cloudflare redirect rule to
`https://wardzinski.info`.

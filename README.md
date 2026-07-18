# EFI Maintenance Page

Static "we'll be back" page shown to users while the EFI prod platform is down for
scheduled maintenance. Live at **https://intheshop.ethicalcharterprogram.org/index.html**.

## How it's set up

| Layer | Config | Where |
|---|---|---|
| Page content | `index.html` in this repo, `main` branch | this repo |
| Hosting | GitHub Pages — `main` branch, root, legacy build, HTTPS enforced | repo Settings → Pages |
| Custom domain | `CNAME` file in repo root (`intheshop.ethicalcharterprogram.org`) — created automatically by Pages, do not delete | this repo |
| DNS | `CNAME intheshop → efi-lab-platform.github.io` | HostGator cPanel, zone `ethicalcharterprogram.org` (NOT Namecheap — registrar only) |
| TLS | Let's Encrypt, issued and auto-renewed by GitHub Pages | automatic |
| Heroku wiring | `MAINTENANCE_PAGE_URL=https://intheshop.ethicalcharterprogram.org/index.html` config var on all 3 prod apps | `prod-app-lab-ecip-api`, `-saq`, `-cp` |

The fallback URL `https://efi-lab-platform.github.io/maintenance-page/index.html` now
301-redirects to the custom domain.

## How it's triggered

Heroku maintenance mode serves `MAINTENANCE_PAGE_URL` instead of the app:

```bash
# ON (start of downtime window) — run for each affected app
heroku maintenance:on -a prod-app-lab-ecip-api
heroku maintenance:on -a prod-app-lab-ecip-saq
heroku maintenance:on -a prod-app-lab-ecip-cp

# verify: requests log as H80 in the router
heroku logs -p router -n 5 -a prod-app-lab-ecip-saq

# OFF (end of window)
heroku maintenance:off -a prod-app-lab-ecip-api   # etc. for -saq, -cp
```

Maintenance mode affects the Heroku-hosted apps only; the public marketing sites
(equitablefood.org, ethicalcharterprogram.org apex) are unaffected.

## How to edit the page

```bash
cd maintenance-page
git pull            # ALWAYS — GitHub commits to this repo itself (e.g. the CNAME file)
# edit index.html (downtime dates/times live in the <ul> list)
git add index.html && git commit -m "Update downtime window" && git push
```

Pages rebuilds automatically on push (~40 s). No Heroku change needed — the URL
stays the same. Verify: `curl -s https://intheshop.ethicalcharterprogram.org/ | grep -i maintenance`.

## Gotchas

- **Order matters when (re)pointing domains**: create the DNS CNAME first, THEN set
  the Pages custom domain. Setting the custom domain first makes the github.io URL
  redirect to a dead hostname.
- Deleting the `CNAME` file from the repo removes the custom domain.
- To change the subdomain: add new DNS CNAME in HostGator, then
  `gh api -X PUT repos/EFI-LAB-Platform/maintenance-page/pages -f cname=<new-host>`,
  wait for cert (`--jq '.https_certificate.state'` until `approved`), then
  `gh api -X PUT ... -F https_enforced=true`, then update `MAINTENANCE_PAGE_URL` on
  all 3 apps.
- Template + populated copy for future windows:
  `efi-30-changeover-resources/inbox/.processed/Maintenance-Downtime-Steps-Template.md`.

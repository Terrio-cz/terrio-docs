# terrio-docs

Build and hosting of the Terrio developer docs and changelog at **https://docs.terrio.cz** (TER-672).

The content is not here: it lives in `docs-site/` of [`Terrio-cz/importer`](https://github.com/Terrio-cz/importer), next to
the code and the OpenAPI contract it documents, and the changelog is generated from the importer's history and release
tags (importer `docs/README.md`). This repository pins which importer commit is live (`IMPORTER_SHA`, plus the release tags
it contains in `IMPORTER_TAGS`), builds it and
deploys it to the Terrio OVH VPS (`<vps-host>`), which also runs the importer's CI runner and `terrio.cz` (TER-671,
`deploy/README.md` of `Terrio-cz/terrio-frontend-pre`: firewall, Cloudflare, `ci.slice`).

## How a release goes live

1. A push to the importer's `master` or a `release/*` tag runs its `.github/workflows/docs-trigger.yml`. It compares master
   with the pin here, not with the pushed range, so a cancelled run loses nothing: when an input of the site changed since
   the pinned commit (`docs-site/`, `docs/openapi/`, `docs/api-changes/`, `design-tokens/`, the oasdiff pin or severity
   list, a `Public-Change*` trailer) or the release tags differ, it commits the master sha and tags here (`main`) with the
   `DOCS_REPO_KEY` deploy key (write access to this repository only). The pin never moves back; most merges change
   nothing in the site and deploy nothing. A manual run of the trigger pins master unconditionally.
2. That push runs `.github/workflows/deploy.yml` (also by hand, `main` only) on a GitHub-hosted runner: checkout of the
   importer at exactly that sha with full history (`IMPORTER_READ_KEY`, a read-only deploy key of the importer), a check
   that the sha is on importer `master` (whoever pushed the pin), `npm ci`,
   `npm run changelog:collect`, `npm run check` (the same build and checks as the importer's `docs-site.yml`), then
   `dist/version.txt` = this repository's commit (the release name: a new tag can change the site without a new importer
   commit), `dist/importer.txt` = the importer sha, `tar czf` of `dist/` piped to `docs-deploy@<vps-host>` with that
   commit as the only argument.
3. The VPS key (`VPS_DEPLOY_KEY` secret, host keys in the `VPS_KNOWN_HOSTS` variable) is `restrict`ed to
   `terrio-docs-deploy-entry`, which accepts only a 40-hex sha and calls `sudo terrio-docs-deploy` (the only sudo rule of
   the `docs-deploy` user, `/etc/sudoers.d/terrio-docs-deploy`). The `docs-deploy` user is separate from the landing page's
   `deploy` user: neither key can replace the other site.
4. `terrio-docs-deploy` refuses an archive over 200 MB or unpacking to more than 1 GiB or 50,000 entries, links, devices,
   `..`, anything outside `dist/`, missing entry pages and a `version.txt` other than the sha; unpacks into
   `/srv/terrio-docs/releases/<sha>`, switches the `current` symlink and checks that Caddy serves the new `version.txt`,
   `/docs/` and `/en/docs/`; otherwise it switches back to the previous release and drops the new one. A re-run of the live
   sha is a no-op, a lock keeps deploys and rollbacks apart, the five newest releases and the live one are kept.
5. The job ends by reading `https://docs.terrio.cz/version.txt` through Cloudflare.

A red build deploys nothing; the live release stays.

## Files

| File | Installed as |
|---|---|
| `deploy/docs.caddy` | `/etc/caddy/sites/docs.caddy`, imported by the last line of `/etc/caddy/Caddyfile` (`import /etc/caddy/sites/*.caddy`) |
| `deploy/terrio-docs-deploy` | `/usr/local/sbin/terrio-docs-deploy` (root) |
| `deploy/terrio-docs-deploy-entry` | `/usr/local/bin/terrio-docs-deploy-entry` (forced command of the VPS key) |

Change them here first, then install them:

```bash
scp deploy/{docs.caddy,terrio-docs-deploy,terrio-docs-deploy-entry} ubuntu@<vps-host>:/tmp/
ssh ubuntu@<vps-host> 'sudo install -m 644 /tmp/docs.caddy /etc/caddy/sites/ && sudo install -m 755 /tmp/terrio-docs-deploy /usr/local/sbin/ && sudo install -m 755 /tmp/terrio-docs-deploy-entry /usr/local/bin/ && sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile && sudo systemctl reload caddy'
```

## How a request is served

- Cloudflare (proxied `A` record `docs` → `<vps-host>`, SSL/TLS Full (strict)) → Caddy on 443, Let's Encrypt certificate.
- Caddy serves `/srv/terrio-docs/current/dist` by the importer's deploy contract (`docs/README.md`, reference
  `docs-site/deploy/nginx.conf`): section paths without a trailing slash 301 to a relative `Location`, unknown paths 404
  with the Czech or (below `/en/`) English 404 page, existing `/_astro/*` files cached for a year (`immutable`; a 404 there
  is not cached), `/version.txt` `no-store`, gzip/zstd, `nosniff`, `Referrer-Policy`, `X-Frame-Options: DENY`,
  `Permissions-Policy` on every response, 404 pages included.
- `/` and `/en/` 302 to `https://terrio.cz/`: the site title and the shared navigation (`/#data`, `/#cenik`, ...) point
  at the website root, which the landing page owns; the landing page has no English version yet.
- `http://127.0.0.1:8791` serves the same site on loopback only, so the deploy check needs neither DNS nor the certificate.

## Rollback

Revert the pin commit on `main` (the deploy builds the older importer commit; the trigger moves the pin forward again only
when the site changes after it), or at once on the VPS:

```bash
ssh ubuntu@<vps-host> 'ls -1t /srv/terrio-docs/releases'
ssh ubuntu@<vps-host> 'sudo terrio-docs-deploy --rollback <sha>'
```

The next pin deploys again.

## Keys

All three are ed25519 keys without a local copy; replace one by generating a new pair:

| Secret | Repository | Public half |
|---|---|---|
| `VPS_DEPLOY_KEY` | this one | `/home/docs-deploy/.ssh/authorized_keys` on the VPS, as `restrict,command="/usr/local/bin/terrio-docs-deploy-entry" <key>` |
| `IMPORTER_READ_KEY` | this one | read-only deploy key of `Terrio-cz/importer` |
| `DOCS_REPO_KEY` | `Terrio-cz/importer` | deploy key with write access of this repository |

Trust boundary: write access to the importer can push here, and write access here can read the importer and replace the
docs. Keep both to the same trusted team.

## Checks

```bash
ssh ubuntu@<vps-host> 'readlink /srv/terrio-docs/current; curl -s http://127.0.0.1:8791/version.txt'
curl -s https://docs.terrio.cz/version.txt https://docs.terrio.cz/importer.txt
```

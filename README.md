# deploy

VPS stack: Caddy edge + basic-service + the proxy inbounds. The blog is a Vercel-only site.

## Hostnames

| Env var | Value | Served by |
| --- | --- | --- |
| `API_HOSTNAME` | `api.wwr.icu` (Cloudflare proxied) | Caddy: `/api/open/*` and `/api/admin/*` only |
| `DIRECT_HOST` | `btp.wwr.icu` (Cloudflare DNS-only, must point at the VPS) | Caddy: `/gp`, `/bp`, `/btp` proxies, everything else -> `https://admin.wwr.icu` |
| `RESOURCE_HOSTNAME` | `res.wwr.icu` (CNAME to `api.wwr.icu`) | Caddy: `/image/*` only |
| `SSH_HOST` | the VPS address, used only as the CI ssh/scp target | not a site |

Off this box: `wwr.icu`, `dev.wwr.icu` and `preview.wwr.icu` are the Vercel `blog-page` project,
`devtoys.wwr.icu` is the Vercel `devtoys` project, and `admin.wwr.icu` is the Vercel `admin-page`
project - it replaces `manage.wwr.icu`, which is retired. The SPA then bridges its own `/api/*` to
`https://api.wwr.icu/api/admin/*`; nothing else of the admin app is served from the VPS, which only
fetches that site as the `DIRECT_HOST` front.

The `/gp`, `/bp` and `/btp` inbounds used to live on `admin.wwr.icu` (in the Caddyfile bundled with the
`admin-page` image). Once that name is a CNAME to Vercel it can no longer reach this box, so client configs
naming `admin.wwr.icu` have to be re-imported against `btp.wwr.icu` - and `btp.wwr.icu` has to exist as a
DNS-only A record before Caddy can get a certificate for it.

## Split between open and privileged API

basic-service answers on three path groups:

- `/open/*` - what the blog renders. Public.
- `/image/*` - image redirects. Public, but only exposed on `RESOURCE_HOSTNAME`.
- `/admin/*` - login, manage, post, tag. Reachable from outside through Caddy's `/api/admin/*` route on `API_HOSTNAME` - no other path on that host is proxied to the service at all - and, since `DIRECT_HOST`'s catch-all fronts `admin.wwr.icu`, through that SPA's own `/api/*` bridge on the second name. Both are public URLs; inside the group the session cookie (`is_admin`) decides. There is no shared secret between Caddy and the service any more: a container on the compose network that dials `basic-service:8000` directly reaches these handlers, which is why `expose:` (never `ports:`) is what keeps that port off the host.

## Manual Caddy deployment

Caddy is no longer baked into the admin-page image; the file lives in [`caddy/Caddyfile`](caddy/Caddyfile) and runs from the official `caddy:2-alpine` image with a read-only bind mount.

```bash
cd /data/deploy

docker compose up -d caddy                                   # first start / recreate
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile   # check before applying
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile     # apply edits
docker compose logs -f caddy
```

The mount is a single-file bind mount: edit `/data/deploy/caddy/Caddyfile` in place (`vim`), do not replace it with `mv`/`scp` of a new file, or the running container keeps seeing the old inode and you need `docker compose up -d --force-recreate caddy` instead.

Certificates and ACME state live in the `/data/caddy/data` volume, so recreating the container does not reissue them.

Access logs are one file per site - `/data/caddy/log` maps to `/var/log`, so they land on the host as
`/data/caddy/log/caddy-api.log`, `caddy-direct.log` and `caddy-resource.log` (10 MiB x 10 each). Separate files
matter because each site block owns its own logger, and two logger instances rotating the same file step on
each other's `roll_keep`.

## The admin app behind `DIRECT_HOST`

The catch-all route in that site block fetches the Vercel `admin-page` site (`https://admin.wwr.icu`, hardcoded
in [`caddy/Caddyfile`](caddy/Caddyfile) - the admin app needs no secrets at the edge, so it is not a `.env` key),
which gives the login UI a second name that resolves straight to this box instead of only through Cloudflare.
Nothing is deployed here: no container, one source of truth on Vercel.

The SPA keeps working behind that name because it calls relative `/api/*` and the login cookie is scoped to
`.wwr.icu`, so `btp.wwr.icu` is same-site for `api.wwr.icu`.

This is a front, not a second copy of the app: that route answers with `X-Robots-Tag: noindex, nofollow`, and it
never touches `/gp`, `/bp` or `/btp`. The trade is the cover - the proxy domain used to read as an ordinary blog in
a browser and now reads as an admin panel.

## CI

`workflow_dispatch` only, all of them run `clean_deploy.yml` first (it rewrites `/data/deploy`, including `.env` and the Caddyfile, from the repo plus the repository secrets):

- `deploy_service.yml` - basic-service release
- `deploy_proxy.yml` - bad-proxy-core release
- `deploy_all.yml` - everything at once
- none of them touch `caddy` except `deploy_all`, which recreates it as part of `up -d`

Required repository secrets: `SSH_HOST`, `API_HOSTNAME`, `DIRECT_HOST`, `RESOURCE_HOSTNAME`, `USERNAME`, `PRIVATE_KEY`, `V2RAY_UUID`, `CF_ACCOUNT_ID`, `CF_KV_NAMESPACE_ID`, `CF_KV_API_TOKEN`.

# deploy

VPS stack: Caddy edge + basic-service + the proxy inbounds. The blog and the admin SPA both live on Vercel.

## Hostnames

| Env var | Default use | Served by |
| --- | --- | --- |
| `API_HOSTNAME` | `api.wwr.icu` | Caddy: `/api/open/*` and `/api/admin/*` only |
| `DIRECT_HOST` | the direct-connect domain (today `btp.wwr.icu`) | Caddy: `/gp`, `/bp`, `/btp` proxies, everything else → the Vercel blog |
| `RESOURCE_HOSTNAME` | `res.wwr.icu` | Caddy: `/image/*` only |
| `SSH_HOST` | the VPS address used by the CI runners | not a site, just the deploy target |

`admin.wwr.icu` is gone: the admin SPA is hosted on Vercel and calls `https://api.wwr.icu/api/admin/*`. Note that
the `/gp`, `/bp` and `/btp` inbounds used to be served on that hostname (they lived in the Caddyfile bundled with
the `admin-page` image), so client configs still naming `admin.wwr.icu` must be re-pointed at `DIRECT_HOST` - or
keep `admin.wwr.icu`'s DNS on this VPS until they are.

## Split between open and privileged API

basic-service answers on three path groups:

- `/open/*` - what the blog renders. Public.
- `/image/*` - image redirects. Public, but only exposed on `RESOURCE_HOSTNAME`.
- `/admin/*` - login, manage, post, tag. Requires the `X-Internal-Token` header to equal `ADMIN_API_TOKEN`, and the only thing that adds that header is Caddy's `/api/admin/*` route (`header_up` replaces whatever the client sent). A caller that reaches the service without going through that route - anything on the compose network, or a blog visitor trying to use blog-page's `/api/*` as a bridge - gets 404 without ever hitting a handler.

`ADMIN_API_TOKEN` must therefore be set in `.env`: both the `caddy` and `basic-service` services declare it with `:?`, so `docker compose up` fails loudly instead of silently starting an unguarded or unreachable admin surface. Anything that bypasses the edge - a container on the compose network talking to `basic-service:8000` directly - is on that second door, not on the public path.

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

## The blog behind `DIRECT_HOST`

The catch-all route in that site block fetches the Vercel site (`https://wwr.icu`, overridable with
`BLOG_UPSTREAM` in `.env`), so the proxy domain reads as an ordinary blog in a browser. The blog is not deployed
here: no container, no canonical-link env, one source of truth on Vercel.

This is a front, not a second copy of the site: that route answers with `X-Robots-Tag: noindex, nofollow` to keep
cover pages out of search results, and it never touches `/gp`, `/bp` or `/btp`.

## CI

`workflow_dispatch` only, all of them run `clean_deploy.yml` first (it rewrites `/data/deploy`, including `.env` and the Caddyfile, from the repo plus the repository secrets):

- `deploy_service.yml` / `dev_service.yml` - basic-service release / snapshot
- `deploy_proxy.yml` / `dev_proxy.yml` - bad-proxy-core release / snapshot
- `deploy_all.yml` / `dev_all.yml` - everything at once
- none of them touch `caddy` except `deploy_all`/`dev_all`, which recreate it as part of `up -d`

Required repository secrets: `SSH_HOST`, `API_HOSTNAME`, `DIRECT_HOST`, `RESOURCE_HOSTNAME`, `ADMIN_API_TOKEN`, `USERNAME`, `PRIVATE_KEY`, `V2RAY_UUID`, `CF_ACCOUNT_ID`, `CF_KV_NAMESPACE_ID`, `CF_KV_API_TOKEN`.

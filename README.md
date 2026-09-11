# caddy-stack

Portainer Git stack for [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy)
on `apple-pi.lan`. Gives every container a hostname instead of a port number.

- **Stack name in Portainer:** `caddy`
- **Owns:** host port `80`
- **Routes by:** `Host` header, from labels read off the Docker socket

## How it works

Caddy watches the Docker socket and rebuilds its own config whenever a container
starts or stops. There is no Caddyfile in this repo and nothing to edit here when
you add a service — the route lives on the service, in its own stack's compose.

DNS is a single wildcard on Tokaido, so new hostnames need no router change:

```
*.svc.lan  ->  192.168.1.12
```

## Adding a service

Two things in the service's own compose: join the `caddy` network, and label it.

```yaml
services:
  grafana:
    networks:
      - loki
      - caddy
    labels:
      - "caddy=http://grafana.svc.lan"
      - "caddy.reverse_proxy={{upstreams 3000}}"

networks:
  caddy:
    external: true
    name: caddy
```

Push, let Portainer poll, and `http://grafana.svc.lan` works. Remove the labels
and the route disappears. `{{upstreams 3000}}` is the *container* port, not the
published one — services behind Caddy do not need to publish a port at all.

### Always write `http://`

A bare `caddy=grafana.svc.lan` makes Caddy try to provision a real certificate,
which cannot work for `.lan` and leaves the route broken. The explicit `http://`
scheme keeps it plaintext. Everything here is LAN-only or already behind
Cloudflare's edge, so nothing is served over plaintext that was not already.

## The network is external on purpose

`caddy` is declared `external: true` with an explicit `name:`, so it is never
prefixed with the compose project name. Proxied services in other stacks
reference it by that literal name. If this stack created the network itself it
would land as `caddy_caddy`, and every other stack's `external: name: caddy`
would fail to start.

It is created once, out of band:

```bash
docker network create caddy
```

## Bootstrap

Order matters — Caddy cannot bind `:80` until the website releases it.

```bash
docker network create caddy
# 1. push the website compose change (drops ports:, adds labels)
# 2. deploy this stack
```

Expect the public site to be unreachable for the ~1-2 minutes between those two
steps.

## builtbybrendan.com goes through here now

`cloudflared` has always sent `builtbybrendan.com` to `http://localhost:80`. That
ingress rule is unchanged — the difference is that Caddy now answers on :80 and
forwards to the website container by `Host` header. The tunnel config needs no
edit.

This means **Caddy is in the path of the public site.** If Caddy is down,
`builtbybrendan.com` is down. It is also the reason the website sets
`TRUST_PROXY=true`: client IPs now arrive as `X-Forwarded-For` from Caddy rather
than from cloudflared directly.

## Not covered by backups

Nothing here needs backing up. Caddy's state is derived from container labels on
every start, and there are no certificates. Rebuilding means redeploying the
stack.

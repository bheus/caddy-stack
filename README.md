# caddy-stack

Portainer Git stack for [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy)
on `apple-pi.lan`. Gives every container a hostname instead of a port number.

- **Stack name in Portainer:** `caddy`
- **Owns:** host ports `80` and `443`
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

The one exception is `invoiceninja.svc.lan`, which uses `tls=internal` — see
*HTTPS* below.

## Caddy does not hold the Docker socket

Caddy reads container labels through `tecnativa/docker-socket-proxy`, not from a
socket bind mount of its own.

This matters because `-v /var/run/docker.sock:...:ro` is not the protection it
looks like. The `:ro` applies to the socket *file*, not the API behind it — a
process with that mount can create containers, mount the host filesystem, and
read every other container's environment variables. Caddy is reachable from the
internet through the Cloudflare tunnel, so it is the last thing that should hold
an unrestricted Docker handle.

The proxy allows only the GET endpoints Caddy actually uses (`CONTAINERS`,
`NETWORKS`, `EVENTS`, `PING`, `VERSION`, `INFO`) and sets `POST=0`, so container
creation returns 403.

`NETWORKS=1` is required even though `CADDY_INGRESS_NETWORKS` is set explicitly;
without it Caddy logs `Failed to get ingress networks` and guesses upstream
addresses instead.

The proxy sits on a separate `dockerapi` network marked `internal`, **not** on
the shared `caddy` network — otherwise every proxied service (grafana,
documenso, …) could reach the Docker API too.

## Deploys are webhook-only

This stack has **no polling interval**. `.github/workflows/deploy.yml` validates
the compose against an empty environment, then POSTs the Portainer stack webhook,
so a push is live in seconds rather than after a poll. That webhook is the only
deploy path — if the workflow does not run, nothing reaches the Pi.

The URL lives in the repo secret `PORTAINER_CADDY_WEBHOOK` and must use the
public hostname:

```
https://deploy.builtbybrendan.com/api/stacks/webhooks/<uuid>
```

**Not** `portainer.svc.lan`, which Portainer's UI suggests — GitHub Actions runs
on the public internet and cannot resolve a LAN name. That `deploy.` hostname is
path-scoped in the tunnel to `^/api/stacks/webhooks/.*$`, so it reaches the
webhook and nothing else of Portainer.

Portainer returns **204** and does nothing when the repo is already in sync, so a
204 alone does not prove a deploy happened. Check the container's `StartedAt`.

The UI shows a candidate webhook URL before the stack is saved, and that UUID is
**not** registered until you save. POSTing it returns 404 from Portainer itself —
which looks exactly like a tunnel problem and is not one.

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

## HTTPS: `invoiceninja.svc.lan` only

Invoice Ninja sets `REQUIRE_HTTPS=true`, so it cannot be served over plain HTTP
behind a proxy. Its label omits the `http://` scheme and sets `tls=internal`:
Caddy signs the certificate with its own local CA instead of attempting ACME.

That CA is not publicly trusted. Each client that uses the route must trust its
root once. Export it:

```bash
ssh bheussler@apple-pi.lan 'docker exec caddy cat /data/caddy/pki/authorities/local/root.crt' > caddy-root.crt
```

- **macOS:** `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain caddy-root.crt`
- **iOS:** AirDrop the `.crt`, install it under Settings → Profile Downloaded,
  then enable it under Settings → General → About → Certificate Trust Settings.
  Installing the profile alone does not trust it.

The root is valid for 10 years; intermediates and leaves rotate automatically
without touching clients.

## `caddy-data` holds the CA root

The volume holds the CA root key, so it is declared `external` to survive a stack
rename. If it is ever lost, Caddy creates a new root and every client has to
trust the new one. Everything else here is derived from labels on each start.

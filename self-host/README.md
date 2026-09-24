# Self-hosting `@gibs/quote-service`

`@gibs/quote-service` is a free, self-hostable backend for the quote
aggregation the gibs.finance interface uses — it asks LI.FI, Relay and NEAR
Intents for you, shares one upstream call between visitors asking the same
question, and never requires a provider key to start. The interface also
works with no backend at all, calling each provider directly from the
browser; running this service just adds shared caching and one place to
hold your own provider keys. Point your own frontend at your own instance
instead of gibs's, or run one for any other reason.

## Image availability

**Not yet runnable.** As of 2026-09-23, no image has been published to
`ghcr.io/gibsfinance/quote-service` — the `docker compose up` command below
will fail with a pull error until one exists. The source project
(`gibsfinance/bridge`, private) does not yet build or push a container
image anywhere public; this guide documents the target flow so it is ready
the day that changes. Check the
[gibsfinance organization](https://github.com/gibsfinance) or this
repository's own release history for an update, or ask the maintainers
directly.

Everything below this notice describes how self-hosting will work once an
image is published — keep reading if you want to understand or prepare for
it.

## Quick start

```sh
mkdir gibs-quote-service && cd gibs-quote-service
curl -fsSLO https://raw.githubusercontent.com/gibsfinance/interface-releases/main/self-host/docker-compose.yml
curl -fsSLO https://raw.githubusercontent.com/gibsfinance/interface-releases/main/self-host/.env.example
cp .env.example .env      # optional — every setting has a working default
docker compose up
```

That is the whole default setup. It:

- Pulls `ghcr.io/gibsfinance/quote-service` and starts ONE container,
  `quote-service`, published to `127.0.0.1:8080` — reachable only from
  this machine.
- Answers `GET http://127.0.0.1:8080/healthz` (process up) and
  `GET http://127.0.0.1:8080/readyz` (at least one provider's chain list
  has loaded — takes a few seconds after start) immediately, and
  `POST /v1/display-quotes` once you set `ALLOWED_ORIGINS` in `.env` to
  match your own frontend's origin.
- Runs with NO provider key configured — LI.FI and Relay both work, at
  their own public/anonymous rate limit. The container logs one startup
  warning naming whichever key is unset; it is not an error.

Check it is alive:

```sh
curl http://127.0.0.1:8080/healthz
curl http://127.0.0.1:8080/v1/info   # {"service":"quote-service","contract":1,"providers":[...],"version":"..."}
```

`/v1/info` follows the same CORS rule as every other route — from `curl`
(no `Origin` header) it answers 403. That is correct: set `Origin` (or add
your frontend's real origin to `ALLOWED_ORIGINS`) to see it succeed, for
example `curl -H 'Origin: http://localhost:5173' http://127.0.0.1:8080/v1/info`.

## Configuration

Every environment variable this service reads is documented in
[`.env.example`](./.env.example) — copy it to `.env` and edit it. The three
things you almost certainly want to set for anything beyond local testing:

1. **`ALLOWED_ORIGINS`** — the origin(s) your frontend runs on. Without
   this, the built-in default is gibs's own hostnames, which will not
   match yours, and every request gets refused with 403.
2. **`LIFI_API_KEY`** / **`RELAY_API_KEY`** — optional, but each provider's
   anonymous rate limit is shared across every visitor asking through your
   instance at once. A key of your own raises it substantially.
3. **`CLIENT_IP_SOURCE`** — leave this unset (`socket`, the default) unless
   you are running your own trusted reverse proxy directly in front of
   this container on the same machine. See `.env.example`'s own note; the
   wrong setting here can let a client bypass rate limiting entirely.

## Expose it

The default port publish, `127.0.0.1:${QUOTE_SERVICE_PORT:-8080}:8080`, only
accepts connections from this machine. To let anything else reach it —
another machine on your network, or the public internet — you have two
reasonable options, in order of preference:

1. **Put a reverse proxy in front of it** (Caddy, nginx, Traefik — whatever
   you already run) and leave the container's own publish on `127.0.0.1`.
   This is the shape gibs's own production deploy uses. If your proxy sets
   a real client-IP header (Caddy's `{client_ip}`, nginx's `$remote_addr`),
   set `CLIENT_IP_SOURCE=proxy-header` in `.env` so rate limiting keys on
   the real client rather than your proxy's own address.
2. **Publish the container's port directly**, if you understand and accept
   what that means (no TLS, no proxy-level protections, rate limiting keyed
   on raw socket addresses — which is exactly right with no proxy in
   front). Edit `docker-compose.yml`'s `quote-service.ports` entry to
   `"${QUOTE_SERVICE_PORT:-8080}:8080"` (drop the `127.0.0.1:` prefix), or
   set a specific bind address you control.

Either way, `CLIENT_IP_SOURCE` should match your actual topology: `socket`
(default) when nothing in front of this container can be trusted to set
`X-Client-IP` honestly; `proxy-header` only when you have deliberately set
up option 1 above.

## What this does not include

This compose file starts only the quote service. It does NOT give you
transfer history or a GraphQL API for tracking bridge transactions — that
is a separate, heavier service the gibs.finance production deploy runs at
`https://indexer.gibs.finance`. Running your own instance of that indexer
requires the full `gibsfinance/bridge` monorepo source, which is not
public, so it is out of scope for this repository. Point your frontend at
the public indexer above; the quote service works independently of it
either way.

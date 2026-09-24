---
name: integrate-gibs-quotes
description: >
  Add cross-chain bridge quotes (LI.FI, Relay, NEAR Intents) to an
  application using the published `@gibs/quotes` package and its optional
  free backend, `@gibs/quote-service`. Use this whenever a user asks to
  "add bridge quotes", "integrate gibs quotes", "show a cross-chain quote",
  "compare bridge routes", "self-host a quote backend", or mentions
  `@gibs/quotes`, `quote-service`, `QuoteClientConfig`, `createQuoteClient`,
  or the `/v1/display-quotes` endpoint. Also use when debugging a quote
  integration that returns no results, a Cross-Origin Resource Sharing
  error, or an unexpectedly scaled amount.
---

# Integrate `@gibs/quotes`

This skill sets up a working cross-chain quote integration end to end:
picking a mode, installing the package, writing the configuration, and
running the free backend if you need one.

## Step 1 — choose a mode

Three modes exist. Pick one before writing any code.

| Mode | When to use it | Where provider keys live |
|---|---|---|
| **Backend** | You run a server and want the best rate limits and full control. | On your server, never sent to a browser. |
| **Browser, direct** | You want the fastest path to a working demo, with no backend. | None, or a public referral identifier only. |
| **Your own service, with fallback** | You want a shared cache and higher rate limits for many visitors, but do not want a single point of failure. | On your server; the browser falls back to direct calls if the server is unreachable. |

If you are unsure, start with **browser, direct**. It needs no backend
and no key. Move to **your own service** later if you get enough traffic
that a shared cache matters.

## Step 2 — install

```sh
npm install @gibs/quotes
```

The package ships its own `llms.txt` inside the installed package
(`node_modules/@gibs/quotes/llms.txt`). Read that file for the exact
`QuoteClientConfig` shape, the three mode code samples, the wire
contract, and the rules that never bend. This skill tells you how to run
the pieces; `llms.txt` tells you the exact application programming
interface.

## Step 3 — configuration per mode

### Backend

```ts
import { createQuoteClient } from '@gibs/quotes'

const client = createQuoteClient(
  {
    providers: {
      lifi: { apiKey: process.env.LIFI_API_KEY },
      relay: { apiKey: process.env.RELAY_API_KEY },
    },
    fallback: 'none',
  },
  { carriers: [/* built-in carrier implementations you wire up */] },
)
```

### Browser, direct

```ts
const client = createQuoteClient(
  { providers: { relay: { referrer: 'your-app.example' } }, fallback: 'none' },
  { carriers: [/* ... */] },
)
```

No `apiKey` field appears anywhere in this block. That is deliberate —
see "Common mistakes," below.

### Your own service, with fallback

```ts
const client = createQuoteClient({
  serviceUrl: 'https://your-quote-service.example',
  fallback: 'direct',
})
```

## Step 4 — run the free backend

Use this step only if you chose "your own service" in Step 1. The
backend is `@gibs/quote-service`. It works without any provider key, at
each provider's own public rate limit.

```sh
mkdir quote-service && cd quote-service
curl -fsSLO https://raw.githubusercontent.com/gibsfinance/interface-releases/main/self-host/docker-compose.yml
curl -fsSLO https://raw.githubusercontent.com/gibsfinance/interface-releases/main/self-host/.env.example
cp .env.example .env
# edit .env: set ALLOWED_ORIGINS to your frontend's real origin
docker compose up
```

Check [`self-host/README.md`](../../self-host/README.md) in this
repository for its current "Image availability" notice before relying on
this path — the compose file pulls `ghcr.io/gibsfinance/quote-service`,
and that image may not be published on every date this guide is read.

This starts one container, published to `127.0.0.1:8080` — reachable
only from the machine it runs on. Put a reverse proxy in front of it
(Caddy, nginx, Traefik) before pointing a real frontend at it from
another machine; see `self-host/README.md`'s own "Expose it" section for
doing that safely.

## Step 5 — check it

```sh
curl -H 'Origin: http://localhost:5173' http://127.0.0.1:8080/v1/info
```

A correct response looks like:

```json
{"service":"quote-service","contract":1,"providers":["lifi","relay","near-intents"],"version":"1.13.0"}
```

Calling `/v1/info` without an `Origin` header answers 403 by design —
every route except `/healthz` and `/readyz` enforces Cross-Origin
Resource Sharing. Set `Origin` (or use a real browser request from your
configured frontend) to see it succeed.

`/healthz` reports only that the process is running. `/readyz` reports
that at least one provider's chain list has loaded — check this one
before sending real traffic; it can take a few seconds after start.

## Common mistakes

- **Cross-Origin Resource Sharing rejects every request.** The backend's
  `ALLOWED_ORIGINS` defaults to nothing you own. Set it to your exact
  frontend origin in `.env` before testing from a browser — a missing or
  mismatched entry answers every request with 403, including `/v1/info`.
- **A provider key ends up in a browser bundle.** A real `apiKey` belongs
  only in a backend or a self-hosted service's own environment variables
  — never in a `QuoteClientConfig` built for "browser, direct" mode. If
  your bundler can reach it, so can anyone who opens the browser's
  developer tools.
- **A request names a chain the aggregators must never see.** If your own
  application has a chain that should never reach LI.FI, Relay, or NEAR
  Intents — because those providers do not support it, or because
  sending it there would leak information you would rather keep local —
  set `neverSendChainIds` in your configuration. An unset list enforces
  nothing.
- **A scaled figure gets treated as an exact one.** A cached answer for a
  nearby amount comes back with `basis: 'scaled-from-nearby-amount'`, not
  `basis: 'exact'`. Never sign a transaction, or show a user a committed
  number, from a scaled answer — ask again with `precision: 'exact'` when
  you need a number safe to act on. The package's own type system refuses
  to compile if you pass a scaled amount to its execution-amount decoder,
  but only if you use that decoder — a hand-rolled read of the raw
  `expectedAmount` string bypasses that protection.

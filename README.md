# Interface Releases

Signed release manifests for the [gibs.finance](https://gibs.finance)
interface, published from the private `gibsfinance/bridge` repository. A
Cloudflare Worker behind `ipfs.gibs.finance` reads this repository (with a
Cloudflare Workers KV copy as a faster path) to resolve the current release
and redirect to it on the InterPlanetary File System.

This repository holds no application source code. It exists so a public
GitHub repository can carry release artifacts that the owner's self-hosted
IPFS node watches and pins — the source repository stays private.

**Reading this with an artificial intelligence agent?** Start with
[`llms.txt`](./llms.txt) — a short, structured map of this repository
meant for exactly that. It links the verification steps below and the
integration skill in [`skills/integrate-gibs-quotes/`](./skills/integrate-gibs-quotes)
for adding cross-chain quotes to your own application.

## What is here

- **`version.json`** — the current signed release manifest, on this
  repository's default branch. It lists every recent release by channel
  (`staging`, `master`), the release each channel currently points at, and
  which release `latest` tracks.
- **`version.json.sig`** — a detached Ed25519 signature over the exact bytes
  of `version.json`, byte for byte.
- **`public-keys.json`** — the Ed25519 public key registry, keyed by
  `keyId`, used to verify `version.json.sig`. Mirrored here on every publish
  from the private repository's own copy.
- **One GitHub Release per publish** — tagged `<version>+<7-character
  commit>` (for example `1.13.1+e4f5a6b`), each carrying:
  - `interface-<id>.car` — a Content Archive of that release's built
    interface (`packages/ui/dist`), packed with Content ID version 1 and raw
    leaves. This is the file the IPFS node pins.
  - `version.json` — the manifest this release updated the default branch
    to, so a reader can recover the exact manifest a given release shipped
    with even after later releases move the default branch copy forward.
  - `version.json.sig` — the signature over that same `version.json`.

## How to verify a release

1. Fetch `version.json` and `version.json.sig` (either from this
   repository's default branch, or from a specific release's assets).
2. Fetch `public-keys.json` and find the entry matching the `keyId` in
   `version.json.sig`.
3. Verify the Ed25519 signature in `version.json.sig` over the exact bytes
   of `version.json`, using that public key.
4. Read the release you want out of `version.json`'s `releases` array (or
   `channels`/`latest`) to get its Content ID (`cid`).
5. Fetch the content from any IPFS gateway using that Content ID, for
   example `https://<cid>.ipfs.4everland.io/`.

The full manifest schema, the signing scheme, and the write order a publish
follows are documented in `gibsfinance/bridge`'s `docs/ipfs-releases.md`.

## Run your own backend

The gibs.finance interface works on its own — it can call LI.FI, Relay and
NEAR Intents directly from the browser. It also supports an optional
backend, `@gibs/quote-service`, that fans the same quote requests out to
those three providers and caches the answers, so many visitors asking the
same question share one upstream call instead of each firing their own.
Point your own build of the interface at your own instance instead of
gibs's, or run one for any other reason.

See [`self-host/`](./self-host) for the guide — read its "Image
availability" notice first, since the published container image this needs
does not exist yet. For adding quotes to your own application, rather than
just running the backend, see
[`skills/integrate-gibs-quotes/SKILL.md`](./skills/integrate-gibs-quotes/SKILL.md).

## Rotation

A `public-keys.json` entry is never removed while any reachable release in
`version.json`'s history still carries a signature made with it — doing so
would make an otherwise-valid old manifest unverifiable.

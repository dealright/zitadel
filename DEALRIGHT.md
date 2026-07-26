# Dealright fork notes

This is a fork of [`zitadel/zitadel`](https://github.com/zitadel/zitadel),
consumed by [`dealright/dealright`](https://github.com/dealright/dealright)
as a git submodule at `/auth`, replacing the stock
`ghcr.io/zitadel/zitadel-login` image currently serving
`auth.dealright.ai`. We want full control over styling, fonts, copy,
imagery, and the ability to add custom pages over time -- the stock image
only exposes 5 layout-level theme env vars (roundness/layout/spacing/
appearance/background image), not fonts, colors, or copy.

## Why a real fork, not a plain vendored copy

Full git lineage (`origin` = this fork, `upstream` = `zitadel/zitadel`)
means future Zitadel login-app improvements/security fixes can be pulled
with an ordinary `git fetch upstream && git merge upstream/main` (or a
newer tag), instead of manually re-diffing a copy-pasted export every time.

## Sparse-checkout

Only a slice of the full `zitadel/zitadel` monorepo is materialized here
(the Go backend, `console/`, e2e `tests/`, etc. are excluded):

```
apps/login/               the Next.js app (this is what we customize)
packages/zitadel-client/  @zitadel/client -- typed API client
packages/zitadel-proto/   @zitadel/proto -- generated from proto/ via buf
proto/                    .proto API definitions
```

Configured via `git sparse-checkout set apps/login packages/zitadel-client
packages/zitadel-proto proto` (cone mode). A fresh clone/submodule-update
needs this re-run once (`git sparse-checkout list` to check current state).

## Version

Pinned base: **`v4.16.0`**, matching the currently-deployed
`ghcr.io/zitadel/zitadel:v4.16.0` and `ghcr.io/zitadel/zitadel-login:v4.16.0`
containers on `use1-data-01` (API compatibility matters -- this isn't just
a frontend, see below). Our customizations live on the `dealright-custom`
branch, based off the `v4.16.0` tag.

## Pulling a newer upstream version later

```bash
git fetch upstream --tags
git merge upstream/vX.Y.Z   # or upstream/main for latest
# resolve conflicts in apps/login (styling/copy/pages we've customized) --
# packages/zitadel-client, packages/zitadel-proto, and proto/ are unlikely
# to conflict since we don't edit those.
git push origin dealright-custom
```
Then bump the submodule pointer in `dealright/dealright` and re-pin the
Docker build's Zitadel-server-version assumption if the API changed.

## Build

```bash
cd auth
pnpm install
pnpm --filter @zitadel/proto run generate   # buf generate ../../proto -> es/cjs/types
pnpm --filter @zitadel/client run build     # tsup
pnpm --filter @zitadel/login run build      # next build (standalone output)
```

## Required Zitadel-side config

A service account with `IAM_LOGIN_CLIENT` instance membership + a personal
access token (mirrors the stock `zitadel-zitadel-login-1` container's
`ZITADEL_SERVICE_USER_TOKEN_FILE` wiring). Env vars: `ZITADEL_API_URL`,
`ZITADEL_SERVICE_USER_ID`, `ZITADEL_SERVICE_USER_TOKEN` (or `_FILE`).

## Deploy

See `infra/greenfield/auth.Dockerfile` in `dealright/dealright` (multi-stage:
`pnpm install` + build happen INSIDE the image, unlike upstream's
`apps/login/Dockerfile`, which only copies a pre-built `.next/standalone`
produced by a separate CI step).

**This app proxies the live OIDC endpoints** (`/oauth/*`, `/oidc/*`,
`/.well-known/*`) to the Zitadel backend (`ZITADEL_API_URL`) -- it is not
just a themed frontend, it sits in front of every login/token request.
Cutting it in to replace the stock `zitadel-zitadel-login-1` container
means re-routing what `auth.dealright.ai` resolves to. Do not cut over
without first proving the new container serves an equivalent login flow
standalone (a parallel test container/hostname first).

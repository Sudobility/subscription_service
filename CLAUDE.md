# Subscription Service - AI Development Guide

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when
> the user explicitly asks in that turn**. Approval for an earlier change does not
> carry forward, and finishing a task is not permission to commit it.
> Stay on the current branch; do not create feature branches for code changes.

## What this is

`@sudobility/subscription_service` — a small, server-side TypeScript library that
wraps the RevenueCat REST API v1 (`GET /subscribers/{user_id}`) to return a user's
active entitlements and subscription details. One public class: `SubscriptionHelper`.

- **Stack**: TypeScript (strict, ES2020), ESM-only output, Bun package manager,
  Vitest tests, ESLint + Prettier.
- **No runtime `dependencies`**; all HTTP uses native `fetch`.
- **License**: BUSL-1.1.

## Commands (all verified)

```bash
bun install          # needs GitHub Packages auth (see Registry gotcha)
bun run typecheck    # tsc --noEmit
bun run lint         # eslint src/
bun run test:unit    # vitest run   <-- use this, NOT `bun test`
bun run build        # tsc -p tsconfig.esm.json + scripts/fix-esm-imports.mjs -> dist/
bun run verify       # typecheck -> lint -> test:unit -> build  (pre-commit gate)
bun run format       # prettier --write
```

## Architecture

```
src/
  index.ts                      # barrel: re-exports types + helper + NONE_ENTITLEMENT
  types/entitlements.ts         # RevenueCat* API response interfaces
  types/subscription.ts         # SubscriptionInfo (library's return shape)
  helpers/SubscriptionHelper.ts # the only logic: fetch + filter + map
tests/SubscriptionHelper.test.ts # Vitest, fetch fully mocked
scripts/fix-esm-imports.mjs     # post-build: adds .js to relative import specifiers
```

`SubscriptionHelper.getSubscriptionInfo()` is the core. It fetches the subscriber,
then for each entitlement:
1. Drops expired ones (`expires_date` in the past; null = lifetime = active).
2. In production (`testMode=false`) drops ones whose subscription
   `environment === "sandbox"`.
3. Tracks the entitlement with the **earliest** `purchase_date`; its subscription
   supplies `platform`, `store`, `productIdentifier`, `expiresDate`, `sandbox`,
   `willRenew`. `willRenew` is `unsubscribe_detected_at == null`.

`platform` is mapped from the store via `STORE_PLATFORM_MAP`:
`stripe`/`rc_billing` → Web, `app_store` → iOS, `play_store` → Android,
`mac_app_store` → macOS, else `null`.

`getEntitlements()` is a thin wrapper returning only `info.entitlements`.

## Key conventions

- Public API is re-exported through `src/index.ts`; add new exports there.
- Both methods throw on empty/non-string `userId`. `userId` is URL-encoded.
- Requests use an `AbortController` timeout (`timeoutMs`, default 10s).
- `SubscriptionInfo` return shape is the library's contract — changing/adding
  fields affects downstream consumers; update tests to match.

## Gotchas (non-obvious)

- **`bun test` is wrong.** Bare `bun test` runs Bun's native runner, which is not
  compatible with the Vitest globals (`vi.unstubAllGlobals`, etc.) and fails.
  Always use `bun run test:unit`.
- **Registry = GitHub Packages, not public npm.** `.npmrc` sets
  `@sudobility:registry=https://npm.pkg.github.com`. `bun install` and publishing
  need a GitHub Packages token (Actions secret `GH_PACKAGES_READ_TOKEN`).
  Publishing is done by CI (`.github/workflows/ci-cd.yml`, reusable workflow with
  `publish-target: github`), not manual `npm publish`.
- **`@sudobility/types` is a runtime import but only in `devDependencies`.** It
  provides `NONE_ENTITLEMENT` (the string `"none"`) and `SubscriptionPlatform`.
  There is no `dependencies` or `peerDependencies` block in `package.json`.
- **Sandbox filtering keys on `environment`, not the `sandbox` flag.** The returned
  `sandbox` field reflects the raw `sandbox` boolean, but the production filter
  excludes subscriptions where `environment === "sandbox"`.
- **`NONE_ENTITLEMENT` is the string `"none"`**, never null/undefined. A 404 from
  RevenueCat is not an error — it returns the `["none"]` result.
- **ESM-only, no CJS.** The build is `tsconfig.esm.json` only; there is no
  `tsconfig.cjs.json`. `fix-esm-imports.mjs` must run after `tsc` or the emitted
  ESM has extensionless relative imports that Node won't resolve.

## Workspace

Part of the Sudobility multi-project workspace (parent directory). Consumed by
other Sudobility backend services; see `../CLAUDE.md` for the full dependency graph
and build order. `plans/IMPROVEMENTS.md` tracks proposed enhancements.

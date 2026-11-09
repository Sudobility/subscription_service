# @sudobility/subscription_service

Shared backend library for subscription management using RevenueCat. Provides a
`SubscriptionHelper` class that calls the RevenueCat REST API v1
(`GET /subscribers/{user_id}`) to fetch a user's active entitlements and
subscription details. **Server-side only** — it uses a RevenueCat secret API key.

- ESM-only TypeScript library (no CJS build), published to **GitHub Packages**.
- No runtime `dependencies`; all HTTP calls use native `fetch`.
- Requires `@sudobility/types` at runtime for the `NONE_ENTITLEMENT` constant and
  `SubscriptionPlatform` enum (see [Dependencies](#dependencies)).

## Installation

This package is published to GitHub Packages under the `@sudobility` scope, not
the public npm registry. A project consuming it needs an `.npmrc` pointing the
scope at GitHub Packages with a read token:

```
@sudobility:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GH_PACKAGES_READ_TOKEN}
```

Then:

```bash
bun add @sudobility/subscription_service
```

## Usage

```typescript
import { SubscriptionHelper } from "@sudobility/subscription_service";
import { NONE_ENTITLEMENT } from "@sudobility/subscription_service"; // re-exported

const helper = new SubscriptionHelper({
  revenueCatApiKey: process.env.REVENUECAT_API_KEY!,
  // baseUrl?: string   — defaults to https://api.revenuecat.com/v1
  // timeoutMs?: number — request timeout, defaults to 10000 (10s)
});

// Active entitlement names only
const entitlements = await helper.getEntitlements(userId);
// => ["pro"]  or  ["none"]

// Full subscription info
const info = await helper.getSubscriptionInfo(userId);

// Check subscription status
if (!info.entitlements.includes(NONE_ENTITLEMENT)) {
  // User has an active subscription
}

// Include sandbox purchases (testing): pass testMode = true
const testInfo = await helper.getSubscriptionInfo(userId, true);
```

## API

### `SubscriptionHelper`

| Method | Signature | Description |
|--------|-----------|-------------|
| `getEntitlements` | `(userId: string, testMode?: boolean) => Promise<string[]>` | Active entitlement names (convenience wrapper over `getSubscriptionInfo`). |
| `getSubscriptionInfo` | `(userId: string, testMode?: boolean) => Promise<SubscriptionInfo>` | Full subscription info. |

- `testMode` defaults to `false`. When `false`, subscriptions whose
  `environment === "sandbox"` are excluded; when `true`, they are included.
- Both methods throw if `userId` is not a non-empty string.
- A `404` from RevenueCat is **not** an error — it means the user is unknown and
  the "none" result is returned.
- A non-404, non-OK response throws `Error("RevenueCat API error: <status> <statusText>")`.

### `SubscriptionHelperConfig`

```typescript
interface SubscriptionHelperConfig {
  revenueCatApiKey: string; // RevenueCat secret API key (server-side)
  baseUrl?: string;         // default: https://api.revenuecat.com/v1
  timeoutMs?: number;       // default: 10000
}
```

### `SubscriptionInfo`

```typescript
interface SubscriptionInfo {
  entitlements: string[];              // active entitlement names, or ["none"]
  subscriptionStartedAt: Date | null;  // earliest active purchase date
  platform: SubscriptionPlatform | null; // mapped from the store of that purchase
  productIdentifier: string | null;
  expiresDate: Date | null;            // null for lifetime / no subscription
  sandbox: boolean;
  store: string | null;                // raw store id, e.g. "stripe", "app_store"
  willRenew: boolean;                  // false if unsubscribe was detected
  managementUrl: string | null;        // e.g. Stripe billing portal
}
```

The `platform`, `store`, and purchase-date fields are taken from the entitlement
with the **earliest** active purchase date. Store → platform mapping:

| Store (`store`) | `platform` |
|---|---|
| `stripe`, `rc_billing` | `Web` |
| `app_store` | `iOS` |
| `play_store` | `Android` |
| `mac_app_store` | `macOS` |
| anything else | `null` |

### Exports

`SubscriptionHelper`, `SubscriptionHelperConfig`, `SubscriptionInfo`,
`RevenueCatEntitlement`, `RevenueCatSubscription`, `RevenueCatSubscriberResponse`,
and `NONE_ENTITLEMENT` (re-exported from `@sudobility/types`).

## Dependencies

`@sudobility/types` is imported at **runtime** (`NONE_ENTITLEMENT`,
`SubscriptionPlatform`) but is currently declared only in `devDependencies`.
Consumers in the Sudobility workspace already have it; standalone consumers must
install it alongside this package.

## Development

Uses Bun as the package manager (do not use npm/yarn/pnpm to install).

```bash
bun install          # install deps (needs GitHub Packages auth, see Installation)

bun run typecheck    # tsc --noEmit
bun run lint         # eslint src/
bun run lint:fix     # eslint src/ --fix
bun run format       # prettier --write
bun run format:check # prettier --check
bun run test:unit    # vitest run  (NOT `bun test` — that uses Bun's native runner)
bun run test:watch   # vitest watch
bun run build        # tsc -p tsconfig.esm.json + scripts/fix-esm-imports.mjs -> dist/
bun run dev          # tsc --watch
bun run clean        # rm -rf dist
bun run verify       # typecheck -> lint -> test:unit -> build
```

> Run tests with `bun run test:unit`, never bare `bun test`. `bun test` invokes
> Bun's built-in runner, which is not API-compatible with the Vitest globals used
> here and will fail.

## Build & Publish

- Build output is **ESM only** to `dist/` (`.js`, `.d.ts`, `.d.ts.map`, source maps).
  `scripts/fix-esm-imports.mjs` rewrites relative import specifiers to add `.js`
  extensions so the emitted ESM resolves correctly.
- Published files: `dist/**/*` and `CLAUDE.md` (see `files` in `package.json`).
- Publishing is handled by CI (`.github/workflows/ci-cd.yml`, which calls a
  reusable workflow with `publish-target: github`) on pushes to `main`/`develop`.
  `prepublishOnly` runs `clean` + `verify`.

## License

BUSL-1.1

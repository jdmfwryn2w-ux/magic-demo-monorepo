# AGENTS.md

Instructions for coding agents working in this repository.

This is a **single Next.js app**, not a package monorepo. There is no `pnpm-workspace.yaml`, Turbo, or `packages/` tree. Two Magic Labs wallet demos share one Next process.

Production reference: [https://demo.magic.link](https://demo.magic.link). Docs: [Magic Embedded Wallets](https://docs.magic.link/embedded-wallets/introduction), [Server Wallets Express API](https://docs.magic.link/server-wallets/express-api/overview).

## Commands

Use **pnpm**. The lockfile is `pnpm-lock.yaml` (v9). Do not add npm or yarn lockfiles.

```bash
pnpm i                 # install
pnpm dev               # next dev --turbopack (http://localhost:3000)
pnpm build             # next build --turbopack
pnpm start             # next start
pnpm lint              # eslint (next/core-web-vitals + next/typescript)
npx tsc --noEmit       # typecheck (no package.json script; tsconfig has noEmit: true)
```

There is **no** test runner, formatter, or CI workflow in this repo. Do not invent `pnpm test` / `pnpm format`. If you add tests, introduce a runner and a `test` script rather than one-off files with no way to run them.

Copy `.env.local.example` to `.env.local` and fill credentials before `pnpm dev`. The app will boot without them, but auth and wallet calls will fail.

## Project layout

```
src/app/page.tsx                         # home: choose Embedded vs Server Wallet
src/app/(demo)/embedded-wallet/          # Magic SDK login + wallet dashboard
src/app/(demo)/server-wallet/            # Google OIDC + TEE wallet dashboard
src/app/api/auth/[...nextauth]/route.ts  # NextAuth Google provider
src/app/api/tee/wallet/smart-wallet/     # Alchemy EIP-7702 smart wallet (server-only)
src/components/embedded-wallet/          # embedded auth + signing UI
src/components/server-wallet/            # server-wallet signing UI
src/contexts/                            # Console, EmbeddedWallet, ServerWallet
src/lib/embedded-wallet/get-magic.ts     # Magic SDK singleton
src/lib/server-wallet/                   # TEE clients + per-chain signing
src/types/tee-types.ts                   # Fridge / Express API paths
```

`(demo)` is a Next.js **route group**. URLs do not include it:

| URL | Purpose |
| --- | --- |
| `/` | Product picker |
| `/embedded-wallet` | Embedded login |
| `/embedded-wallet/callback` | Magic OAuth2 redirect |
| `/embedded-wallet/wallet` | Embedded methods dashboard |
| `/server-wallet` | Google sign-in |
| `/server-wallet/wallet` | Server Wallet methods dashboard |

Path alias: `@/*` → `./src/*`. Static assets are often imported from `public/...` (for example `import logoGoogle from "public/logos/logo-google.svg"`).

## Architecture

### Embedded Wallet (client Magic SDK)

- Singleton: `MagicService` in `src/lib/embedded-wallet/get-magic.ts`.
- Key: `NEXT_PUBLIC_MAGIC_EMBEDDED_WALLET_KEY`.
- Extensions: OAuth2, Solana (devnet), Hedera (mainnet), EVM (Polygon default + Optimism), Wallet Kit, Passkey.
- Login page: Wallet Kit (MetaMask, Coinbase, Phantom, Rabby, WalletConnect), email OTP, OAuth (Google enabled; other providers are commented in `OAuthProvidersConfig.ts`), passkey.
- State: `WalletProvider` in `src/contexts/EmbeddedWalletContext.tsx`. Polygon / Ethereum / Optimism share the Ethereum wallet address; Hedera and Solana have their own.
- Network is stored in `localStorage.magic_selectedNetwork`. Wallet Kit logins also set `magic_widget_login_method`.
- Console panel (`ConsolePanel`) is shown only on `/embedded-wallet*` routes (`AppLayout`).

Signing UI is network-specific: `SignMethods` (EVM), `HederaSignMethods`, `SolanaSignMethods`. Reuse `MethodsCard` / `SigningMethodsLayout` for new methods.

### Server Wallet (TEE / Express API)

- Google OIDC via NextAuth v4 (`src/app/api/auth/[...nextauth]/route.ts`). JWT session stores `idToken` and refreshes via Google's token endpoint.
- NextAuth is **not** global. Only the server-wallet layout wraps `SessionProvider` + `ServerWalletProvider`.
- Browser TEE client: `src/lib/server-wallet/express-proxy.ts` (uses `getSession()`).
- Server TEE client: `src/lib/server-wallet/express.ts` (JWT passed in). Do not mix the two.
- Backend: `https://tee.express.magiclabs.com` (comments call this Fridge). Paths in `src/types/tee-types.ts`: `/v2/wallet`, `/v2/wallet/sign/data`, `/v2/wallet/sign/message`, `/v2/wallet/sign/eip7702`.
- Headers: `Authorization: Bearer {idToken}`, `X-Magic-API-Key`, `X-OIDC-Provider-ID`, `X-Magic-Chain`, and a **hardcoded** `X-Magic-Referrer: https://demo.magic.link`.
- UI network `avalanche` maps to API chain `ETH` (same EOA). Fuji-specific send/broadcast lives in `src/lib/server-wallet/avalanche.ts`.
- Smart wallet: `POST /api/tee/wallet/smart-wallet` wraps the TEE EOA in an Alchemy EIP-7702 account on **Base Sepolia**. Fridge `r`/`s` are decimal strings and must be converted to 32-byte hex. Do not send a plain ETH transfer to the EOA itself once delegated.

Signing UI: `EVMSignMethods` (default), `AvalancheSignMethods`, `SolanaSignMethods`, plus `AlchemySmartWallet` on EVM.

## Environment

Documented in `.env.local.example` and README:

```
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
NEXT_PUBLIC_OIDC_PROVIDER_ID
NEXTAUTH_SECRET
NEXT_PUBLIC_MAGIC_EMBEDDED_WALLET_KEY
NEXT_PUBLIC_MAGIC_SERVER_WALLET_KEY
SERVER_WALLET_SECRET_KEY
```

Used in code but **not** in the example file:

- `ALCHEMY_API_KEY` — required for the smart-wallet demo
- `ALCHEMY_GAS_POLICY_ID` — optional; if set, the demo marks txs as sponsored

`SERVER_WALLET_SECRET_KEY` is listed in the example but unused in source. Local NextAuth often also needs `NEXTAUTH_URL=http://localhost:3000`.

`.gitignore` ignores `.env*` except `.env.local.example`. Never commit real keys.

`NEXT_PUBLIC_*` values ship to the browser. Google `client_secret` and `NEXTAUTH_SECRET` must stay server-only. The smart-wallet route is server-side because it uses Alchemy + secret material.

## Code style

- TypeScript strict. ESLint allows `@typescript-eslint/no-explicit-any` (Magic SDK and `userInfo` are typed as `any`).
- Tailwind CSS v4 via `@tailwindcss/postcss`. Merge classes with `cn()` from `src/utils/tailwind.ts`.
- `"use client"` on interactive pages and components. Root `layout.tsx` is a server component.
- Pages/layouts: `export default`. Shared UI: prefer named exports (`export const Button`, `export function UserInfo`). Existing exception: `server-wallet/UserMethods.tsx` is a default export.
- Match neighboring files for quotes and trailing commas (double quotes are more common).
- Reuse `Button`, `PageHeader`, `Card`, `MethodsCard`, `LoadingScreen` instead of new primitives.
- Keep product folders split: `components/embedded-wallet/*` vs `components/server-wallet/*`.

## Security and demo-only behavior

- `UserMethods` can **export a private key** via `@magic-sdk/server-wallets`. That is demo-only; do not copy it into production patterns.
- Do not log tokens, JWTs, or private keys to the console panel.
- Embedded Polygon and Hedera point at **mainnet**. Solana is **devnet**. Server Avalanche is **Fuji**. Alchemy smart wallet is **Base Sepolia**. Do not send real funds in demos.

## PR / change checklist

1. Stay in this directory (`magic-demo-monorepo`) for git and package commands.
2. Touch only the product you are changing unless a shared component is required.
3. Run `pnpm lint` and `npx tsc --noEmit` on TypeScript/UI changes.
4. Run `pnpm build` if you change Next config, env usage, or API routes.
5. Do not commit `node_modules/`, `.next/`, or `.env.local`.
6. Do not regenerate or convert fonts under `public/fonts/` unless the task is about fonts.
7. Keep commit messages short and imperative (existing history: `bump nextjs version`, `adds avalanche c chain section`).

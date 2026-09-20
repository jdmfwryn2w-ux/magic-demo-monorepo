# Magic Demo

Interactive Next.js demo for [Magic](https://magic.link) Embedded Wallet and Server Wallet (TEE / Express API). Production URL: https://demo.magic.link

Read `README.md` for setup. Copy `.env.local.example` to `.env.local` and fill in credentials from the Magic and Google developer dashboards.

## Stack

- Next.js 16 App Router, React 19, TypeScript (strict)
- Tailwind CSS 4 + PostCSS
- pnpm (lockfile: `pnpm-lock.yaml`)
- next-auth v4 (Google OIDC) for Server Wallet
- `magic-sdk` + extensions (`oauth2`, `solana`, `hedera`, `evm`, `wallet-kit`, `passkey`) for Embedded Wallet
- Alchemy Account Kit for the EIP-7702 smart-wallet demo

## Commands

- `pnpm i` — install dependencies
- `pnpm dev` — Next.js + Turbopack on port 3000
- `pnpm lint` — ESLint (`next/core-web-vitals`, `next/typescript`; `@typescript-eslint/no-explicit-any` is off)
- `pnpm build` — production build
- `pnpm start` — serve the production build

There is no test suite. For UI work, exercise the flow in the browser: home → Embedded Wallet or Server Wallet → auth → wallet methods.

## Architecture

Two product surfaces share layout and UI primitives but have separate auth, context, and signing stacks.

| Path | Role |
| --- | --- |
| `src/app/page.tsx` | Home: choose Embedded vs Server Wallet |
| `src/app/(demo)/embedded-wallet/` | Client-side Magic SDK auth and wallet methods |
| `src/app/(demo)/server-wallet/` | Google OIDC + Magic TEE Express API wallets |
| `src/app/api/auth/[...nextauth]/route.ts` | next-auth Google provider and token refresh |
| `src/app/api/tee/wallet/smart-wallet/route.ts` | Server route: wrap TEE EOA in Alchemy smart wallet (Base Sepolia) |
| `src/lib/embedded-wallet/get-magic.ts` | Singleton `MagicService` (SDK + chain switch) |
| `src/lib/server-wallet/express-proxy.ts` | Browser TEE client (uses NextAuth session JWT) |
| `src/lib/server-wallet/express.ts` | Server TEE client (explicit JWT) |
| `src/lib/server-wallet/wallet.ts` | Maps UI networks to TEE chain ids (`avalanche` → `ETH`) |
| `src/contexts/` | `EmbeddedWalletContext`, `ServerWalletContext`, `ConsoleContext` |
| `src/components/` | Shared UI plus `embedded-wallet/` and `server-wallet/` method cards |
| `src/types/tee-types.ts` | TEE endpoint paths and response types |

Routes:

- `/` — product picker
- `/embedded-wallet` — email OTP, OAuth, passkey, Wallet Kit widget
- `/embedded-wallet/wallet` — user info + network-specific signing
- `/embedded-wallet/callback` — OAuth redirect callback
- `/server-wallet` — Google sign-in
- `/server-wallet/wallet` — TEE wallet + EVM / Solana / Avalanche / Alchemy methods

`AppLayout` mounts `ConsoleProvider` globally and shows `ConsolePanel` only on `/embedded-wallet*`.

## Conventions

- Path alias `@/*` → `src/*`. Import public assets from `public/` (e.g. `public/icons/...`).
- Client pages and wallet UI are `"use client"`. Keep TEE secrets and `SERVER_WALLET_SECRET_KEY` off the client.
- Reuse `Button`, `Card`, `MethodsCard`, `PageHeader`, `LoadingScreen`, `WalletAddress`. New Magic/TEE calls belong in a methods component that feeds `MethodsCard` tabs (`functionName`, `payload`, `handler`).
- Log Magic/TEE calls through `useConsole()` (`LogType` + `LogMethod`). Add a `LogMethod` enum value when introducing a new SDK method.
- Network-specific signing: Embedded Wallet uses `Network` in `EmbeddedWalletContext` (Polygon default; Ethereum/Optimism share the ETH address; Hedera and Solana have their own method cards). Server Wallet stores the selected network in `localStorage` key `server-wallet-network`.
- Merge classes with `cn()` from `src/utils/tailwind.ts`. Prefer existing Tailwind tokens (`slate-*`, `font-gt-super`) over new palettes.
- Named exports for components. Match existing file names (`PascalCase.tsx` for components, kebab-case for lib modules).
- Prefer clear names over comments. Do not commit `.env*` files other than `.env.local.example`.

## Security

- Publishable keys: `NEXT_PUBLIC_MAGIC_EMBEDDED_WALLET_KEY`, `NEXT_PUBLIC_MAGIC_SERVER_WALLET_KEY`, `NEXT_PUBLIC_OIDC_PROVIDER_ID`.
- Server-only: `GOOGLE_CLIENT_SECRET`, `NEXTAUTH_SECRET`, `SERVER_WALLET_SECRET_KEY`.
- Never log JWTs, refresh tokens, or secret keys. TEE calls send `Authorization: Bearer <idToken>` plus Magic headers; do not copy those tokens into the demo console payload.
- Browser TEE traffic goes to `https://tee.express.magiclabs.com`. Do not point production keys at local mock servers unless the task says to.

## Docs

- Magic JS SDK: https://magic.link/docs
- Magic Server Wallets / Express: https://magic.link/docs
- next-auth: https://next-auth.js.org
- Next.js App Router: https://nextjs.org/docs

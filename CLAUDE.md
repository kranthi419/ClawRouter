# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build
npm run build          # TypeScript compilation via tsup
npm run dev            # Watch mode (tsup)

# Type checking & linting
npm run typecheck      # tsc --noEmit
npm run lint           # ESLint on src/
npm run format         # Prettier (write)
npm run format:check   # Prettier (validate only)

# Tests
npm test               # Vitest unit tests (vitest run)
npm run test:watch     # Vitest in watch mode

# Resilience tests (require running infrastructure)
npm run test:resilience:errors     # Error handling
npm run test:resilience:stability  # 5-min stability (DURATION_MINUTES to override)
npm run test:resilience:lifecycle  # State management
npm run test:resilience:quick      # errors + lifecycle
npm run test:resilience:full       # All resilience tests

# E2E / Docker tests
npm run test:e2e:tool-ids
npm run test:docker:install
npm run test:docker:edge-cases
npm run test:docker:integration
```

## Architecture

ClawRouter is an OpenClaw plugin that runs as a local HTTP proxy on port 8402. It intercepts LLM API calls, routes them to the cheapest capable model, and pays via USDC micropayments using the x402 protocol (no credit cards — wallet signatures only).

### Request flow

```
Client (OpenClaw) → Proxy (port 8402)
  1. Deduplication — SHA-256 hash check against 30s response cache
  2. Smart routing — 15-dimension weighted scorer → tier → model selection
  3. Balance check — USDC balance with buffer (EVM or Solana)
  4. SSE heartbeat — streaming headers sent immediately, `:heartbeat` every 2s
  5. x402 payment — EIP-712 signing (EVM/Base) or SPL tx signing (Solana)
  6. API call — BlockRun API (blockrun.ai/api or sol.blockrun.ai/api)
  7. Response — streamed or buffered back to client
  8. Logging — usage recorded to ~/.openclaw/blockrun/logs/
```

### Key source files

| File | Role |
|------|------|
| `src/proxy.ts` | HTTP server, SSE streaming, fallback chain, payment orchestration |
| `src/index.ts` | OpenClaw plugin entry point, skill installation, CLI handlers |
| `src/router/index.ts` | Routing orchestration and tier selection |
| `src/router/rules.ts` | 15-dimension weighted scorer (keywords in 9 languages) |
| `src/router/selector.ts` | Tier-to-model mapping and cost ranking |
| `src/router/config.ts` | Default tier definitions, keyword lists, dimension weights |
| `src/models.ts` | 55+ model definitions — pricing, aliases, context windows, features |
| `src/wallet.ts` | BIP-39 mnemonic generation/recovery, EVM + Solana key derivation (SLIP-10) |
| `src/auth.ts` | Wallet key resolution: file → env → auto-generate |
| `src/x402.ts` | EIP-712 typed data signing for EVM, x402 protocol client |
| `src/balance.ts` | EVM USDC balance monitoring (60s cache) |
| `src/solana-balance.ts` | Solana SPL token balance monitoring (60s cache + retry) |
| `src/payment-preauth.ts` | Pre-authorization cache to skip 402 round trips (EVM only) |
| `src/dedup.ts` | Request deduplication via SHA-256, 30s response cache |
| `src/errors.ts` | Custom error types: `EmptyWallet`, `InsufficientFunds`, etc. |

### Routing system

Four tiers: **SIMPLE** → **MEDIUM** → **COMPLEX** → **REASONING**

Three profiles (selected via `/model` command):
- `auto` — balanced cost/quality (default)
- `eco` — maximum cost reduction
- `premium` — best quality, no routing

The scorer in `src/router/rules.ts` evaluates 15 dimensions (reasoning markers 0.18, code presence 0.15, token count 0.12, technical keywords 0.10, ...) and uses sigmoid calibration to produce a confidence score that maps to a tier.

### Payment

- **EVM/Base**: signs EIP-712 typed data, sends `X-PAYMENT` header, calls `blockrun.ai/api`
- **Solana**: signs SPL token transfer tx, calls `sol.blockrun.ai/api`
- Chain is selected at startup based on wallet type
- Pre-authorization (EVM only) caches signed payments with 20% buffer to skip 402 round trips

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `BLOCKRUN_WALLET_KEY` | auto-generated | Wallet private key |
| `BLOCKRUN_PROXY_PORT` | `8402` | Proxy listen port |
| `CLAWROUTER_SOLANA_RPC_URL` | mainnet-beta | Solana RPC endpoint |
| `CLAWROUTER_DISABLED` | — | Set to `true` to disable smart routing |
| `CLAWROUTER_WORKER` | — | Enable experimental worker/health-check mode |

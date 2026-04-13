# JinOS Architecture
This document describes the system design, monorepo structure, and technical decisions behind JinOS.
---
## Overview
JinOS is a **full-stack TypeScript monorepo** managed with `pnpm` workspaces.

jinos/
├── artifacts/
│ ├── api-server/ ← Node.js + Express REST API
│ └── jinos/ ← React + Vite frontend
├── lib/
│ ├── db/ ← PostgreSQL schema (Drizzle ORM)
│ ├── api-zod/ ← Zod validators (source of truth for types)
│ └── api-client-react/ ← Fetch client + React Query hooks
├── package.json ← Root workspace config
├── pnpm-workspace.yaml ← Workspace definitions + version catalog
└── tsconfig.json ← TypeScript project references

---
## Tech Stack
| Layer | Technology | Why |
|---|---|---|
| Frontend | React 19 + Vite 7 + TypeScript | Fast DX, RSC-ready |
| Styling | Tailwind CSS v4 + Glassmorphism | Custom design system |
| Auth | Privy (email + MPC embedded wallet) | Non-custodial, frictionless |
| Backend | Node.js + Express 5 + TypeScript | Lightweight, typed |
| Database | PostgreSQL + Drizzle ORM | Type-safe, fast migrations |
| Blockchain | BNB Smart Chain (BSC) Mainnet | Low fees, fast blocks |
| API Contract | Zod schemas | Shared between FE and BE |
| Package Manager | pnpm workspaces | Monorepo with catalogs |
---
## Frontend (`artifacts/jinos`)
Built with React + Vite. Routing via `wouter` (lightweight, 2KB).
### Pages
| Route | Component | Description |
|---|---|---|
| `/` | `dashboard.tsx` | Main overview: balance, market, activity |
| `/wallet` | `wallet.tsx` | BSC wallet: address, tokens, export |
| `/card` | `card.tsx` | Virtual card CRUD |
| `/portfolio` | `portfolio.tsx` | Chart + asset allocation |
| `/transactions` | `transactions.tsx` | On-chain history with filters |
| `/swap` | `swap.tsx` | DEX swap (coming soon) |
| `/agent` | `agent.tsx` | AI chat + task config |
| `/docs` | `docs.tsx` | In-app documentation |
### Design System
- **Primary accent:** `#C8F000` (lime-yellow) — used for all highlights, active states, glows
- **Background:** `#030A03` (deep dark green)
- **Font:** `GeistMono` (monospaced) for all text
- **Glass cards:** `backdrop-blur-xl bg-black/30 border border-[rgba(200,240,0,0.15)] rounded-2xl`
- **NEVER use:** cyan, purple, blue, or orange as accent colors
### Auth Context
`AuthContext.tsx` wraps Privy auth and:
1. On login: calls `POST /api/users/sync` with `privyId`, `email`, `embeddedWalletAddress`
2. Sets `setAuthTokenGetter(() => privyId)` — all API requests automatically include `Authorization: Bearer <privyId>`
3. Exposes `walletAddress`, `jinosUser`, `login`, `logout` to all pages
---
## Backend (`artifacts/api-server`)
Node.js + Express 5 REST API. Auth middleware reads the `Authorization: Bearer <privyId>` header and sets `req.userId`.
### Endpoints
| Method | Path | Description |
|---|---|---|
| GET | `/api/health` | Health check |
| POST | `/api/users/sync` | Upsert user from Privy auth |
| GET | `/api/dashboard/summary` | Balance, gas, block, top movers |
| GET | `/api/dashboard/activity` | Recent card transactions |
| GET | `/api/dashboard/market` | Live CoinGecko market data |
| GET | `/api/wallet` | Wallet info + live token balances |
| GET | `/api/cards` | List user's virtual cards |
| POST | `/api/cards` | Issue a new virtual card |
| GET | `/api/cards/:id` | Get card detail + transactions |
| PATCH | `/api/cards/:id` | Update card (freeze/unfreeze, limit) |
| POST | `/api/cards/:id/fund` | Fund card from wallet |
| GET | `/api/transactions` | On-chain transaction history |
| GET | `/api/portfolio` | Portfolio chart + asset allocation |
| POST | `/api/agent/chat` | Send message to AI agent |
| GET | `/api/agent/history` | Chat history |
| GET | `/api/agent/config` | Agent task configuration |
| PUT | `/api/agent/config` | Update agent task config |
| GET | `/api/swap` | Swap info (coming soon) |
### BSC Integration (`lib/bsc.ts`)
Direct JSON-RPC calls to BSC public nodes — no third-party SDK dependency:
- **Endpoints used:** `bsc-dataseed1/2/3.binance.org`
- **Methods:** `eth_getBalance`, `eth_call` (ERC-20 balanceOf), `eth_blockNumber`, `eth_gasPrice`
- **Failover:** Tries all 3 endpoints sequentially, returns 0 on total failure
- **Caching:** Wallet balances cached 30s, prices cached 60s (CoinGecko)
### Price Data
Live prices fetched from CoinGecko free API:
- Symbols tracked: BNB, USDT, USDC, CAKE, XVS, BTC, ETH
- Cache TTL: 60 seconds
- Fallback: returns last cached data on API failure
---
## Database (`lib/db`)
PostgreSQL with Drizzle ORM.
### Tables
| Table | Description |
|---|---|
| `users` | Privy user records: `privyId`, `email`, `embeddedWalletAddress` |
| `wallets` | BSC wallet records: `address`, `chainId`, `network` |
| `virtual_cards` | Card records: `cardNumber`, `expiry`, `cvv`, `balance`, `status` |
| `card_agent_configs` | AI task configs per card: `taskId`, `enabled`, `config` JSON |
| `card_transactions` | Card activity: fund, purchase, transfer, withdraw |
| `transactions` | On-chain tx history: `hash`, `type`, `amount`, `amountUsd` |
| `agent_messages` | AI chat history: `sessionId`, `role`, `message` |
### Schema Conventions
- All IDs: `serial` (auto-increment integer)
- All timestamps: `timestamp with timezone`, default `now()`
- Decimal amounts: `numeric(18, 6)` or `numeric(28, 8)` for precision
- No ORM-level relations — joins done manually in routes for flexibility
---
## API URL Pattern
Critical convention used across the entire codebase:
```typescript
// In frontend files (AuthContext.tsx, card.tsx, etc.)
const API_BASE = (import.meta.env.VITE_API_URL ?? "").replace(/\/+$/, "")
// Dev: VITE_API_URL="" → API_BASE="" → fetch("/api/...") → Vite proxy → backend
// Prod: VITE_API_URL="https://jinos.xyz" → API_BASE="https://jinos.xyz" → fetch("https://jinos.xyz/api/...")

Vite proxy config (dev only):

server: {
  proxy: {
    "/api": {
      target: `http://localhost:${API_PORT}`,
      changeOrigin: true,
    }
  }
}

Authentication Flow
User enters email
      ↓
Privy sends OTP to email
      ↓
User enters OTP → Privy authenticates
      ↓
Privy creates embedded BSC wallet (if user has none)
      ↓
Frontend gets user.id (privyId) from usePrivy()
      ↓
Frontend calls POST /api/users/sync with:
  - Authorization: Bearer <privyId>
  - body: { privyId, email, embeddedWalletAddress }
      ↓
Backend upserts user in PostgreSQL
      ↓
All subsequent API calls include:
  Authorization: Bearer <privyId>
      ↓
Backend middleware: req.userId = privyId
All DB queries: WHERE user_id = req.userId
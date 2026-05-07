# mpp-test-sdk

[![npm](https://img.shields.io/npm/v/mpp-test-sdk)](https://www.npmjs.com/package/mpp-test-sdk)
[![Node.js](https://img.shields.io/node/v/mpp-test-sdk)](https://nodejs.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-zinc.svg)](LICENSE)

SDK for building and testing HTTP 402 pay-per-request APIs on Solana. The client intercepts 402 responses and resolves them automatically — wallet creation, faucet funding, on-chain payment, and retry with proof. The server is a single Express middleware line.

No accounts. No config. No real money on devnet/testnet.

**[Live playground →](https://mpptestkit.com/playground)**

---

## Contents

- [How it works](#how-it-works)
- [Packages](#packages)
- [Quick start](#quick-start)
- [Client API](#client-api)
- [Server API](#server-api)
- [Error handling](#error-handling)
- [Protocol reference](#protocol-reference)
- [Networks](#networks)
- [Development](#development)

---

## How it works

HTTP 402 Payment Required has existed since 1999. This SDK makes it real — on Solana.

```
Client                                    Server
  │                                          │
  │   GET /api/data                          │
  │─────────────────────────────────────────▶│
  │                                          │
  │   402 Payment Required                   │
  │   Payment-Request: solana;               │
  │     amount="0.001";                      │
  │     recipient="9WzDX...AWWM";            │
  │     network="devnet"                     │
  │◀─────────────────────────────────────────│
  │                                          │
  │   [SDK generates ephemeral keypair]      │
  │   [SDK airdrops SOL from faucet]         │
  │   [SDK submits Solana transaction]       │
  │                                          │
  │   GET /api/data                          │
  │   Payment-Receipt: solana;               │
  │     signature="3xKm7...";               │
  │     network="devnet"                     │
  │─────────────────────────────────────────▶│
  │                                          │
  │   [Server verifies tx on-chain]          │
  │                                          │
  │   200 OK                                 │
  │   { "data": "premium content" }          │
  │◀─────────────────────────────────────────│
```

The full flow — wallet, faucet, payment, retry — happens inside `mppFetch`. Your application code doesn't change.

---

## Packages

This is an npm workspace monorepo.

| Package | Description | Published |
|---|---|---|
| [`packages/sdk`](packages/sdk) | Client and server SDK | yes — `mpp-test-sdk` |
| [`packages/example-server`](packages/example-server) | Express server with free and paid endpoints | no |
| [`packages/playground`](packages/playground) | Next.js interactive demo and docs | no |

---

## Quick start

**Requirements:** Node.js 18+

```bash
git clone https://github.com/mpptestkit/mpptestkit.git
cd mpptestkit
npm install
npm run dev
```

Opens the playground at `http://localhost:3000` and starts the example server at `http://localhost:3001`.

### Install the SDK only

```bash
npm install mpp-test-sdk
```

---

## Client API

### `mppFetch(url, init?)`

Drop-in replacement for `fetch`. Uses a shared client instance that is lazily created on first call. Automatically handles the full 402 → pay → retry cycle.

```ts
import { mppFetch } from "mpp-test-sdk";

const res = await mppFetch("https://api.example.com/data");
const data = await res.json();

// What happened automatically:
// 1. SDK generated a Solana keypair
// 2. Airdropped 2 SOL from the devnet faucet
// 3. Server returned 402 Payment Required
// 4. SDK sent 0.001 SOL on Solana (~800ms)
// 5. Retried with Payment-Receipt header → 200 OK
```

The shared client persists its wallet across calls. Call `mppFetch.reset()` to discard it and generate a new wallet on the next request.

```ts
mppFetch.reset();
```

### `createTestClient(config?)`

Creates a client with its own isolated wallet and keypair.

```ts
import { createTestClient } from "mpp-test-sdk";

const client = await createTestClient({
  network: "devnet",
  onStep: (step) => console.log(step.type, step.message),
  timeout: 60_000,
});

const res = await client.fetch("https://api.example.com/data");
```

**Config options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `network` | `"devnet" \| "testnet" \| "mainnet"` | `"devnet"` | Solana network to use |
| `secretKey` | `Uint8Array \| number[]` | auto-generated | Reuse a pre-funded keypair across restarts (required for mainnet) |
| `onStep` | `(step: PaymentStep) => void` | — | Lifecycle callback — fires for each stage of the payment flow |
| `timeout` | `number` | `30000` | Full flow timeout in ms (wallet + payment + retry) |
| `maxRetries` | `number` | `1` | Max retry attempts on transient failure |

**Returns:** `Promise<TestClient>`

```ts
interface TestClient {
  address: string;           // Solana public key (base58)
  network: SolanaNetwork;
  fetch: (url: string, init?: RequestInit) => Promise<Response>;
}
```

**Throws:**
- `MppFaucetError` — faucet unreachable (devnet/testnet only)
- `MppNetworkError` — mainnet used without providing `secretKey`

### `PaymentStep` events

The `onStep` callback receives structured events throughout the payment lifecycle:

| `step.type` | When |
|---|---|
| `"wallet-created"` | New ephemeral keypair generated |
| `"funded"` | Faucet airdrop confirmed (devnet/testnet) |
| `"request"` | Outgoing HTTP request fired |
| `"payment"` | Solana transaction submitted and confirmed |
| `"retry"` | Request retried with Payment-Receipt header |
| `"success"` | Final 200 response received |
| `"error"` | Flow failed |

```ts
const client = await createTestClient({
  onStep: ({ type, message, data }) => {
    if (type === "payment") console.log("tx:", data?.signature);
    if (type === "funded")  console.log("wallet:", data?.address);
  },
});
```

---

## Server API

### `createTestServer(config?)`

Creates an Express-compatible middleware factory that enforces payment on any route.

```ts
import express from "express";
import { createTestServer } from "mpp-test-sdk";

const app = express();
const mpp = createTestServer(); // auto-generates server wallet

// Free — no middleware
app.get("/api/ping", (req, res) => res.json({ ok: true }));

// Paid — one line of middleware
app.get("/api/data", mpp.charge({ amount: "0.001" }), (req, res) => {
  res.json({ data: "premium content" });
});

app.listen(3001);
console.log("Recipient wallet:", mpp.recipientAddress);
```

**Config options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `network` | `"devnet" \| "testnet" \| "mainnet"` | `"devnet"` | Solana network to verify payments on |
| `secretKey` | `Uint8Array \| number[]` | auto-generated | Server wallet keypair (persist this to keep the same recipient address across restarts) |

**Properties:**

| Property | Type | Description |
|---|---|---|
| `mpp.recipientAddress` | `string` | Server wallet public key — payments go here |
| `mpp.network` | `SolanaNetwork` | Active network |

### `mpp.charge(opts)`

Returns an Express `RequestHandler`. Place it before your route handler.

```ts
mpp.charge({ amount: "0.001" })
```

| Option | Type | Description |
|---|---|---|
| `amount` | `string` | Amount in SOL, e.g. `"0.001"` |

**Flow when a request arrives without a valid receipt:**

1. Middleware returns `402 Payment Required` with a `Payment-Request` header
2. Client pays on Solana and retries with `Payment-Receipt` header
3. Middleware verifies the transaction on-chain — confirms recipient and amount
4. If valid, calls `next()` — your route handler runs normally
5. If invalid or missing, returns `403 Forbidden`

---

## Error handling

The SDK exports typed error classes for precise catch handling.

```ts
import {
  MppError,
  MppFaucetError,
  MppPaymentError,
  MppTimeoutError,
  MppNetworkError,
} from "mpp-test-sdk";

try {
  const res = await mppFetch("https://api.example.com/data");
} catch (err) {
  if (err instanceof MppNetworkError) {
    // Mainnet used without secretKey — no auto-funding available
  } else if (err instanceof MppFaucetError) {
    // Devnet/testnet faucet unreachable — err.address is the wallet that failed to fund
  } else if (err instanceof MppPaymentError) {
    // On-chain payment rejected — err.status, err.url
  } else if (err instanceof MppTimeoutError) {
    // Full flow exceeded timeout — err.url, err.timeoutMs
  } else if (err instanceof MppError) {
    // Base class for all SDK errors
  }
}
```

**Error reference:**

| Class | Properties | Thrown when |
|---|---|---|
| `MppNetworkError` | `network` | Mainnet requested but no `secretKey` provided |
| `MppFaucetError` | `address`, `cause?` | Devnet/testnet faucet is down or returns an error |
| `MppPaymentError` | `status`, `url`, `cause?` | Server returns non-200/non-402, or payment is rejected |
| `MppTimeoutError` | `url`, `timeoutMs` | Full flow exceeds `timeout` |

**Tip:** Pass a pre-funded `secretKey` to `createTestClient` to skip faucet calls entirely — useful in CI or when the devnet faucet is rate-limiting.

---

## Protocol reference

### Headers

**`Payment-Request`** — sent by the server on 402:

```
Payment-Request: solana;
  amount="0.001";
  recipient="9WzDX...AWWM";
  network="devnet"
```

**`Payment-Receipt`** — sent by the client on retry:

```
Payment-Receipt: solana;
  signature="3xKm7...";
  network="devnet"
```

### HTTP status codes

| Status | Meaning |
|---|---|
| `402 Payment Required` | No receipt provided — client should pay and retry |
| `200 OK` | Payment verified — response contains the resource |
| `403 Forbidden` | Receipt present but invalid, wrong recipient, or insufficient amount |

### On-chain verification

The server verifies:
1. Transaction exists and is confirmed on the specified Solana network
2. The recipient in `accountKeys` matches `mpp.recipientAddress`
3. The SOL delta on the recipient account is ≥ the required `amount`

---

## Networks

| | devnet | testnet | mainnet |
|---|---|---|---|
| Faucet | Automatic (2 SOL) | Automatic (2 SOL) | Bring your own funded wallet |
| Cost | Free | Free | Real SOL |
| RPC | `api.devnet.solana.com` | `api.testnet.solana.com` | `api.mainnet-beta.solana.com` |
| Use for | Local dev, CI | Pre-production | Production |

Switch networks with one config flag:

```ts
// Development
const client = await createTestClient({ network: "devnet" });

// Production
const client = await createTestClient({
  network: "mainnet",
  secretKey: Uint8Array.from(JSON.parse(process.env.SOLANA_SECRET_KEY!)),
});
```

---

## Development

```bash
# Install all workspace dependencies
npm install

# Build the SDK (CJS + ESM + .d.ts)
npm run build:sdk

# Run tests
npm run test
npm run test:watch

# Lint + format
npm run lint
npm run format
```

### Running locally

```bash
# Runs example-server (port 3001) + playground (port 3000) concurrently
npm run dev
```

### Running tests

```bash
# From repo root
npm run test --workspace=packages/sdk

# Watch mode
npm run test:watch --workspace=packages/sdk
```

Tests use Vitest. All three test suites run real Solana interactions against mocked RPC — no live network calls, no faucet rate limits, sub-second CI.

### Monorepo layout

```
mpp-test-sdk/
├── package.json                  # npm workspaces root
├── packages/
│   ├── sdk/
│   │   ├── src/
│   │   │   ├── index.ts          # public exports
│   │   │   ├── client.ts         # createTestClient, mppFetch
│   │   │   ├── server.ts         # createTestServer, mpp.charge()
│   │   │   └── errors.ts         # MppError subclasses
│   │   └── src/__tests__/
│   │       ├── client.test.ts    # 28 client tests
│   │       ├── server.test.ts    # 18 server tests
│   │       └── integration.test.ts # 8 end-to-end tests
│   ├── example-server/
│   │   └── src/server.ts         # Express app — free + paid endpoints
│   └── playground/
│       └── src/                  # Next.js — interactive demo + docs
```

---

Contract Address: DFvGJ7rSxYdBkDzdSPEA8VbaxdHW7gB5t2C8xGLEpump

## License

MIT

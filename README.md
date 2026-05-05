# mpp-test-sdk

[![npm](https://img.shields.io/npm/v/mpp-test-sdk)](https://www.npmjs.com/package/mpp-test-sdk)
[![Node.js](https://img.shields.io/node/v/mpp-test-sdk)](https://nodejs.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-zinc.svg)](LICENSE)

SDK for building and testing HTTP 402 pay-per-request APIs on [Tempo testnet](https://tempo.xyz). The client intercepts 402 responses and resolves them automatically — wallet creation, faucet funding, on-chain payment, and retry with proof. The server is a single Express middleware line.

No accounts. No config. No real money.

---

## Contents

- [How it works](#how-it-works)
- [Packages](#packages)
- [Quick start](#quick-start)
- [Client API](#client-api)
- [Server API](#server-api)
- [Error handling](#error-handling)
- [Protocol reference](#protocol-reference)
- [Network](#network)
- [Development](#development)

---

## How it works

HTTP 402 Payment Required has existed since 1999. This SDK makes it useful.

```
Client                                    Server
  │                                          │
  │   GET /api/data                          │
  │─────────────────────────────────────────▶│
  │                                          │
  │   402 Payment Required                   │
  │   Payment-Request: tempo;                │
  │     amount="0.01";                       │
  │     currency="PathUSD";                  │
  │     network="42431";                     │
  │     recipient="0x..."                    │
  │◀─────────────────────────────────────────│
  │                                          │
  │   [SDK creates wallet if needed]         │
  │   [SDK funds from Tempo faucet]          │
  │   [SDK submits on-chain payment]         │
  │                                          │
  │   GET /api/data                          │
  │   Payment-Receipt: tempo;                │
  │     reference="0x<txhash>";              │
  │     network="42431";                     │
  │     amount="0.01"                        │
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

**Requirements:** Node.js 22+

```bash
git clone https://github.com/anthropics/mpp-test-sdk.git
cd mpp-test-sdk
npm install
cp .env.example .env
# Add your MPP_SECRET_KEY to .env
npm run dev
```

Opens the playground at `http://localhost:5173` and starts the example server at `http://localhost:3001`.

### Install the SDK only

```bash
npm install mpp-test-sdk
```

---

## Client API

### `mppFetch(url, init?)`

Drop-in replacement for `fetch`. Uses a shared client instance that is lazily created on first call.

```ts
import { mppFetch } from "mpp-test-sdk";

const res = await mppFetch("http://localhost:3001/api/data");
const data = await res.json();
```

The shared client persists its wallet across calls. Call `mppFetch.reset()` to discard it and generate a new wallet on the next request.

```ts
mppFetch.reset();
```

### `createTestClient(config?)`

Creates a client with its own isolated wallet.

```ts
import { createTestClient } from "mpp-test-sdk";

const client = await createTestClient({
  onStep: (step) => console.log(step.type, step.message),
  timeout: 60_000,
});

const res = await client.fetch("http://localhost:3001/api/data");
```

**Config options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `privateKey` | `` `0x${string}` `` | auto-generated | Reuse a pre-funded wallet across restarts |
| `onStep` | `(step: PaymentStep) => void` | — | Lifecycle callback — fires for each stage of the payment flow |
| `timeout` | `number` | `30000` | Full flow timeout in ms (wallet + payment + retry) |
| `maxRetries` | `number` | `1` | Max retry attempts on transient failure |

**Returns:** `Promise<TestClient>` — resolves after the wallet is created and funded.

```ts
interface TestClient {
  address: string;
  method: "tempo";
  fetch: (url: string, init?: RequestInit) => Promise<Response>;
}
```

**Throws:** `MppFaucetError` if the testnet faucet is unreachable.

### `PaymentStep` events

The `onStep` callback receives structured events throughout the payment lifecycle:

| `step.type` | When |
|---|---|
| `"wallet-created"` | New ephemeral wallet generated |
| `"funded"` | Faucet funding confirmed |
| `"request"` | Outgoing HTTP request |
| `"payment"` | On-chain payment submitted |
| `"success"` | Final 200 response received |
| `"error"` | Flow failed |

```ts
const client = await createTestClient({
  onStep: ({ type, message, data }) => {
    if (type === "payment") console.log("tx:", data?.txHash);
  },
});
```

---

## Server API

### `createTestServer(config)`

Creates Express middleware that enforces payment on any route.

```ts
import express from "express";
import { createTestServer } from "mpp-test-sdk";

const app = express();
const mpp = createTestServer({ secretKey: process.env.MPP_SECRET_KEY });

// Free — no middleware
app.get("/api/ping", (req, res) => res.json({ ok: true }));

// Paid — one line
app.get("/api/data", mpp.charge({ amount: "0.01" }), (req, res) => {
  res.json({ data: "premium content" });
});

app.listen(3001);
```

**Config options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `secretKey` | `string` | **required** | MPP secret key for payment verification |
| `privateKey` | `` `0x${string}` `` | auto-generated | Server wallet private key |
| `currency` | `` `0x${string}` `` | PathUSD (`0x20c0...`) | ERC-20 token address to accept |

**Throws:** `Error` synchronously if `secretKey` is missing or empty.

### `mpp.charge(opts)`

Returns an Express `RequestHandler`. Place it before your route handler.

```ts
mpp.charge({ amount: "0.05" })
```

| Option | Type | Description |
|---|---|---|
| `amount` | `string` | Amount in PathUSD units, e.g. `"0.01"` |

**Flow when a request arrives without a valid receipt:**

1. Middleware returns `402 Payment Required` with a `Payment-Request` header
2. Client pays on-chain and retries with `Payment-Receipt` header
3. Middleware verifies the transaction on-chain
4. If valid, calls `next()` — your handler runs normally
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
} from "mpp-test-sdk";

try {
  const res = await client.fetch("http://localhost:3001/api/data");
} catch (err) {
  if (err instanceof MppFaucetError) {
    // Testnet faucet unreachable — err.address is the wallet that failed to fund
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
| `MppFaucetError` | `address`, `cause?` | Faucet is down or returns an error |
| `MppPaymentError` | `status`, `url`, `cause?` | Server returns non-200/non-402, or payment is rejected |
| `MppTimeoutError` | `url`, `timeoutMs` | Full flow exceeds `timeout` |

**Tip:** Faucet errors are almost always transient. Pass a pre-funded `privateKey` to `createTestClient` during local development to skip faucet calls entirely.

---

## Protocol reference

### Request headers

**`Payment-Request`** — sent by the server on 402:

```
Payment-Request: tempo;
  amount="0.01";
  currency="PathUSD";
  network="42431";
  recipient="0x<server-wallet-address>"
```

**`Payment-Receipt`** — sent by the client on retry:

```
Payment-Receipt: tempo;
  reference="0x<transaction-hash>";
  network="42431";
  amount="0.01"
```

### HTTP status codes

| Status | Meaning |
|---|---|
| `402 Payment Required` | No receipt provided — client should pay and retry |
| `200 OK` | Payment verified — response contains the resource |
| `403 Forbidden` | Receipt present but invalid or insufficient amount |
| `408 Request Timeout` | Payment verification timed out on the server |
| `503 Service Unavailable` | Payment network unreachable from the server |

---

## Network

All payments use **PathUSD** test tokens on **Tempo Moderato Testnet**. Transactions are real — they appear on-chain — but the tokens have no financial value.

| | Testnet | Mainnet |
|---|---|---|
| Name | Tempo Moderato | Tempo |
| Chain ID | `42431` | `42430` |
| RPC | `https://rpc.testnet.tempo.xyz` | `https://rpc.tempo.xyz` |
| Explorer | `https://explorer.testnet.tempo.xyz` | `https://explorer.tempo.xyz` |
| PathUSD address | `0x20c0000000000000000000000000000000000000` | — |
| Faucet | Automatic via SDK | N/A |

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
# Runs example-server (port 3001) + playground (port 5173) concurrently
npm run dev
```

The playground connects to the example server by default. To point it at your own server, update the host and port in the playground UI.

### Environment

```bash
# packages/example-server/.env
MPP_SECRET_KEY=your_secret_key_here
PORT=3001                            # optional, default 3001
```

### Monorepo layout

```
mpp-test-sdk/
├── package.json                  # npm workspaces root
├── packages/
│   ├── sdk/
│   │   ├── src/
│   │   │   ├── index.ts          # public exports
│   │   │   ├── client.ts         # createTestClient, mppFetch
│   │   │   ├── server.ts         # createTestServer
│   │   │   └── errors.ts         # MppError subclasses
│   │   └── __tests__/
│   │       ├── client.test.ts
│   │       └── server.test.ts
│   ├── example-server/
│   │   └── src/server.ts         # Express app with free + paid endpoints
│   └── playground/
│       └── src/                  # Next.js interactive demo
```

---

## License

MIT

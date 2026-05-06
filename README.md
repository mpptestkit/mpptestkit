# mpp-test-sdk

[![npm](https://img.shields.io/npm/v/mpp-test-sdk)](https://www.npmjs.com/package/mpp-test-sdk)
[![Node.js](https://img.shields.io/node/v/mpp-test-sdk)](https://nodejs.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-zinc.svg)](LICENSE)

Test pay-per-request APIs on **Solana** — devnet, testnet, or mainnet. Auto-creates wallets, airdrops SOL, handles HTTP 402 MPP payments. Zero setup required.

**[mpptestkit.com](https://mpptestkit.com)** · [Playground](https://mpptestkit.com/playground) · [Docs](https://mpptestkit.com/docs) · [GitHub](https://github.com/mpptestkit/mpp-test-sdk) · [X](https://x.com/mpptestkit)

---

## Install

```bash
npm i mpp-test-sdk
```

Requires Node.js 18+.

---

## Client

```ts
import { mppFetch } from "mpp-test-sdk";

// One line. No wallet setup. No config.
const res = await mppFetch("https://your-api.com/api/data");
const data = await res.json();

// What happened automatically:
// 1. SDK generated a Solana keypair
// 2. Airdropped 2 SOL from the devnet faucet
// 3. Server returned 402 Payment Required
// 4. SDK sent 0.001 SOL on Solana (~800ms finality)
// 5. Retried with Payment-Receipt header → 200 OK
```

Or with a dedicated client instance:

```ts
import { createTestClient } from "mpp-test-sdk";

const client = await createTestClient({
  network: "devnet",
  onStep: (step) => console.log(step.type, step.message),
});

const res = await client.fetch("https://your-api.com/api/data");
```

Call `mppFetch.reset()` to discard the shared client and generate a fresh wallet on the next request.

---

## Server

```ts
import express from "express";
import { createTestServer } from "mpp-test-sdk";

const app = express();
const mpp = createTestServer(); // auto-generates server wallet

// Free — no middleware
app.get("/api/ping", (req, res) => res.json({ ok: true }));

// Paid — one line (0.001 SOL)
app.get("/api/data",
  mpp.charge({ amount: "0.001" }),
  (req, res) => res.json({ data: "premium content" })
);

app.listen(3001);
console.log("Payments go to:", mpp.recipientAddress);
```

---

## API Reference

### `mppFetch(url, init?)`

Drop-in replacement for `fetch`. Lazily creates a shared client on first call and reuses it across requests.

### `createTestClient(config?)`

Creates a client with its own isolated Solana wallet.

| Option | Type | Default | Description |
|---|---|---|---|
| `network` | `"devnet" \| "testnet" \| "mainnet"` | `"devnet"` | Solana network to use |
| `secretKey` | `Uint8Array \| number[]` | auto-generated | Reuse a pre-funded keypair (required on mainnet) |
| `onStep` | `(step: PaymentStep) => void` | — | Lifecycle event callback |
| `timeout` | `number` | `30000` | Full flow timeout in ms |

Returns `Promise<TestClient>`:

```ts
interface TestClient {
  address: string;        // Solana public key (base58)
  network: SolanaNetwork;
  fetch: (url: string, init?: RequestInit) => Promise<Response>;
}
```

**Throws:**
- `MppFaucetError` — devnet/testnet airdrop failed
- `MppNetworkError` — mainnet used without a `secretKey`

### `createTestServer(config?)`

Creates Express middleware that enforces payment on any route.

| Option | Type | Default | Description |
|---|---|---|---|
| `network` | `"devnet" \| "testnet" \| "mainnet"` | `"devnet"` | Solana network to verify payments on |
| `secretKey` | `Uint8Array \| number[]` | auto-generated | Server wallet keypair |

**Properties:**

| Property | Description |
|---|---|
| `mpp.recipientAddress` | Server wallet public key — payments land here |
| `mpp.network` | Active network |

### `mpp.charge(opts)`

Express `RequestHandler`. Place before your route handler.

| Option | Type | Description |
|---|---|---|
| `amount` | `string` | Amount in SOL, e.g. `"0.001"` |

### `PaymentStep` events

| `step.type` | When |
|---|---|
| `"wallet-created"` | Solana keypair generated |
| `"funded"` | Faucet airdrop confirmed (devnet/testnet) |
| `"request"` | Outgoing HTTP request fired |
| `"payment"` | SOL transaction submitted and confirmed |
| `"retry"` | Request retried with Payment-Receipt header |
| `"success"` | Final 200 response received |
| `"error"` | Flow failed |

---

## Error Handling

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
    // Mainnet used without secretKey
  } else if (err instanceof MppFaucetError) {
    // Devnet/testnet airdrop failed — err.address
  } else if (err instanceof MppPaymentError) {
    // Server rejected payment — err.status, err.url
  } else if (err instanceof MppTimeoutError) {
    // Flow timed out — err.url, err.timeoutMs
  }
}
```

**Tip:** Pass a pre-funded `secretKey` to skip faucet calls entirely — useful in CI or when devnet is rate-limiting.

---

## Protocol

The SDK implements **MPP (Machine Payments Protocol)** on Solana. Plain HTTP headers, no custom transport:

```
# Server → Client (402 Payment Required)
Payment-Request: solana; amount="0.001"; recipient="9WzDX..."; network="devnet"

# Client → Server (retry with proof)
Payment-Receipt: solana; signature="3xKm7..."; network="devnet"
```

On-chain verification confirms: transaction exists, recipient matches, SOL delta ≥ required amount.

---

## Networks

| | devnet | testnet | mainnet |
|---|---|---|---|
| Faucet | Auto (2 SOL) | Auto (2 SOL) | Bring your own |
| Cost | Free | Free | Real SOL |
| Use for | Local dev, CI | Pre-production | Production |

```ts
// Devnet (default)
const client = await createTestClient({ network: "devnet" });

// Mainnet — bring your own funded keypair
const client = await createTestClient({
  network: "mainnet",
  secretKey: Uint8Array.from(JSON.parse(process.env.SOLANA_SECRET_KEY!)),
});
```

---

## Troubleshooting

**`MppFaucetError`** — Devnet airdrop is rate-limited. Wait 30–60 seconds and retry, or pass a pre-funded `secretKey` to skip the faucet.

**`MppNetworkError`** — You passed `network: "mainnet"` without a `secretKey`. Mainnet has no faucet — provide a funded keypair.

**`MppTimeoutError`** — Increase `timeout` in `createTestClient`. Solana devnet confirmations typically take 1–3 seconds.

**402 not handled** — Ensure `mpp.charge()` middleware is placed before the route handler on the server.

---

## License

MIT

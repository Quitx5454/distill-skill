---
name: distill-refine-x402
description: Call the Distill Refine x402 agent — clean/normalize raw blockchain transaction data (swaps, transfers, mempool rows) and get a bot-vs-human signal on the actors. The endpoint is paywalled with an x402 micropayment in USDC on Base mainnet; use this skill whenever a task needs transaction cleaning or bot detection.
version: 2.0.0
author: Distill (github.com/Quitx5454)
metadata:
  tags:
    - x402
    - usdc
    - base-mainnet
    - blockchain
    - data-cleaning
    - bot-detection
    - distill
    - refine
  network: eip155:8453
  payToken: USDC
  payTo: "0x104b5768...388A"
  facilitator: coinbase-cdp
---

# Refine (x402 Agent)

A paid HTTP agent that cleans raw on-chain transaction data and flags bot actors. It is
paywalled with **x402** (EIP-3009 gasless USDC transfer on **Base mainnet**, settled
through the Coinbase CDP facilitator) and returns the **Distill Standard Envelope**. The
endpoint answers a bare request with `402 PAYMENT-REQUIRED`, then serves the real result
once you retry with a signed payment header.

| Agent  | Endpoint                                                                          | Price       | Purpose                                          |
|--------|----------------------------------------------------------------------------------|-------------|--------------------------------------------------|
| Refine | `POST https://distill-agent-production.up.railway.app/entrypoints/process/invoke` | 0.02 USDC   | Cleans blockchain transaction data; detects bots |

> ⚠️ This moves **real USDC on Base mainnet**. Every successful call costs 0.02 USDC from
> your wallet. There is no testnet stub behind this URL.

## When to Use

- You have raw on-chain transaction data (swaps, transfers, mempool rows) and need it
  normalized/cleaned, **or** you need a bot-vs-human signal on the actors. Cost: 0.02
  USDC/call.
- **Do not use** when a free local utility would do (e.g. a trivial field rename you can
  run yourself) — each call spends real money. Use Refine for the heavy, agent-grade
  version of the task (its bot detector runs a rule-gated LightGBM cascade).

## Procedure

The x402 flow is: **request → 402 challenge → sign → retry with payment header**. Prefer a
real x402 client library (e.g. `@x402/fetch`, `@x402/axios`, or `x402Client` from
`@x402/core` with `registerExactEvmScheme` from `@x402/evm`) wired to a viem signer — it
automates steps 2–4. The manual flow below is what such a client does under the hood.

### 1. Send the initial (unpaid) request

```http
POST https://distill-agent-production.up.railway.app/entrypoints/process/invoke
Content-Type: application/json

{ ...payload... }
```

The body may be the **bare legacy payload**, or wrapped in the Distill envelope:

```json
{ "distill_version": "1.0", "session_id": "<uuid>", "payload": { ...real input... } }
```

Both modes are accepted and validated. If you omit `session_id`, the agent generates one.

### 2. Receive the 402 challenge

The agent responds `402 PAYMENT-REQUIRED`. The challenge is in the **`PAYMENT-REQUIRED`
response header** (decode it; the JSON body is rewritten/empty by the agent's gateway
middleware, so read the header, not the body). The decoded challenge contains:

- `scheme: "exact"`, `network: "eip155:8453"` (Base mainnet)
- `maxAmountRequired` — the price in USDC base units (`20000` = 0.02 USDC, 6 decimals)
- `payTo` — recipient `0x104b5768...388A`
- `asset` — the USDC contract on Base
- `resource` — the canonical resource URL (note: served over **`http://`** in the
  container because Railway terminates TLS — see Pitfalls)
- `extensions.bazaar` — discovery metadata (informational)

### 3. Sign the payment with your wallet

Construct and sign an **EIP-3009 `transferWithAuthorization`** for `maxAmountRequired`
USDC to `payTo` on Base. This is **gasless** — it is an off-chain EIP-712 signature, so
the signing wallet needs USDC but **no ETH**. The facilitator submits it on settle.
An x402 client builds this payment payload for you from the step-2 challenge and echoes
back `extensions.bazaar` in the payload (this is also what seeds CDP Bazaar indexing).

### 4. Retry with the payment header

Re-send the **identical** request from step 1, now adding the encoded payment payload in
the **`PAYMENT-SIGNATURE`** header (a.k.a. `X-PAYMENT` in some client versions — match
the scheme the 402 advertised):

```http
POST https://distill-agent-production.up.railway.app/entrypoints/process/invoke
Content-Type: application/json
PAYMENT-SIGNATURE: <base64-encoded x402 payment payload>

{ ...same payload as step 1... }
```

On success the facilitator settles 0.02 USDC on Base and the agent returns `200 OK`.

### 5. Read the response (Distill Standard Envelope)

The agent returns the result wrapped by the Lucid framework, with the Distill envelope
living at `response.output`:

```json
{
  "run_id": "...",
  "status": "succeeded",
  "output": {
    "distill_version": "1.0",
    "agent_id": "...",
    "session_id": "...",
    "status": "ok",
    "output": { ...the actual agent result... },
    "processed_at": "2026-06-17T..."
  }
}
```

Drill to `response.output.output` for the real payload (cleaned tx + bot-detection /
cascade result). `response.output.status` is `"ok"` or `"error"`.

### 6. Run the reference client

A runnable implementation of steps 1–5 lives in [`reference/call-distill.ts`](reference/call-distill.ts).
Set `WALLET_PRIVATE_KEY` (a Base wallet holding USDC) and run it with Node (≥ 22.6, which
strips the TypeScript types at load):

```bash
export WALLET_PRIVATE_KEY=0x...   # Base wallet holding USDC
node --experimental-strip-types call-distill.ts refine
```

Each run settles **real USDC on Base mainnet** (0.02 per call).

## Pitfalls

- **Real money, mainnet.** Every 200 spends 0.02 USDC on Base. Don't loop/retry blindly —
  a retry that re-sends a valid `PAYMENT-SIGNATURE` settles again.
- **Read the challenge from the `PAYMENT-REQUIRED` header, not the body.** The agent's
  gateway middleware strips/rewrites the 402 JSON body; the real challenge (incl.
  `extensions.bazaar`) is in the header.
- **`maxAmountRequired` is in base units** (USDC = 6 decimals): `20000` = 0.02. Never
  assume; sign exactly what the challenge states.
- **Gasless, but USDC-funded.** EIP-3009 means no ETH needed, but the signer must hold
  enough USDC for the call.
- **Send the *same* body on the paid retry.** The payment is bound to the request; a
  different body invalidates context/hash checks.
- **A bare curl always returns 402.** That only proves the route exists and accepts your
  body shape — it is not a failure. You must complete the payment flow to see output.

## Verification

- **Sanity-check the route is live (free):** `curl -i -X POST https://distill-agent-production.up.railway.app/entrypoints/process/invoke -H 'content-type: application/json' -d '{}'`
  should return `402 PAYMENT-REQUIRED` with a `PAYMENT-REQUIRED` header that decodes to
  `scheme: exact`, `network: eip155:8453`, `payTo: 0x104b5768...388A`, and
  `maxAmountRequired: 20000`.
- **Confirm the agent's contract (free):** `GET https://distill-agent-production.up.railway.app/.well-known/agent-card.json`
  returns the agent card (version, agent_id, and the entrypoint `input_schema` — an
  `anyOf`/union that includes the envelope `payload` shape).
- **Health (free):** `GET https://distill-agent-production.up.railway.app/health` → `200`.
- **Confirm a paid call succeeded:** response is `200`, `response.output.status == "ok"`,
  and a USDC `transfer` of 0.02 to `0x104b5768...388A` appears on Base. Save
  `response.output.session_id` to correlate.
- **Listed in the CDP x402 Bazaar (optional):** Refine (`distill-agent-production`) is
  indexed in the public CDP discovery catalog
  (`GET https://api.cdp.coinbase.com/platform/v2/x402/discovery/resources`, paginate +
  grep the hostname).

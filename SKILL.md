---
name: distill-suite-x402
description: Call the three Distill x402 agents — Refine (clean blockchain tx data + bot detection), Shield (PII redaction + payment-context sanitization), and Trace (parse agent execution logs into structured JSON). Each endpoint is paywalled with an x402 micropayment in USDC on Base mainnet; use this skill whenever a task needs transaction cleaning, sensitive-data sanitization, or log parsing.
version: 1.0.0
author: Distill (github.com/Quitx5454)
metadata:
  tags:
    - x402
    - usdc
    - base-mainnet
    - blockchain
    - data-cleaning
    - bot-detection
    - pii-redaction
    - log-parsing
    - distill
  network: eip155:8453
  payToken: USDC
  payTo: "0x104b5768...388A"
  facilitator: coinbase-cdp
---

# Distill Suite (x402 Agents)

Three paid HTTP agents that share one payment scheme (x402, EIP-3009 gasless USDC
transfer on **Base mainnet**, settled through the Coinbase CDP facilitator) and one
response shape (the **Distill Standard Envelope**). Each endpoint answers a bare
request with `402 PAYMENT-REQUIRED`, then serves the real result once you retry with
a signed payment header.

| Agent  | Endpoint                                                                          | Price       | Purpose                                                        |
|--------|----------------------------------------------------------------------------------|-------------|---------------------------------------------------------------|
| Refine | `POST https://distill-agent-production.up.railway.app/entrypoints/process/invoke` | 0.02 USDC   | Cleans blockchain transaction data; detects bots              |
| Shield | `POST https://shield-agent-v2-production.up.railway.app/entrypoints/shield/invoke`| 0.005 USDC  | PII redaction; payment-context sanitization                   |
| Trace  | `POST https://trace-agent-production.up.railway.app/entrypoints/trace/invoke`     | 0.01 USDC   | Parses agent execution logs into structured JSON              |

> ⚠️ These move **real USDC on Base mainnet**. Every successful call costs the listed
> amount from your wallet. There is no testnet stub behind these URLs.

## When to Use

- **Refine** — you have raw on-chain transaction data (swaps, transfers, mempool rows)
  and need it normalized/cleaned, or you need a bot-vs-human signal on the actors. Cost:
  0.02 USDC/call.
- **Shield** — you are about to log, forward, or display a payment request / arbitrary
  payload and must strip PII (emails, phones, SSNs, IBANs, ETH addresses, API keys/JWTs)
  and sanitize payment context first. Cost: 0.005 USDC/call.
- **Trace** — you have unstructured agent execution logs (free-text run output) and need
  them parsed into structured JSON for analysis or replay. Cost: 0.01 USDC/call.
- **Do not use** when a free local utility would do (e.g. a trivial regex redaction you
  can run yourself) — each call spends real money. Use these for the heavy, agent-grade
  versions of the task.

## Procedure

The x402 flow is the same for all three agents: **request → 402 challenge → sign →
retry with payment header**. Prefer a real x402 client library (e.g. `@x402/fetch`,
`@x402/axios`, or `x402Client` from `@x402/core` with `registerExactEvmScheme` from
`@x402/evm`) wired to a viem signer — it automates steps 2–4. The manual flow below is
what such a client does under the hood.

### 1. Send the initial (unpaid) request

```http
POST <endpoint>
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
- `maxAmountRequired` — the price in USDC base units (e.g. `20000` = 0.02 USDC, 6 decimals)
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
POST <endpoint>
Content-Type: application/json
PAYMENT-SIGNATURE: <base64-encoded x402 payment payload>

{ ...same payload as step 1... }
```

On success the facilitator settles 0.005–0.02 USDC on Base and the agent returns
`200 OK`.

### 5. (Shield only) add the second authorization header

Shield runs a 5-layer sanitization chain and its **context-verify** layer needs a
*second*, separate EIP-712 signature in the **`X-Shield-Authorization`** header (Shield's
own `PaymentAuthorization`, EIP-712 domain `Shield_x402_Middleware` v2.0, chainId 8453).
This is **distinct from** the x402 `PAYMENT-SIGNATURE` micropayment:

- `PAYMENT-SIGNATURE` = the x402 micropayment that pays **to use** Shield.
- `X-Shield-Authorization` = the EIP-712 auth that Shield's nonce-lock/context-verify
  layers decode and validate.

The `resourceHash` preimage inside `X-Shield-Authorization` is
`keccak256(url | method | bodyHash)`, and **`url` must use the `http://` scheme** (not
`https://`) — Railway terminates TLS so the container sees `http`. Sign with `https://`
and you get a `403` *after* the paywall already charged you. Without this header at all,
Shield returns `401 "Missing X-Shield-Authorization header"` (fires before the handler, so
no USDC is spent, but you also get no result).

### 6. Read the response (Distill Standard Envelope)

Every agent returns the result wrapped by the Lucid framework, with the Distill envelope
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

Drill to `response.output.output` for the real payload (cleaned tx / sanitized request /
structured log JSON). `response.output.status` is `"ok"` or `"error"`.

### 7. Run the reference client

A runnable implementation of steps 1–6 lives in [`reference/call-distill.ts`](reference/call-distill.ts).
Set `WALLET_PRIVATE_KEY` (a Base wallet holding USDC) and run it with Node (≥ 22.6, which
strips the TypeScript types at load):

```bash
export WALLET_PRIVATE_KEY=0x...   # Base wallet holding USDC
node --experimental-strip-types call-distill.ts refine   # or: trace | shield
```

Each run settles **real USDC on Base mainnet** (0.005–0.02 per call).

## Pitfalls

- **Real money, mainnet.** Every 200 spends USDC on Base. Don't loop/retry blindly —
  a retry that re-sends a valid `PAYMENT-SIGNATURE` settles again.
- **Read the challenge from the `PAYMENT-REQUIRED` header, not the body.** The agents'
  gateway middleware strips/rewrites the 402 JSON body; the real challenge (incl.
  `extensions.bazaar`) is in the header.
- **Shield needs TWO signatures** (`PAYMENT-SIGNATURE` + `X-Shield-Authorization`). The
  other two agents need only `PAYMENT-SIGNATURE`. See step 5.
- **Shield's `http://` preimage gotcha.** The `X-Shield-Authorization` `resourceHash`
  must hash the `http://` resource URL. `https://` → `403` after you've already paid.
- **Shield base URL redirects.** Post to `/entrypoints/shield/invoke`, not the bare host
  (the root path 301-redirects). All three real endpoints are the `/entrypoints/*/invoke`
  routes in the table above.
- **`maxAmountRequired` is in base units** (USDC = 6 decimals): `5000` = 0.005,
  `10000` = 0.01, `20000` = 0.02. Never assume; sign exactly what the challenge states.
- **Gasless, but USDC-funded.** EIP-3009 means no ETH needed, but the signer must hold
  enough USDC for the call.
- **Send the *same* body on the paid retry.** The payment is bound to the request; a
  different body invalidates context/hash checks (and on Shield, the `resourceHash`).
- **A bare curl always returns 402.** That only proves the route exists and accepts your
  body shape — it is not a failure. You must complete the payment flow to see output.

## Verification

- **Sanity-check the route is live (free):** `curl -i -X POST <endpoint> -H 'content-type:
  application/json' -d '{}'` should return `402 PAYMENT-REQUIRED` with a `PAYMENT-REQUIRED`
  header that decodes to `scheme: exact`, `network: eip155:8453`, `payTo: 0x104b5768...388A`,
  and the expected `maxAmountRequired` (5000 / 10000 / 20000).
- **Confirm the agent's contract (free):** `GET <host>/.well-known/agent-card.json`
  returns the agent card (version, agent_id, and the entrypoint `input_schema` — an
  `anyOf`/union that includes the envelope `payload` shape).
- **Health (free):** `GET <host>/health` → `200`.
- **Confirm a paid call succeeded:** response is `200`, `response.output.status == "ok"`,
  and a USDC `transfer` to `0x104b5768...388A` appears on Base for the amount you signed
  (0.005 / 0.01 / 0.02). Save `response.output.session_id` to correlate.
- **Listed in the CDP x402 Bazaar (optional):** Refine (`distill-agent-production`) and
  Trace (`trace-agent-production`) are indexed in the public CDP discovery catalog
  (`GET https://api.cdp.coinbase.com/platform/v2/x402/discovery/resources`, paginate +
  grep the hostname). Shield's settle path differs, so it may not appear there.

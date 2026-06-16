# Distill x402 reference client

A runnable companion to [`../SKILL.md`](../SKILL.md). Shows the full x402 flow
(request → 402 → sign → retry) against the three Distill agents, with the
Shield-specific `X-Shield-Authorization` EIP-712 header built by hand.

> ⚠️ **Real USDC on Base mainnet.** Each successful call settles 0.005–0.02 USDC
> from `WALLET_PRIVATE_KEY`. Gasless (EIP-3009): the wallet needs USDC, no ETH.

## Setup

```bash
cd reference
npm install                      # install deps
export WALLET_PRIVATE_KEY=0x...  # Base wallet holding USDC
```

> Requires Node ≥ 22.6 for `--experimental-strip-types` (runs the `.ts` file directly).

## Run

```bash
node --experimental-strip-types call-distill.ts refine   # 0.02  USDC — clean tx data + bot detection
node --experimental-strip-types call-distill.ts trace    # 0.01  USDC — parse execution logs to JSON
node --experimental-strip-types call-distill.ts shield   # 0.005 USDC — PII redaction + sanitization
```

## What it shows

- `wrapFetchWithPayment` (from `@x402/fetch`) wired to an `x402Client` +
  `registerExactEvmScheme` automates the x402 loop: it catches the `402`, reads the
  challenge from the `PAYMENT-REQUIRED` header, signs the EIP-3009 USDC transfer,
  and retries with the `PAYMENT-SIGNATURE` header. This mirrors the production
  client in `shield-agent-v2/test-live.ts` (`buildPayFetch`).
- For **Shield only**, `buildShieldAuthHeader()` constructs the extra EIP-712
  `X-Shield-Authorization` header. The struct is reconciled against
  `shield-agent-v2/src/lib/types.ts` and verified by the passing paid E2E test:
  - EIP-712 domain `Shield_x402_Middleware` v2.0, `chainId 8453`,
    `verifyingContract 0x0…0`.
  - `PaymentAuthorization` fields: `amount`, `nonce`, `deadline`, `payerAddress`,
    `resourceHash` (all `uint256`/`address`/`bytes32`).
  - `resourceHash = keccak256(toBytes(`${url}|POST|${bodyHash}`))`, where
    `bodyHash = keccak256(toBytes(body))` and **`url` uses `http://`** (Railway
    terminates TLS → the container's `c.req.url` is `http`). Signing `https://`
    → `403` *after* you've already paid the x402 toll.
  - wire payload is base64-encoded JSON with all BigInts stringified.
- Response parsing drills through the Distill envelope:
  `response.output.output` holds the actual agent result.

> If Shield rejects post-payment: `403` = `resourceHash` mismatch (check the
> `http://` scheme and that the body sent is byte-identical to the body hashed);
> `401` = bad/missing signature or missing `X-Shield-Authorization` header.

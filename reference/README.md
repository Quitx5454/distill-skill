# Distill Refine x402 reference client

A runnable companion to [`../SKILL.md`](../SKILL.md). Shows the full x402 flow
(request → 402 → sign → retry) against the Refine agent.

> ⚠️ **Real USDC on Base mainnet.** Each successful call settles 0.02 USDC
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
node --experimental-strip-types call-distill.ts refine   # 0.02 USDC — clean tx data + bot detection
```

## What it shows

- `wrapFetchWithPayment` (from `@x402/fetch`) wired to an `x402Client` +
  `registerExactEvmScheme` automates the x402 loop: it catches the `402`, reads the
  challenge from the `PAYMENT-REQUIRED` header, signs the EIP-3009 USDC transfer,
  and retries with the `PAYMENT-SIGNATURE` header.
- The call sends the sample transaction batch to Refine, then prints the cleaned
  summary plus only the cascade rows flagged as bots.
- Response parsing drills through the Distill envelope:
  `response.output.output` holds the actual agent result.

---
name: fetch
description: Read-only verification of token claims on Arc (chain 5042) before trading. Use when asked to verify, audit, or "fetch" a token's receipts, when evaluating a token before a trade, or before trusting any claim about supply, ownership, dev holdings, or liquidity.
---

# fetch

Verify a token's claims against chain state before trading it. Execution skills place the trade; this skill checks what you are trading. Read-only, zero dependencies, no wallet or authentication required.

## Basic checks (any Arc token)

All checks are `eth_call` against a public RPC (`https://rpc.mainnet.arc.io`, chain 5042, CORS open, rate limited):

```json
{"jsonrpc":"2.0","id":1,"method":"eth_call","params":[{"to":"<token>","data":"<calldata>"},"latest"]}
```

1. **Contract exists**: `eth_getCode` on the address. Empty `0x` means nothing is deployed there.
2. **Identity and supply**: `name()` `0x06fdde03`, `symbol()` `0x95d89b41`, `decimals()` `0x313ce567`, `totalSupply()` `0x18160ddd`.
3. **Ownership**: `owner()` `0x8da5cb5b`. Zero address or `0x...dEaD` means renounced; a live address means an admin can still touch the contract; a revert means not Ownable (neither good nor bad on its own).
4. **Burned supply**: `balanceOf(0x...dEaD)` and `balanceOf(0x0)` (`0x70a08231` + 32-byte-padded address), as a share of totalSupply.

Report results per check. PASS on these checks is not an endorsement and says nothing about the pool, holders, or price.

## Published receipts (fetchable tokens)

Some tokens publish machine-readable receipts: claims paired with the exact `eth_call` and expected result, per the fetchable spec (https://github.com/mdogai/fetchable).

1. Look up the contract address in the registry: https://raw.githubusercontent.com/mdogai/fetchable/main/registry.json
2. If listed, fetch the token's `fetch.json` and run every claim: POST the claim's `{to, data}` as an `eth_call`, compare the result to `expect` case-insensitively. Match = PASS, mismatch = FAIL (the published claim is false), transport error = retry once then report ERROR.
3. Never trust `expect` as fact. The document proves nothing by existing; only the chain state does. Run the calls.

Reference implementation: https://mdog.ai/fetch.json (live checker at https://mdog.ai).

## Rules

- Read-only. This skill never signs, sends, or approves anything.
- Report PASS as "published claims match chain state", never as "safe" or a trade recommendation.
- Public RPCs serve current state only (pruned history); all proofs read `latest`.
- Beware decoy pairs on aggregators; if a token's receipts name a canonical pair, treat every other pair as noise.

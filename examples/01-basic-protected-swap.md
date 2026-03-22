# Example: Basic Protected Swap

**Scenario:** An agent wants to swap 1000 USDC for WETH and enforce a minimum receive amount.

> **Note:** Token addresses in this example are illustrative mainnet values, shown as already provided by the user. In practice, the agent requests explicit addresses before proceeding if symbols alone are given — see [Example C in 00-trigger-conversations.md](00-trigger-conversations.md).

---

## Conversation

**User:**
> Swap 1000 USDC for WETH on mainnet. I want at least 0.49 WETH and I'm ok spending up to 1000 USDC.

**Agent (step 1 — parse and confirm):**

Parsed protections:
- `max_spend(USDC, 1000)` → ΔUSDC ≥ -1000
- `min_receive(WETH, 0.49)` → ΔWETH ≥ +0.49

> I translated your intent into these enforceable protections:
>
>   max spend:    1000 USDC  →  ΔUSDC ≥ -1000
>   min receive:  0.49 WETH  →  ΔWETH ≥ +0.49
>
> Confirm to proceed with protected execution.

**User:** Confirmed.

**Agent (step 2 — prepare full bundle):**

The Uniswap `actionTx` is unsigned at this point. Calls `prepare_protected_transaction` with the confirmed `protectionIntent` — the unsigned calldata stays with the agent, not passed to the MCP.

```
prepare_protected_transaction({
  userAddress: "0xSigner...",
  protectionIntent: {
    protections: [
      { type: "max_spend",   token: { symbol: "USDC", address: "0xA0b8..." }, amount: "1000" },
      { type: "min_receive", token: { symbol: "WETH", address: "0xC02a..." }, amount: "0.49" }
    ]
  }
})
```

Response:
```json
{
  "summary": { "protections": ["max spend: 1000 USDC", "min receive: 0.49 WETH"], ... },
  "preTx":  { "nonce": 42, ... },
  "postTx": { "nonce": 44, ... },
  "nonceLayout": { "pre": 42, "action": 43, "post": 44 }
}
```

**Agent (step 3 — assign nonce + sign all three):**

Assigns nonce 43 to the upstream Uniswap `userTx`. Signs `preTx`, `userTx`, and `postTx` — all with the same key (`userAddress`).

**Agent (step 4 — submit bundle):**

```
submit_protected_bundle({
  signedPreTx:  "0x02f8...",
  signedUserTx: "0x02f8...",   // signed actionTx
  signedPostTx: "0x02f8..."
})
```

Response:
```json
{
  "status": "included",
  "txHash": "0xabc123...",
  "blockNumber": 19500005
}
```

**Agent (step 5 — confirm inclusion):**

`submit_protected_bundle` returns inclusion status once the bundle lands.

**Agent to user:**
> Transaction confirmed at block 19500005.
> Protection verified: received 0.512 WETH, spent 998.4 USDC.
> Both constraints satisfied.

---

## Outcome

| Constraint | Required | Actual | Result |
|---|---|---|---|
| USDC outflow | ≤ 1000 | 998.4 | ✓ satisfied |
| WETH inflow | ≥ 0.49 | 0.512 | ✓ satisfied |

Bundle included. Trade executed.

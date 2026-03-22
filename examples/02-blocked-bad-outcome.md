# Example: Blocked — Outcome Violated

**Scenario:** An agent submits a protected swap. Between bundle construction and block inclusion, MEV activity shifts the price. The post-check constraint fails. The bundle is not included on-chain — the user transaction never executes, and no gas is paid.

> **Note:** Token addresses are illustrative mainnet values, shown as already provided by the user. In practice, the agent requests explicit addresses before proceeding — see [Example C in 00-trigger-conversations.md](00-trigger-conversations.md).

---

## Conversation

**User:**
> Swap 1000 USDC for WETH. Don't receive less than 0.49 WETH.

**Agent (steps 1–3 — same as basic flow):**

Parses `min_receive(WETH, 0.49)`, calls `prepare_protected_transaction`, receives `preTx`, `postTx`, and `nonceLayout`. Assigns nonce N+1 to the upstream `actionTx`. Signs all three.

**Agent (step 4 — submit bundle):**

The agent calls `submit_protected_bundle` once with `retryUntilBlock`. The MCP server retries internally across blocks until the bundle is included or the block window expires. The agent does not resubmit.

```
submit_protected_bundle({
  signedPreTx:  "0x02f8...",
  signedUserTx: "0x02f8...",   // signed actionTx
  signedPostTx: "0x02f8...",
  retryUntilBlock: 19500007
})
```

The MCP polls across blocks 19500002–19500007. Market conditions do not recover — the post-check would fail on each attempt. After the block window expires, the MCP returns the final status:

```json
{
  "status": "protected",
  "error": "RETRY_EXHAUSTED",
  "violatedConstraints": [
    {
      "token": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
      "required": { "minInflow": "0.49" },
      "actual":   { "inflow": "0.41" }
    }
  ]
}
```

**Agent to user:**
> All retry attempts exhausted. Market conditions did not meet your constraints within the validity window.
> Would have received 0.41 WETH — below your minimum of 0.49.
> No transaction was submitted. No gas was consumed.
>
> Options:
> - Widen the minimum receive (currently 0.49 WETH)
> - Try again with a new block window
> - Cancel

---

## Outcome

| Constraint | Required | Actual | Result |
|---|---|---|---|
| WETH inflow | ≥ 0.49 | 0.41 | ✗ violated |

Bundle not included on-chain. User transaction never executed. No gas paid.

---

## Why this matters

This is not a failed transaction. This is the enforcement working as designed.

The difference from standard DeFi protection:

| Mechanism | What it does | What it cannot do |
|---|---|---|
| `amountOutMin` in swap | Reverts the swap if slippage exceeds threshold | Does not prevent gas cost on revert |
| IntentGuard post-check | Bundle not included if post-check would fail | Action tx never executes on-chain — no gas paid |
| UI price impact warning | Informs the user | Does not enforce the outcome |

IntentGuard's enforcement happens before the user transaction is broadcast. The outcome condition is checked atomically — if it fails, nothing lands.

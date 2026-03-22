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
  "status": "protected",
  "error": "PROTECTED",
  "violatedConstraints": [
    {
      "token": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
      "required": { "minInflow": "0.49" },
      "actual":   { "inflow": "0.41" }
    }
  ],
  "retriesRemaining": 4
}
```

**Agent to user:**
> Protection triggered at block 19500002.
> Would have received 0.41 WETH — below your minimum of 0.49.
> Transaction was not submitted. No gas consumed.
>
> Retrying in next block...

**Agent (retry — block 19500003):**

Resubmits the same signed bundle.

```
submit_protected_bundle({ ..., retryUntilBlock: 19500007 })
→ { "status": "protected", "error": "PROTECTED", "retriesRemaining": 3 }
```

**Agent to user (after RETRY_EXHAUSTED at block 19500007):**
> All retry attempts exhausted. Market conditions did not meet your constraints within the validity window.
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

Bundle reverted atomically. User transaction never landed. Zero gas consumed.

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

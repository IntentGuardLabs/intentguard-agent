# Example: MetaMask Delegated Execution

**Scenario:** A user has delegated swap authority to an AI agent via MetaMask's delegation framework. The user wants the agent to execute a swap — but only if the economic outcome meets their conditions.

> **Note:** Token addresses are illustrative mainnet values, shown as already provided by the user. In practice, the agent requests explicit addresses before proceeding — see [Example C in 00-trigger-conversations.md](00-trigger-conversations.md).

---

## The distinction

MetaMask Delegation answers: **what is the agent allowed to do?**

IntentGuard answers: **what result is the agent allowed to produce?**

These are two different questions. Delegation without outcome enforcement means the agent is authorized to act, but not constrained on what happens as a result.

---

## Conversation

**User:**
> I've delegated swap permissions to my trading agent. Have it swap up to 1000 USDC for WETH — but don't let the swap go through unless I receive at least 0.49 WETH.

**Agent:**
> Your delegated permissions authorize me to perform the swap.
> I'm layering IntentGuard outcome enforcement so execution only proceeds if:
>
> - USDC spent ≤ 1000
> - WETH received ≥ 0.49
>
> Delegation authorizes the action. IntentGuard constrains the result.
>
> Confirm to proceed with protected execution.

**User:** Confirmed.

**Agent:** Preparing protected bundle...

**Agent (step 1 — parse intent):**

Parsed protections:
- `max_spend(USDC, 1000)` → ΔUSDC ≥ -1000
- `min_receive(WETH, 0.49)` → ΔWETH ≥ +0.49

**Agent (step 2 — prepare full bundle):**

The swap `actionTx` is unsigned at this point. Calls `prepare_protected_transaction` with the protection intent — the unsigned calldata stays with the agent, not passed to the MCP.

```
prepare_protected_transaction({
  userAddress: "0xDelegatedAgent...",
  protectionIntent: { protections: [...] }
})
```

Returns `preTx`, `postTx`, and `nonceLayout: { pre: N, action: N+1, post: N+2 }`. The swap calldata (`actionTx`) was built upstream — it is NOT passed to the MCP.

Protection summary presented to user:
```
Protection active:
  max spend:    1000 USDC
  min receive:  0.49 WETH

Execution authority: delegated agent
Outcome enforcement: IntentGuard post-check (trustless)
```

**Agent (step 3 — assign nonce, sign all three, submit):**

Assigns nonce N+1 to the `userTx`. Signs `preTx`, `userTx`, and `postTx`. Calls `submit_protected_bundle` with `signedPreTx`, `signedUserTx`, `signedPostTx`.

---

## Outcome A — execution succeeds

Market conditions are stable. WETH received = 0.503.

```json
{
  "status": "included",
  "txHash": "0xdef456...",
  "blockNumber": 21800004
}
```

**Agent to user:**
> Swap executed at block 21800004.
> Received 0.503 WETH, spent 997.2 USDC.
> All constraints satisfied.

---

## Outcome B — execution blocked

MEV activity shifts price. WETH would be 0.461 — below the constraint.

```json
{
  "status": "protected",
  "error": "PROTECTED",
  "violatedConstraints": [
    { "token": "WETH", "required": { "minInflow": "0.49" }, "actual": { "inflow": "0.461" } }
  ]
}
```

**Agent to user:**
> Execution blocked. Would have received 0.461 WETH — below your minimum of 0.49.
> No swap was executed. No funds moved.
>
> Your delegation remains active. Retry or adjust the minimum receive.

---

## Why this integration matters

Without IntentGuard, the delegation flow looks like:

```
User grants permission → Agent executes → Hope outcome is acceptable
```

With IntentGuard:

```
User grants permission → Agent prepares → IntentGuard enforces outcome → Execute only if acceptable
```

Delegated agents need both:
- **Permission controls** — what the agent may do (MetaMask Delegation)
- **Outcome enforcement** — what result is acceptable (IntentGuard)

Without the second, the agent is authorized but not safe.

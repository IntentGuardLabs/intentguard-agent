# Example: Blocked Token Drain

**Scenario:** A user is about to sign a transaction that is supposed to interact with a contract. They want to ensure specific assets are not touched, regardless of what the calldata actually does.

> **Note:** Token addresses are illustrative mainnet values, shown as already provided by the user. In practice, the agent requests explicit addresses before proceeding — see [Example C in 00-trigger-conversations.md](00-trigger-conversations.md).

---

## The point

IntentGuard does not inspect or classify calldata. It does not detect phishing semantically.

What it does: if the declared assets would move in a way that violates the stated constraints, the bundle is not included on-chain. The user transaction never executes.

This means even if the transaction is malicious, misleading, or simply unexpected in its effects — protected assets remain untouched.

---

## Conversation

**User:**
> Execute this transaction, but my USDC and DAI balances must not decrease.

**Agent:**
> Understood. I'll enforce these outcome constraints:
>
> - USDC balance must not decrease
> - DAI balance must not decrease
>
> If the transaction would cause either balance to drop, it will not be included on-chain.
>
> Preparing protected bundle...

**Agent (step 1 — parse intent):**

Parsed protections:
- `no_balance_decrease(USDC)` → ΔUSDC ≥ 0
- `no_balance_decrease(DAI)` → ΔDAI ≥ 0

**Agent (step 2–4 — prepare and sign bundle):**

Standard bundle preparation. User transaction calldata is unmodified.

---

## Outcome A — transaction is safe

The transaction interacts with the target contract as expected. USDC and DAI balances are unchanged.

```json
{ "status": "included", "txHash": "0xaaa111...", "blockNumber": 20100042 }
```

**Agent to user:**
> Transaction confirmed. USDC and DAI balances unchanged.

---

## Outcome B — transaction would drain assets

The calldata, despite its appearance, includes a transfer of USDC to an external address.

Post-check detects: ΔUSDC = -500 (violated constraint ΔUSDC ≥ 0).

```json
{
  "status": "protected",
  "error": "PROTECTED",
  "violatedConstraints": [
    { "token": "USDC", "required": { "maxOutflow": "0" }, "actual": { "outflow": "500" } }
  ]
}
```

**Agent to user:**
> Execution blocked. The transaction would have moved 500 USDC out of your account.
> Your USDC balance is unchanged. No gas was paid.
>
> This may indicate unexpected contract behavior or a malicious transaction.

---

## What this shows

IntentGuard is not limited to swaps. The same outcome-enforcement primitive applies to any transaction where the user wants to constrain what assets may move.

```
"Execute this, but WETH must not be touched."
"Run this interaction, but my stablecoin balances must not decrease."
"Submit this transaction only if ETH spent is under 0.1."
```

In each case: if the protected assets would move in violation of the declared constraints, the bundle is not included. The user pays no gas.

---

## Important caveat

IntentGuard does not:
- classify calldata as malicious or safe
- inspect contract logic or storage
- detect phishing at the intent level

It enforces: **if the transaction would produce this outcome, do not include it.**

That is a narrower and more precise claim — and it is defensible.

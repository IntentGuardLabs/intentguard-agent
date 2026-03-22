# Example: Uniswap Protected Swap

**Scenario:** An agent executes a Uniswap swap and enforces a minimum receive on the final outcome — regardless of route, pool state, or intermediary steps.

> **Note:** Token addresses are illustrative mainnet values, shown as already provided by the user. In practice, the agent requests explicit addresses before proceeding — see [Example C in 00-trigger-conversations.md](00-trigger-conversations.md).

---

## The point

Uniswap offers route- and swap-level parameters such as minimum output thresholds. IntentGuard adds an independent, account-level outcome guarantee on top — enforcing the net balance change for the user's account regardless of route, pool state, or intermediary steps.

The difference is not about replacing Uniswap's protection. It is about where enforcement happens and what it covers.

---

## Conversation

**User:**
> Swap 10 ETH to USDC on Uniswap. Only execute if I receive at least 24,500 USDC.

**Agent (step 1 — parse intent):**

Parsed protections:
- `max_spend(ETH, 10)` → ΔnativeBalance ≥ -10 ETH
- `min_receive(USDC, 24500)` → ΔUSDC ≥ +24500

**Agent (confirmation prompt):**
> I translated your intent into these enforceable protections:
>
>   max spend:    10 ETH    →  ΔnativeBalance ≥ -10
>   min receive:  24,500 USDC  →  ΔUSDC ≥ +24500
>
> This constraint applies to your account's net balance change — regardless of the swap route or intermediary hops.
>
> Confirm to proceed with protected execution.

**User:** Confirmed.

**Agent (step 2 — prepare full bundle):**

The Uniswap `actionTx` is unsigned at this point. Calls `prepare_protected_transaction` with the protection intent — the unsigned calldata stays with the agent, not passed to the MCP.

```
prepare_protected_transaction({
  userAddress: "0xSigner...",
  protectionIntent: {
    protections: [
      { type: "max_spend",   token: { symbol: "ETH",  address: "0xEeeee..." }, amount: "10" },
      { type: "min_receive", token: { symbol: "USDC", address: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48" }, amount: "24500" }
    ]
  }
})
```

Returns `preTx`, `postTx`, and `nonceLayout: { pre: N, action: N+1, post: N+2 }`.

The Uniswap calldata (`actionTx`) was built upstream. The agent now assigns nonce N+1 to it.

Protection summary:
```
Protection active:
  max spend:    10 ETH
  min receive:  24,500 USDC

Constraint basis: net account balance change
Route: determined by Uniswap — IntentGuard enforces outcome, not path
```

**Agent (step 3 — assign nonce, sign all three, submit):**

Assigns nonce N+1 to the Uniswap `userTx`. Signs `preTx`, `userTx`, and `postTx`. Calls `submit_protected_bundle` with `signedPreTx`, `signedUserTx`, `signedPostTx`.

---

## Outcome A — execution succeeds

Uniswap routes through ETH → WETH → USDC. Final USDC received = 24,847.

```json
{
  "status": "included",
  "txHash": "0xfed789...",
  "blockNumber": 21900005
}
```

**Agent to user:**
> Swap confirmed at block 21900005.
> Received 24,847 USDC for 10 ETH.
> Constraint satisfied.

---

## Outcome B — execution blocked (multi-hop loss)

Route goes ETH → WETH → intermediate token → USDC. Slippage on the second hop brings final USDC to 24,120.

```json
{
  "status": "protected",
  "error": "PROTECTED",
  "violatedConstraints": [
    { "token": "USDC", "required": { "minInflow": "24500" }, "actual": { "inflow": "24120" } }
  ]
}
```

**Agent to user:**
> Execution blocked. Multi-hop route would have returned 24,120 USDC — below your minimum of 24,500.
> No swap executed. Bundle not included on-chain — no gas paid.

---

## Key distinction from native Uniswap protection

| | Uniswap `amountOutMin` | IntentGuard |
|---|---|---|
| What it checks | Router output at time of call | Net account balance after full execution |
| Covers multi-hop losses | Partially (only final router output) | Yes — net balance change captures everything |
| Gas on failed outcome | Charged (tx executes then reverts) | Not charged — bundle not included, no execution |
| Private routing | No | Yes — private mempool, no sandwich exposure |
| Who enforces | Uniswap contract | On-chain IntentGuard enforcer (independent of Uniswap) |

IntentGuard does not replace Uniswap's protection. It adds a protocol-independent outcome guarantee on top of it.

# Trigger Conversations

This file defines the routing contract for intentguard-agent.

IntentGuard Agent should activate when a user expresses:
- a blockchain action to perform
- an economic condition that must hold after execution

Both must be present.

---

## Positive triggers

> **Note:** Token addresses are omitted from these examples for readability. In practice, the agent must request explicit on-chain addresses before constructing a `ProtectionIntent` — see Example C below.

### Example 1 — Protected swap

**User:**
> Swap 1000 USDC for WETH and make sure I receive at least 0.49 WETH.

**Expected extraction:**
```
action: swap USDC → WETH
protections:
  max_spend(USDC, 1000)
  min_receive(WETH, 0.49)
```

**Expected tool path:**
```
prepare_protected_transaction → [sign preTx, userTx, postTx] → submit_protected_bundle
```

**Why this triggers:** The user specifies both an action and an enforceable outcome condition.

---

### Example 2 — Protected submit with balance invariant

**User:**
> Submit this transaction, but my DAI balance must not decrease.

**Expected extraction:**
```
action: submit provided transaction (calldata supplied by user)
protections:
  no_balance_decrease(DAI)
```

**Expected tool path:**
```
prepare_protected_transaction → [sign preTx, userTx, postTx] → submit_protected_bundle
```

**Why this triggers:** The user provides a transaction and an explicit asset-protection condition.

**Note:** DAI token address must be confirmed before proceeding — see Example C.

---

### Example 3 — Minimum receive only

**User:**
> Execute this swap only if I receive at least 24,500 USDC.

**Expected extraction:**
```
action: execute swap (calldata provided by upstream protocol skill or orchestrator)
protections:
  min_receive(USDC, 24500)
```

**Why this triggers:** Economic protection condition on an execution request.

---

### Example 4 — Multi-asset constraint

**User:**
> Protect this transaction: spend at most 2 ETH, receive at least 3800 USDC.

**Expected extraction:**
```
action: execute provided transaction
protections:
  max_spend(ETH, 2)
  min_receive(USDC, 3800)
```

**Why this triggers:** Two protection conditions, both enforceable as balance-change constraints.

---

### Example 5 — Asset must not be touched

**User:**
> Execute this interaction, but WETH must not be touched.

**Expected extraction:**
```
action: execute provided transaction
protections:
  no_balance_decrease(WETH)
```

**Why this triggers:** "Must not be touched" maps to `no_balance_decrease` — a valid and enforceable constraint.

---

## Negative triggers

### Example 1 — Conceptual question

**User:**
> What is IntentGuard?

**Expected behavior:** Do not trigger intentguard-agent. Route to explainer or demo skill.

---

### Example 2 — Quote or simulation request

**User:**
> How much WETH would I get for 1000 USDC right now?

**Expected behavior:** Do not trigger. This is a price inquiry, not a protected execution request.

---

### Example 3 — Plain transfer

**User:**
> Send 100 USDC to Alice.

**Expected behavior:** Do not trigger unless the user adds a protection condition.

---

### Example 4 — Balance check

**User:**
> What is my current USDC balance?

**Expected behavior:** Do not trigger. This is a read-only query.

---

## Ambiguous cases

### Example A — Action without explicit constraint

**User:**
> Swap 1000 USDC for WETH.

**Expected behavior:** Do not trigger automatically. Ask the user if they want outcome protection.

Suggested agent response:
> Before executing, would you like to set a minimum receive amount for WETH or a maximum spend limit on USDC? IntentGuard can enforce these constraints so the trade only goes through if the outcome meets your conditions.

---

### Example B — Vague protection condition

**User:**
> Swap USDC for WETH, but make sure it's a good trade.

**Expected behavior:** Ask for clarification. "Good trade" is not a quantified constraint.

Suggested agent response:
> To enforce that, I need a specific condition — for example: "receive at least 0.49 WETH" or "spend no more than 1000 USDC." What threshold would you like to enforce?

---

### Example C — Missing token addresses

**User:**
> Swap 1000 USDC for WETH and make sure I receive at least 0.49 WETH.

**Expected behavior:** Do NOT proceed to protection compilation. Token symbols alone are not sufficient to define enforceable constraints — the agent must have explicit on-chain addresses before constructing a `ProtectionIntent`.

Suggested agent response:
> To construct enforceable constraints, I need the token addresses for USDC and WETH.
>
> Please provide:
> - USDC address (e.g. `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` on mainnet)
> - WETH address (e.g. `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` on mainnet)
>
> Which chain are you on?

**Why:** IntentGuard enforces only explicit, unambiguous constraints. Asset identity must be fully specified before protection is applied. The agent must not infer or guess token addresses from symbols.

---

### Example D — Chain not specified

**User:**
> Protect this swap: spend at most 1000 USDC, receive at least 0.49 WETH. Here are the addresses: USDC = 0xA0b8..., WETH = 0xC02a...

**Expected behavior:** Do NOT proceed if chain is not confirmed. The same symbol and even the same address prefix may exist on multiple chains. Constraints must be unambiguous about which network they apply to.

Suggested agent response:
> Which network are these addresses on? (e.g. Ethereum mainnet, Sepolia, etc.)
> The addresses you provided match mainnet values — please confirm so I can verify the token decimals and construct the correct protection.

**Why:** Token addresses differ across chains. A protection constructed with mainnet addresses submitted on a different network will enforce against the wrong asset or fail silently.

---

### Example E — Conflicting constraints

**User:**
> Execute this USDC→WETH swap. Max USDC spend: 0. Min USDC receive: 500.

**Expected behavior:** Flag the contradiction before proceeding. `max_spend(USDC, 0)` means no USDC may leave the account, but the declared action is a swap where USDC is the input token. The trade cannot execute without spending USDC — the bundle would be blocked unconditionally.

Suggested agent response:
> These constraints appear conflicting: you've specified a maximum USDC spend of 0, but this transaction requires spending USDC to execute the swap. The bundle would be blocked before executing.
>
> Did you mean to constrain a different token, or would you like to adjust the spend limit?

**Why:** Constraint contradictions are only detectable in context of the declared action — the same two constraints on a different action may be valid. Surface contradictions before bundle preparation, not after exhausting retry attempts.

---

### Example F — User cancels after confirmation

**User:**
> [After confirming constraints] Actually, cancel this. I don't want to proceed.

**Expected behavior:** Stop immediately. Do not submit the bundle. Do not reuse the prepared transactions.

Signed transactions from an aborted flow must be treated as invalid and discarded. If the user initiates a new protected transaction, the agent must call `prepare_protected_transaction` again for a fresh bundle.

**Why:** Signed transactions with valid nonces could be submitted independently. The agent must not retain or reuse them across sessions or after cancellation.

---

### Example G — Nonce design constraint and collision recovery

**Design constraint:** The IntentGuard 3-transaction bundle requires three consecutive nonces: N (preTx), N+1 (userTx), N+2 (postTx). The MCP fetches nonce N at `prepare_protected_transaction` time and returns `nonceLayout: { pre: N, action: N+1, post: N+2 }`.

The upstream protocol skill or orchestrator that builds the unsigned `actionTx` MUST NOT pre-set a nonce. The IntentGuard agent assigns `nonceLayout.action` (N+1) to the `actionTx` after `prepare_protected_transaction` returns.

The agent should communicate this to any protocol skill that builds the action transaction:
> This transaction will be wrapped in an IntentGuard protection bundle. Do not pre-set a nonce — the IntentGuard agent will assign nonce N+1 after the MCP returns the nonce layout.

---

**Collision scenario:** Despite the above, if another transaction from `userAddress` is included on-chain between `prepare_protected_transaction` and `submit_protected_bundle`, the nonces shift and the error `NONCE_COLLISION` is returned.

**Error received:**
```json
{ "error": "NONCE_COLLISION" }
```

**Expected behavior:** Do NOT retry the existing bundle. The nonces are permanently invalid.

Suggested agent response:
> A transaction from your account was included on-chain while the bundle was being prepared, which invalidated the nonce assignment.
>
> I'll re-prepare the bundle with fresh nonces now.

Then: re-call `prepare_protected_transaction` with the same `userAddress` and `protectionIntent`. Assign the new `nonceLayout.action` to the `userTx`. Sign the new bundle and submit.

**Why:** The nonce layout (N, N+1, N+2) must match the current on-chain state. Re-preparation is the only valid recovery — the previous bundle cannot be patched.

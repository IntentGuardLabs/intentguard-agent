---
name: intentguard-agent
description: "Use when an AI agent or automated system wants to execute a blockchain transaction subject to enforceable balance-outcome constraints expressed in natural language. Trigger when the user describes a transaction alongside a protection condition like 'don't lose more than X', 'receive at least Y', or 'token Z must not be touched'. This skill orchestrates self-signed transaction protection: parse natural language into structured protections, compile into constraints, build the full 3-transaction signing package, and guide the signing and submission flow. Do NOT trigger for conceptual questions about IntentGuard (use intentguard-demo) or for delegated-mode submission where the server signs on the user's behalf (use intentguard-protection)."
metadata:
  mcp-server: intentguard
---

# IntentGuard Agent — Outcome-Enforced Transaction Protection

IntentGuard protects user intent by enforcing transaction outcomes on-chain.

Users express economic conditions in natural language. The agent translates them into explicit balance constraints, presents them for user verification, and only then prepares protected execution. The transaction is included on-chain only if the resulting state satisfies all declared constraints. If any constraint is violated, the bundle is not included — the action transaction never executes, and no gas is paid.

Recommended for trading agents and automated systems that need execution guarded by enforceable outcome constraints. This skill orchestrates self-signed transaction protection — the agent pays for its own protection, no delegated signer required.

## Core invariant

For any protected transaction:

```
final_state MUST satisfy all declared constraints

If any constraint is violated:
→ the transaction MUST NOT be included on-chain
```

Constraints map directly to net balance changes:

| Protection | Formal constraint |
|---|---|
| `max_spend(token, X)` | `Δbalance(token) ≥ -X` |
| `min_receive(token, Y)` | `Δbalance(token) ≥ +Y` |
| `no_balance_decrease(token)` | `Δbalance(token) ≥ 0` |

This invariant is enforced on-chain by the post-check transaction. IntentGuard infrastructure cannot override it.

## Agent interface

**Input:**
- Natural language protection intent (e.g. "don't lose more than 1000 USDC")
- `userAddress` — the account whose balances are being protected

The `actionTx` is **not** an input to this skill. It is built by an upstream protocol skill or orchestrator and never passed to IntentGuard.

**Output:**
- `summary` — human-readable description of the enforced protections
- `preTx`, `postTx` — two unsigned enforcement transactions, plus `nonceLayout: { pre: N, action: N+1, post: N+2 }`
- The agent assigns nonce N+1 to `actionTx`, then signs all three
- Transaction included on-chain only if all constraints are satisfied after execution
- Bundle not included if any constraint is violated — no gas paid

## Deterministic responsibility split

| Layer | Responsibility | Trust |
|---|---|---|
| Skill (this file) | Natural language → structured `ProtectionIntent` | Untrusted parsing |
| MCP server | Nonce/block resolution + deterministic bundle compilation | Trusted logic |
| Agent / wallet | Transaction signing | Key holder |
| On-chain enforcer | Post-condition verification | Trustless |

The on-chain enforcer is the only component that gates execution. The skill and MCP server are preparation layers — they cannot approve or block inclusion.

## Architectural contract

**Rule 1 — MCP prepares only the protective transactions.**
`prepare_protected_transaction` does not construct or return the user/action transaction. It returns `preTx`, `postTx`, and `nonceLayout`. The action transaction is built and held by the upstream protocol skill or orchestrator.

**Rule 2 — The upstream builder is responsible for nonce assignment.**
After calling `prepare_protected_transaction`, the agent must assign `nonceLayout.action` (N+1) to the action transaction before signing. The MCP reserves this slot but cannot enforce it — correctness depends on the upstream builder using the returned nonce layout.

**Rule 3 — Simulation nonce is provisional.**
An upstream action may be simulated with the latest executable nonce (N) for preflight validation, but protected execution must use the reserved bundle nonce layout. Never submit the simulation transaction unchanged if its nonce does not match `nonceLayout.action` (N+1).

## The 3-transaction bundle

Every self-signed protection consists of exactly three transactions submitted together:

```
preTx      (nonce N)    → snapshots balances before execution
actionTx   (nonce N+1)  → the action transaction, unmodified
postTx     (nonce N+2)  → verifies balance constraints after execution
```

**Invariant:**
- All three transactions must be signed by the same account
- `preTx` and `postTx` nonces are assigned by the MCP. The agent assigns `nonceLayout.action` (N+1) to the `actionTx`
- The bundle must be submitted atomically in order

If any constraint is violated, the bundle is not included in any block. None of the transactions execute on-chain, and the user pays no gas.

**Why no gas is paid:** The bundle is submitted to a private relay (Flashbots-compatible). Builders evaluate the bundle atomically — if the post-check would fail, the bundle is dropped before any on-chain execution. No transaction in the bundle is ever included, so no gas is consumed.

## Lifecycle

```
1. Parse protections + confirm with user or calling agent (this skill)
2. prepare_protected_transaction → returns preTx, postTx, nonceLayout (MCP)
3. Assign nonceLayout.action to actionTx; sign all 3 transactions (agent/wallet)
4. submit_protected_bundle → returns inclusion status (MCP)
```

## Triggering inputs

Trigger when the user expresses **a blockchain action + an economic protection condition**.

Both must be present. An action alone is not enough. A question alone is not enough.

**Trigger when:**
- A transaction or on-chain action is being executed
- The user specifies an economic outcome condition on that action

**Trigger examples:**
- "Swap 1000 USDC for WETH and make sure I receive at least 0.49 WETH"
- "Don't lose more than 100 USDC on this swap"
- "Execute this trade only if my USDC balance doesn't drop below X"
- "Protect this transaction: spend at most 2 ETH, receive at least 3800 USDC"
- "My DAI and USDC balances must not decrease"
- "Prepare a protected swap: send this calldata but only if the outcome satisfies these constraints"

**Agent-to-agent trigger examples (no human in the loop):**
- Orchestrator sends: `{ "action": "protect_and_submit", "actionTx": "0x...", "protections": [{ "type": "min_receive", "token": {...}, "amount": "24500" }] }`
- Trading agent sends: `{ "intent": "execute_swap", "constraints": { "maxSpend": { "USDC": "1000" }, "minReceive": { "WETH": "0.49" } } }`
- Pipeline sends: `{ "userAddress": "0xAgent...", "unsignedTx": "...", "protectionPolicy": "no_balance_decrease(DAI)" }`

In agent-to-agent mode, skip the human confirmation step — the calling agent is the confirming principal. Present the parsed constraints in the response so the orchestrator can log them.

**Do NOT trigger when:**
- The user is asking for explanation or a conceptual question → use `intentguard-demo`
- The user wants delegated-mode submission where the server signs → use `intentguard-protection`
- There is no protection condition to enforce (plain transfer, balance check, price quote, simulation)

**Ambiguous cases — always ask, never proceed:**
- Action stated without a protection condition → ask if the user wants outcome enforcement
- Protection condition is vague ("make sure it's a good trade") → ask for a specific quantified threshold
- Token symbol provided without address → ask for the explicit on-chain address
- Chain not specified → ask which network (mainnet, Sepolia, etc.) to ensure the correct token address
- Constraints are conflicting (e.g. `max_spend(USDC, 0)` + `min_receive(USDC, 500)` — net USDC can't increase if spending is zero) → flag the contradiction, ask user to clarify
- Constraints appear infeasible at current market conditions (e.g. "receive at least 100 WETH for 1000 USDC") → warn the user before preparing the bundle; do not silently exhaust retries

**Expected tool path for any trigger:**
```
prepare_protected_transaction → [sign all 3 txs externally] → submit_protected_bundle
```

See [examples/00-trigger-conversations.md](examples/00-trigger-conversations.md) for positive triggers, negative triggers, and ambiguous cases with expected agent behavior.

## Available tools

| Tool | Purpose |
|---|---|
| `prepare_protected_transaction` | Compile protection intent → unsigned `preTx` + `postTx` + `nonceLayout` |
| `submit_protected_bundle` | Submit 3 signed txs (preTx + actionTx + postTx) as a self-signed bundle |

`get_block_number`, `get_nonce`, and decimal resolution are handled internally by the MCP server. The agent does not need to call them.

Amounts are supplied as human-readable values (e.g. `"1000"` for 1000 USDC, `"0.49"` for 0.49 WETH). The MCP server resolves token decimals automatically.

## Tool contracts

### `prepare_protected_transaction`

**Input:**

```json
{
  "userAddress": "0x...",
  "protectionIntent": {
    "protections": [...]
  },
  "validUntilBlock": 19500010
}
```

`validUntilBlock` is optional — defaults to `currentBlock + 10`.

The `actionTx` is **not** passed to this tool. It is built upstream by the protocol skill or orchestrator and remains outside the MCP boundary. The MCP never inspects or modifies it.

**Behavior:** Fetches current nonce and block internally. Establishes nonce layout N, N+1, N+2. Compiles protection intent to on-chain constraints. Builds two unsigned enforcement transactions.

**Output:**

```json
{
  "summary": {
    "protections": ["max spend: 1000 USDC", "min receive: 0.49 WETH"],
    "signer": "0x...",
    "nonceLayout": { "pre": 42, "action": 43, "post": 44 },
    "validUntilBlock": 19500010,
    "chainId": 1,
    "signingInstructions": "Sign 3 transactions with address 0x... on chain 1: ..."
  },
  "preTx": { "to": "0x...", "data": "0x...", "value": "0x0", "nonce": 42, "chainId": 1 },
  "postTx": { "to": "0x...", "data": "0x...", "value": "0x0", "nonce": 44, "chainId": 1 },
  "nonceLayout": { "pre": 42, "action": 43, "post": 44 }
}
```

**Nonce contract:** The MCP returns `nonceLayout.action = N+1`. The upstream protocol skill or orchestrator must construct the action transaction using this nonce before signing. The MCP does not receive, inspect, or modify the action transaction. Nonce correctness is verified structurally at `submit_protected_bundle` time — sequential nonces (N, N+1, N+2) are required or the bundle is rejected.

**Simulation guidance:** An upstream protocol skill may simulate the action transaction using the current executable nonce (N) to verify that the transaction logic succeeds. This simulation is a preflight check only. For protected execution, the final signed bundle must use the nonce layout returned by `prepare_protected_transaction`. A transaction simulated with nonce N must be rebuilt or re-signed with nonce N+1 before protected submission. Treat the simulation transaction as a validity check — not as the final artifact to sign.

---

### `submit_protected_bundle`

**Input:**

```json
{
  "signedPreTx": "0x02f8...",
  "signedUserTx": "0x02f8...",
  "signedPostTx": "0x02f8...",
  "retryUntilBlock": 19500010
}
```

`retryUntilBlock` is optional — defaults to `currentBlock + 25` if omitted.

**Behavior:** Validates all 3 transactions are EIP-1559, signed by the same address, with sequential nonces (N, N+1, N+2), on the same chain. Submits the bundle to the relay. Retries until `retryUntilBlock`. Returns the final inclusion status directly — no separate receipt polling needed.

---

## Protection intent schema

Natural language maps to a structured `ProtectionIntent`:

```json
{
  "protections": [
    {
      "type": "max_spend",
      "token": { "symbol": "USDC", "address": "0xA0b8..." },
      "amount": "1000"
    },
    {
      "type": "min_receive",
      "token": { "symbol": "WETH", "address": "0xC02a..." },
      "amount": "0.49"
    },
    {
      "type": "no_balance_decrease",
      "token": { "symbol": "DAI", "address": "0x6B17..." }
    }
  ]
}
```

### Natural language → protection type mapping

| User says | Protection type | Constraint |
|---|---|---|
| "don't lose more than X of token A" | `max_spend` | maxOutflow(A) = X |
| "spend at most X of token A" | `max_spend` | maxOutflow(A) = X |
| "receive at least Y of token A" | `min_receive` | minInflow(A) = Y |
| "I expect to get at least Y of token A" | `min_receive` | minInflow(A) = Y |
| "token A must not decrease" | `no_balance_decrease` | maxOutflow(A) = 0 |
| "token A shouldn't be touched" | `no_balance_decrease` | maxOutflow(A) = 0 |
| "0 transfer on token A" | `no_balance_decrease` | maxOutflow(A) = 0 |

### Token reference

Each token MUST include both:
- `symbol` (for readability)
- `address` (for execution)

IntentGuard does not perform token symbol resolution. If a user provides only a token symbol (e.g. "USDC"), the agent MUST request the corresponding token address before constructing the protection intent. This ensures constraints are applied to the correct on-chain asset without ambiguity.

Native ETH is represented using the chain's sentinel address (`0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`).

Amounts are human-readable (e.g. "1000" for 1000 USDC, "0.95" for 0.95 WETH). Decimals are resolved automatically by the server.

## Orchestration sequence

See [e2e_flow.md](e2e_flow.md) for the full sequence diagram.

**Prerequisite:** This skill only needs `userAddress`, `protectionIntent`, and optionally `validUntilBlock` to call `prepare_protected_transaction`. The `actionTx` is an upstream concern — built by a protocol skill or orchestrator before the full bundle is assembled. The agent does NOT ask the user for `actionTx`.

**Nonce window constraint:** Between `prepare_protected_transaction` and `submit_protected_bundle`, no other transaction from `userAddress` should be submitted. The MCP assigns nonces at prepare time. If another transaction from the same address is included on-chain during this window, the bundle nonces become invalid and `NONCE_COLLISION` is returned — requiring a full re-preparation.

### Step 1: Parse and verify with user

Convert the protection request into a structured `ProtectionIntent` using the mapping table above. Token addresses must be explicitly provided — if only a symbol is given, ask for the on-chain address and chain before proceeding.

**Present the parsed constraints to the user or calling agent for confirmation before proceeding.** This is the intent verification step — the constraints to be enforced must be verified before any transaction is signed.

Example:
```
I translated your intent into these enforceable protections:

  max spend:    1000 USDC  →  ΔUSDC ≥ -1000
  min receive:  0.49 WETH  →  ΔWETH ≥ +0.49

Confirm to proceed with protected execution.
```

Only proceed after confirmation.

### Step 2: Prepare the full bundle

Call `prepare_protected_transaction` with:
- `userAddress`: the signer's address
- `protectionIntent`: the structured intent from step 1
- `validUntilBlock` (optional)

The MCP server handles nonce and block resolution internally. The `actionTx` is **not** passed here — it remains outside the MCP boundary.

This returns:
- `summary`: human-readable protection description and signing instructions
- `preTx`: unsigned pre-enforcement transaction (nonce N)
- `postTx`: unsigned post-enforcement transaction (nonce N+2)
- `nonceLayout`: `{ pre: N, action: N+1, post: N+2 }`

### Step 3: Assign nonce + sign all three transactions

1. Take `nonceLayout.action` from the response and assign it as the nonce on the `actionTx`
2. Sign all three transactions with `userAddress`:
   - `preTx` (nonce N — returned by MCP)
   - `actionTx` (nonce N+1 — built upstream, nonce just assigned)
   - `postTx` (nonce N+2 — returned by MCP)

All three must be EIP-1559 (type 2). Signing is a separate concern — in a multi-agent setup, a dedicated Signing Agent handles this step. The IntentGuard Agent does not hold or manage private keys.

### Step 4: Submit bundle

```
Call submit_protected_bundle with:
  - signedPreTx
  - signedUserTx   ← the signed action transaction
  - signedPostTx
  - retryUntilBlock (optional — defaults to currentBlock + 25)
```

### Step 5: Confirm inclusion

`submit_protected_bundle` returns an inclusion status directly. If the bundle is pending, the MCP server polls until the block window expires and returns the final status. The agent does not need to poll separately.

## Error handling

| Error | Action |
|---|---|
| `PROTECTED` | Constraints violated — bundle was not included. Retry with adjusted constraints or new block window. |
| `RETRY_EXHAUSTED` | All retry attempts failed. Ask user to widen constraints or try later. |
| `NONCE_COLLISION` | A transaction from `userAddress` was included between `prepare` and `submit`, shifting nonces. Re-call `prepare_protected_transaction` to get a fresh bundle with correct nonces. |
| `INVALID_CONSTRAINTS` | Malformed constraint fields. Check addresses (0x + 40 hex chars) and amounts (non-negative decimals). |
| `SIGNER_MISMATCH_SELF_SIGNED` | preTx/signedUserTx/postTx are not signed by the same address. Re-sign with consistent key. |
| `MISSING_SIGNED_TX` | One or more txs in the bundle are missing or malformed. |
| `NETWORK_ERROR` | RPC unreachable. Wait and retry. |

## Example flow

User: "Swap 1000 USDC for WETH on mainnet. Don't lose more than 1000 USDC and I want at least 0.49 WETH."

1. Parse → `max_spend(USDC, 1000)` + `min_receive(WETH, 0.49)`
2. Present to user: "max spend: 1000 USDC, min receive: 0.49 WETH — confirm?"
3. Call `prepare_protected_transaction`:
   ```json
   {
     "userAddress": "0xSigner...",
     "protectionIntent": {
       "protections": [
         { "type": "max_spend",   "token": { "symbol": "USDC", "address": "0xA0b8..." }, "amount": "1000" },
         { "type": "min_receive", "token": { "symbol": "WETH", "address": "0xC02a..." }, "amount": "0.49" }
       ]
     }
   }
   ```
4. Receive `preTx`, `postTx`, and `nonceLayout: { pre: N, action: N+1, post: N+2 }`
5. Assign nonce N+1 to the upstream `actionTx` (the Uniswap calldata built earlier)
6. Sign all three (`preTx`, `actionTx`, `postTx`) with `userAddress`
7. Call `submit_protected_bundle` with `signedPreTx`, `signedUserTx`, `signedPostTx` → returns inclusion status

## Limitations (v1)

- Balance-outcome constraints only (maxOutflow, minInflow)
- No approval detection — token approvals granted within a protected transaction persist after execution
- No contract storage invariant checks
- No transaction semantic validation — the compiler does not inspect what the transaction does, only what balance changes to enforce
- Token resolution is the agent's responsibility — the MCP server does not resolve symbols
- EIP-1559 (type 2) transactions only
- **Raw bundle transactions must not be queryable after failure.** If a bundle is dropped (constraints violated, never included on-chain), the raw `preTx`, `actionTx`, and `postTx` must not be retrievable via any API. Exposing them leaks the user's trade calldata, protection thresholds, and nonce layout — even though protection worked as intended. The MCP server must not persist or serve bundle transaction data after a failed or expired submission.

## Agent integration checklist

1. Parse intent → `ProtectionIntent` (explicit token addresses required)
2. Present constraints for confirmation — do not proceed without it
3. Call `prepare_protected_transaction(userAddress, protectionIntent)` — actionTx is NOT passed
4. Assign `nonceLayout.action` (N+1) to the upstream `actionTx`
5. Sign `preTx`, `actionTx`, `postTx` — all with the same key
6. Submit via `submit_protected_bundle(signedPreTx, signedUserTx, signedPostTx)` → returns inclusion status

## IntentGuard as an Agent Service

IntentGuard is designed to be consumed by agents as a protection service, not by humans through a UI.

**Service model:**

```
Trading agent  →  x402 payment  →  IntentGuard MCP  →  protected bundle  →  on-chain enforcer
```

An orchestrator or trading agent calls `prepare_protected_transaction` and `submit_protected_bundle` as service calls. The agent is both the caller and the economic actor. The human principal set the constraints at delegation time — they do not need to confirm each transaction.

**Billing (intended, not yet in v1):** Each protected bundle call is a billable service interaction. The agent pays via [x402](https://x402.org) before the MCP responds with the prepared bundle. If payment is not received, the service returns `402 Payment Required` and the bundle is not prepared. This makes protection cost explicit: the agent pays per execution guarantee, not per outcome.

**What changes in agent-to-agent mode:**

| Property | Human-facing mode | Agent service mode |
|---|---|---|
| Confirmation | User confirms constraints in chat | Calling agent is the confirming principal — skip interactive prompt |
| Payment gate | Not present in v1 | x402 payment check before prepare (intended) |
| Error handling | Explain to user, offer options | Return structured error for orchestrator to handle |
| Constraint source | Natural language parsed in real time | Structured `ProtectionIntent` passed directly |

**Trust model:** The on-chain enforcer is the root of trust. Neither the skill, the MCP server, nor the calling agent can override it. An agent that provides a malformed protection intent gets constraints compiled from what it sent — garbage in, garbage enforced.

---

## Critical rules

- NEVER submit without presenting the protection summary to the user or calling agent first
- If the user cancels or declines at any step, stop immediately — do not reuse prepared bundles or signed transactions
- If `NONCE_COLLISION` is received, re-call `prepare_protected_transaction` for a fresh bundle — do not reuse the previous one
- The MCP does NOT return the actionTx — it returns `preTx`, `postTx`, and `nonceLayout`. Assign `nonceLayout.action` as the nonce on the upstream actionTx before signing
- All 3 transactions must be signed by the same address (`signedUserTx` = the signed action transaction)
- Do NOT override or guess nonces — use the layout returned by `prepare_protected_transaction`
- Token addresses MUST be explicitly provided in `ProtectionIntent` — the agent MUST NOT infer addresses from symbols
- If a token address is missing or ambiguous, the agent MUST ask for clarification before proceeding
- Amounts are human-readable — do NOT convert to base units manually
- **Token approval side effects persist even if the bundle is blocked.** Any ERC-20 `approve()` calls within the protected transaction that execute before the post-check will not be reverted if the bundle is later dropped. This is expected EVM behavior — inform the user if the action transaction includes approvals

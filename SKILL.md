---
name: intentguard-agent
description: "Use when an AI agent or automated system wants to protect a blockchain transaction with balance constraints using natural language. Trigger when the user describes protections like 'don't lose more than X', 'receive at least Y', 'token Z must not be touched', or asks to prepare a self-signed protection bundle. This skill orchestrates the full lifecycle: parse natural language into structured protections, compile into constraints, build unsigned enforcement transactions, and guide the signing and submission flow. Do NOT trigger for conceptual questions about IntentGuard (use intentguard-demo) or for delegated-mode submission (use intentguard-protection)."
metadata:
  mcp-server: intentguard
---

# IntentGuard Agent Protection Skill

IntentGuard ensures that the final balance changes of a transaction match the user's declared constraints. If the resulting state violates those constraints, the transaction is not executed.

Recommended default for real trading agents and autonomous systems. This skill orchestrates self-signed transaction protection — the agent pays for its own protection, no delegated signer required.

## Responsibility split

| Layer | Responsibility | Trust boundary |
|---|---|---|
| Skill (this file) | Natural language → structured `ProtectionIntent` | Untrusted parsing layer |
| MCP server | Deterministic compilation + bundle preparation | Trusted enforcement logic |
| Wallet / agent | Transaction signing | Key holder |
| IntentGuard RPC | Simulation, enforcement, bundle inclusion | Execution infrastructure |

## The 3-transaction bundle

Every self-signed protection consists of exactly three transactions submitted together:

```
pre-check   (nonce N)    → snapshots balances before execution
user tx     (nonce N+1)  → the original transaction, unmodified
post-check  (nonce N+2)  → verifies balance constraints after execution
```

**Invariant:**
- All three transactions must be signed by the same account
- Nonces must be sequential (N, N+1, N+2)
- The bundle must be submitted atomically in order

If any constraint is violated, the entire bundle reverts atomically — no gas is consumed by the user.

## Lifecycle

```
1. Parse protections (this skill)
2. Compile → constraints + unsigned pre/post txs (MCP tool)
3. Sign pre + post externally (agent/wallet)
4. Submit bundle (MCP tool)
5. Poll receipt (MCP tool)
```

## When to use

Trigger when the user or agent expresses a protection intent alongside a transaction:
- "Don't lose more than 100 USDC on this swap"
- "I expect to receive at least 0.5 WETH"
- "My DAI balance must not decrease"
- "Protect this transaction: spend at most 2 ETH, receive at least 3800 USDC"
- "Token X shouldn't be touched"

Do NOT trigger for:
- Conceptual questions about IntentGuard → use `intentguard-demo`
- Delegated-mode submission (server signs) → use `intentguard-protection`
- Plain transfers with no protection requirement

## Available tools

| Tool | Purpose |
|---|---|
| `get_block_number` | Get current block number (for validUntilBlock calculation) |
| `get_nonce` | Get current nonce for an address (for signing pre/post txs) |
| `get_token_decimals` | Get ERC-20 decimals (cached) |
| `prepare_protected_transaction` | Compile protection intent + already-signed user tx (exactly as `eth_sendRawTransaction` expects) → unsigned pre/post txs |
| `submit_protected_bundle` | Submit 3 signed txs (pre + user + post) as a self-signed bundle |
| `get_transaction_receipt` | Poll for transaction inclusion after submission |

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

Each token MUST include both `symbol` (for readability) and `address` (for execution). The agent is responsible for resolving token symbols to addresses. The MCP server does not resolve symbols.

Amounts are human-readable (e.g. "1000" for 1000 USDC, "0.95" for 0.95 WETH). Decimals are resolved automatically by the server.

## Orchestration sequence

### Step 1: Parse natural language

Convert the user's protection request into a structured `ProtectionIntent` using the mapping table above. Resolve token symbols to on-chain addresses.

### Step 2: Get block number and nonce

```
Call get_block_number → currentBlock
Set validUntilBlock = currentBlock + 5 to 10

Call get_nonce for the signer address → N
```

**CRITICAL nonce planning:** Fetch current pending nonce N. Sign the user transaction with nonce N+1, so nonce N remains available for the pre-check transaction and nonce N+2 for the post-check transaction. If the user tx has nonce 0, the bundle cannot be constructed because the pre tx would need nonce -1.

### Step 3: Sign and prepare

1. Sign the user transaction with nonce **N+1** (reserving N for the pre tx)
2. Call `prepare_protected_transaction` with:
   - `protectionIntent`: the structured intent from step 1
   - `signedUserTx`: the signed user transaction from above
   - `validUntilBlock`: from step 2

The tool validates:
- User tx nonce > 0 (pre tx needs nonce N-1)
- User tx chain ID matches the MCP server's chain
- On-chain nonce matches the expected bundle layout (pre nonce == on-chain nonce)

This returns:
- `summary`: human-readable protection description + signing instructions
- `package`: machine-readable bundle with unsigned pre/post txs, suggested gas parameters (from user tx), and nonce ordering

**Present the summary to the user for confirmation before proceeding.**

### Step 4: Sign pre and post transactions

The agent or wallet must sign:
1. **Pre tx** with nonce N (from the package's `unsignedPreTx`) — use the `gasParams` from the package
2. **Post tx** with nonce N+2 (from the package's `unsignedPostTx`) — use the `gasParams` from the package

All three transactions must be:
- Signed by the same address
- EIP-1559 (type 2)
- On the same chain

### Step 5: Submit bundle

```
Call submit_protected_bundle with:
  - signedPreTx: signed pre enforcement tx
  - signedUserTx: the original signed user tx (echoed from step 3)
  - signedPostTx: signed post enforcement tx
  - retryUntilBlock: same as validUntilBlock, or currentBlock + 5-10
```

### Step 6: Poll receipt

```
Call get_transaction_receipt with the returned txHash
Wait 12-15 seconds between polls (one block time)
Up to 5 attempts
```

## Error handling

| Error | Action |
|---|---|
| `PROTECTED` | Constraints violated — tx was NOT submitted. Retry with adjusted constraints or new block window. |
| `RETRY_EXHAUSTED` | All retry attempts failed. Ask user to widen constraints or try later. |
| `INVALID_CONSTRAINTS` | Malformed constraint fields. Check addresses (0x + 40 hex chars) and amounts (non-negative decimals). |
| `SIGNER_MISMATCH_SELF_SIGNED` | Pre/user/post txs are not signed by the same address. Re-sign with consistent key. |
| `MISSING_SIGNED_TX` | One or more txs in the bundle are missing or malformed. |
| `NETWORK_ERROR` | RPC unreachable. Wait and retry. |

## Example flow

User: "Swap 1000 USDC for WETH on mainnet. Don't lose more than 1000 USDC and I want at least 0.49 WETH."

1. Parse → `max_spend(USDC, 1000)` + `min_receive(WETH, 0.49)`
2. Get block number → 19500000, set validUntilBlock = 19500007
3. Get nonce → 42 (on-chain nonce for signer)
4. Sign swap tx with nonce **43** (reserving 42 for pre tx)
5. Call `prepare_protected_transaction`:
   ```json
   {
     "protectionIntent": {
       "protections": [
         {
           "type": "max_spend",
           "token": { "symbol": "USDC", "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48" },
           "amount": "1000"
         },
         {
           "type": "min_receive",
           "token": { "symbol": "WETH", "address": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2" },
           "amount": "0.49"
         }
       ]
     },
     "signedUserTx": "0x02f8...",
     "validUntilBlock": 19500007
   }
   ```
6. Present summary: "max spend: 1000 USDC, min receive: 0.49 WETH"
7. Sign pre tx (nonce 42) and post tx (nonce 44)
8. Call `submit_protected_bundle` with all 3 signed txs
9. Poll `get_transaction_receipt` until included

## Limitations (v1)

- Balance-outcome constraints only (maxOutflow, minInflow)
- No approval detection — token approvals granted within a protected transaction persist after execution
- No contract storage invariant checks
- No transaction trace analysis
- No transaction semantic validation — the compiler does not inspect what the transaction does, only what balance changes to enforce
- Token resolution is the agent's responsibility — the MCP server does not resolve symbols
- EIP-1559 (type 2) transactions only

## Agent integration checklist

1. Fetch current nonce N via `get_nonce`
2. Sign user transaction with nonce **N+1**
3. Call `prepare_protected_transaction` with signed user tx + protection intent
4. Sign pre tx with nonce **N** using gas params from the package
5. Sign post tx with nonce **N+2** using gas params from the package
6. Submit all three via `submit_protected_bundle`
7. Poll `get_transaction_receipt` until included

## Critical rules

- NEVER submit without presenting the protection summary to the user first
- The signed user tx must NOT be modified after preparation — it is echoed back unchanged
- All 3 transactions must be signed by the same address
- Nonces must be sequential: N (pre), N+1 (user), N+2 (post)
- Token resolution is the agent's responsibility — provide both symbol and address
- Amounts are human-readable — do NOT convert to base units manually

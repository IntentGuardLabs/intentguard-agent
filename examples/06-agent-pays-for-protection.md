# Example: Agent Pays for Protection

**Scenario:** A trading agent executes a swap autonomously. No human is in the loop. The agent calls IntentGuard as a service to get outcome-enforced execution — and pays for that service via x402 before the bundle is prepared.

> **Note on v1:** x402 payment is the intended billing model and is shown here as a design illustration. It is not implemented in v1 — protection is currently provided without a payment gate.

---

## The service model

```
Trading agent  →  [x402 payment]  →  IntentGuard MCP  →  preTx + postTx + nonceLayout
                                                        ↓
                                   Agent signs all 3 + submits bundle
                                                        ↓
                                          On-chain enforcer (trustless)
```

The agent is the economic actor, the caller, and the signer. The human principal delegated authority and set risk parameters at setup time — they do not interact with each transaction.

---

## Agent-to-agent flow

**Orchestrator → IntentGuard agent:**

```json
{
  "action": "protect_and_submit",
  "userAddress": "0xTradingAgent...",
  "protections": [
    { "type": "min_receive", "token": { "symbol": "USDC", "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48" }, "amount": "24500" },
    { "type": "max_spend",   "token": { "symbol": "ETH",  "address": "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE" }, "amount": "10" }
  ]
}
```

No natural language. No interactive confirmation. The calling agent is the confirming principal.

---

## Step 1 — Parse and log constraints (no human confirmation)

Parsed protections:
- `max_spend(ETH, 10)` → ΔnativeBalance ≥ -10 ETH
- `min_receive(USDC, 24500)` → ΔUSDC ≥ +24500

In agent-to-agent mode, the constraints are logged and returned to the orchestrator in the response — not presented as a conversational confirmation prompt.

---

## Step 2 — Payment check (x402 — intended, not in v1)

Before calling `prepare_protected_transaction`, the agent sends an x402 payment header:

```
GET /prepare
402 Payment Required
  X-Payment-Required: amount=0.001 ETH, recipient=0xIntentGuard..., reference=bundle-id-xyz
```

Agent pays:

```
POST /prepare
  X-Payment: 0x{signed-payment-tx}
```

Service proceeds. If no valid payment: `402 Payment Required` returned, bundle not prepared, agent retries or escalates.

---

## Step 3 — Prepare full bundle

```json
prepare_protected_transaction({
  "userAddress": "0xTradingAgent...",
  "protectionIntent": {
    "protections": [
      { "type": "max_spend",   "token": { "symbol": "ETH",  "address": "0xEeeee..." }, "amount": "10" },
      { "type": "min_receive", "token": { "symbol": "USDC", "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48" }, "amount": "24500" }
    ]
  }
})
```

Returns `preTx`, `postTx`, and `nonceLayout: { pre: N, action: N+1, post: N+2 }`.

The swap `actionTx` was built upstream. The agent assigns nonce N+1 to it.

---

## Step 4 — Sign and submit

Agent signs all three transactions with its own key:
- `preTx` (nonce N)
- `actionTx` (nonce N+1 — built upstream, nonce just assigned)
- `postTx` (nonce N+2)

```json
submit_protected_bundle({
  "signedPreTx":  "0x02f8...",
  "signedUserTx": "0x02f8...",
  "signedPostTx": "0x02f8..."
})
```

---

## Outcome A — execution succeeds

Market conditions meet the constraints. Final USDC received = 24,847.

```json
{
  "status": "included",
  "txHash": "0xabc123...",
  "blockNumber": 21900010
}
```

**Agent response to orchestrator:**
```json
{
  "status": "executed",
  "constraints": { "satisfied": true },
  "received": { "USDC": "24847" },
  "spent": { "ETH": "9.98" },
  "txHash": "0xabc123...",
  "blockNumber": 21900010
}
```

---

## Outcome B — execution blocked

MEV activity shifts price. Final USDC would be 24,120 — below constraint.

```json
{
  "status": "protected",
  "error": "PROTECTED",
  "violatedConstraints": [
    { "token": "USDC", "required": { "minInflow": "24500" }, "actual": { "inflow": "24120" } }
  ],
  "retriesRemaining": 4
}
```

**Agent response to orchestrator:**
```json
{
  "status": "blocked",
  "reason": "PROTECTED",
  "violatedConstraints": [...],
  "gasConsumed": 0,
  "action": "retrying"
}
```

Agent retries automatically within the block window. After `RETRY_EXHAUSTED`, escalates to the orchestrator for policy decision — widen constraints, delay, or cancel.

---

## What this demonstrates

| Property | Detail |
|---|---|
| No human in the loop | Agent is caller, signer, and economic actor |
| Structured intent | No NL parsing — protection intent passed directly |
| x402 payment gate | Service is paid per execution guarantee (v1: not yet enforced) |
| Structured error handling | Machine-readable responses, no chat UX |
| Escalation path | `RETRY_EXHAUSTED` → orchestrator decides policy |

---

## Why this is a service, not a library

Outcome enforcement requires an independent execution layer — this cannot be implemented purely as a local library inside the agent.

IntentGuard is not a smart contract that the agent calls directly. It is a service that:

1. Resolves the current blockchain state (nonce, block number)
2. Compiles the protection intent into on-chain constraint calldata
3. Constructs the pre- and post-enforcement transactions
4. Submits and monitors the bundle via private relay

The agent does not implement any of this logic. It calls the service, signs what it receives, and submits. This makes IntentGuard composable — any agent with a signing key can obtain outcome-enforced execution without understanding the enforcement internals.

# intentguard-agent

**Transactions only execute if their on-chain outcome matches your declared intent. Otherwise they don't execute at all.**

IntentGuard is an enforcement layer for AI agents. Agents express protection in natural language. IntentGuard compiles it into on-chain constraints, bundles them atomically with the transaction, and submits privately. If any constraint is violated at execution time, the entire bundle reverts — no gas consumed.

---

## The problem

Autonomous agents can construct and sign transactions, but they cannot guarantee outcomes.

Between simulation and block inclusion:
- Liquidity shifts
- MEV bots front-run or sandwich
- Oracle prices move
- Multi-step workflows accumulate slippage that per-protocol checks miss

Existing solutions — slippage params, `amountOutMin`, deadline fields — are per-protocol and per-step. They do not enforce the **aggregate outcome** across the full transaction.

IntentGuard does.

---

## How it works

Every protected transaction becomes a 3-transaction bundle:

```
pre-check  (nonce N)    →  snapshots token balances before execution
user tx    (nonce N+1)  →  the original transaction, unmodified
post-check (nonce N+2)  →  verifies outcome constraints after execution
```

If any constraint is violated, the entire bundle reverts atomically — the user tx never lands.

The bundle is submitted via **private routing** (Flashbots-compatible). Bots cannot sandwich what they cannot see.

---

## What the agent expresses

```
"Swap 1000 USDC for WETH. Don't spend more than 1000 USDC. Receive at least 0.49 WETH."
```

IntentGuard compiles this into:

```json
[
  { "token": "USDC", "maxOutflow": "1000", "minInflow": "0" },
  { "token": "WETH", "maxOutflow": "0",    "minInflow": "0.49" }
]
```

Constraints are **net balance changes** — not per-step, not per-protocol. The full transaction is evaluated against them as a unit.

---

## What happens in each case

**Success:** WETH received ≥ 0.49, USDC spent ≤ 1000 → bundle lands, trade executes.

**Blocked:** MEV bot extracts value, WETH received = 0.41 → post-check fails, bundle reverts atomically, user loses nothing.

---

## Design principles

| Step | Responsibility |
|---|---|
| Parse intent | Skill (this repo) — natural language → structured protections |
| Compile constraints | MCP server — deterministic, auditable |
| Sign | Agent or wallet — IntentGuard never holds keys |
| Enforce | On-chain enforcer contract — infrastructure cannot override |

**IntentGuard infrastructure does not gate execution.** The on-chain post-check contract does. This is the trust boundary that matters.

---

## Architecture note

Current skills are self-contained: each skill builds and submits its own transaction.

The natural direction for composable agent stacks:

```
protocol skill     →  builds the transaction
intentguard-agent  →  derives constraints + submits as protected bundle
enforcer contract  →  verifies outcome on-chain
```

This separates *what to do* from *how to guarantee it*. Any protocol skill can delegate enforcement to IntentGuard without knowing how it works internally.

---

## Using this skill

This skill targets AI agents and automated systems that sign their own transactions.

For delegated signing (server signs on your behalf), see `intentguard-protection`.

### Prerequisites

- Claude Code or compatible agent harness
- `intentguard` MCP server configured
- Agent wallet with signing capability

### Load the skill

```
"Load the skill at https://raw.githubusercontent.com/IntentGuardLabs/intentguard-agent/main/SKILL.md"
```

### Example prompts

```
"Swap 1000 USDC for WETH. Don't lose more than 1000 USDC. Receive at least 0.49 WETH."
"Protect this transaction: spend at most 2 ETH, receive at least 3800 USDC."
"My DAI balance must not decrease. Submit this transaction."
```

See [SKILL.md](./SKILL.md) for the full orchestration flow, tool reference, and error handling.

---

## Limitations (v1)

- Balance-outcome constraints only (maxOutflow, minInflow)
- EIP-1559 (type 2) transactions only — Ethereum mainnet + Sepolia
- Token approval side effects persist on revert (expected EVM behavior)
- Token symbol → address resolution is the agent's responsibility

---

## Built at The Synthesis Hackathon

[The Synthesis](https://synthesis.md) · March 2026

Jean-Loïc Mugnier · [@theFrBrGuy](https://x.com/theFrBrGuy) · [IPS Protocol](https://ipsprotocol.xyz)

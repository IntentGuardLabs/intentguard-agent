# intentguard-agent

**AI agents can be authorized to act. IntentGuard ensures they only execute when the resulting on-chain outcome matches user-defined constraints.**

A transaction can be signed, valid, and confirmed — and still be economically wrong.

IntentGuard enforces a post-condition on transaction execution. Agents express protection in natural language. IntentGuard compiles it into on-chain constraints, bundles them atomically with the transaction, and submits privately. If any constraint is violated at execution time, the entire bundle reverts — no gas consumed.

Authorized execution is not enough. Execution must also be economically correct.

---

## The problem

Autonomous agents can construct and sign transactions, but they cannot guarantee outcomes.

Between simulation and block inclusion:
- Liquidity shifts
- MEV bots front-run or sandwich
- Oracle prices move
- Multi-step workflows accumulate slippage that per-protocol checks miss

Existing solutions — slippage params, `amountOutMin`, deadline fields — are per-protocol and per-step. They do not enforce the **aggregate outcome** across the full transaction.

### Consent is not protection

Consider a ~$50M swap executed through a DeFi interface:

- The interface displayed a price impact warning
- The user explicitly confirmed the risk
- The transaction executed exactly as designed

The outcome was still economically catastrophic.

The system did not fail technically. The user gave informed consent. The transaction was valid.

**But the outcome violated the user's true intent.**

This is not a bug in Aave or CoW Swap. It is a structural limitation of how DeFi protections work today: they are informational (warnings, UI alerts) or parameter-based (slippage tolerance). They do not enforce the final economic outcome.

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

## Core invariant

```
final_state MUST satisfy all declared constraints

If any constraint is violated:
→ the transaction MUST NOT be included on-chain
```

| Protection | Formal constraint |
|---|---|
| `max_spend(token, X)` | `Δbalance(token) ≥ -X` |
| `min_receive(token, Y)` | `Δbalance(token) ≥ +Y` |
| `no_balance_decrease(token)` | `Δbalance(token) ≥ 0` |

This invariant is enforced on-chain by the post-check transaction. IntentGuard infrastructure cannot override it.

## Agent interface

**Input:** natural language protection intent + signed user transaction

**Output:** transaction included on-chain only if all constraints satisfied — atomic revert with no gas cost otherwise

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

| Layer | Responsibility | Trust |
|---|---|---|
| Skill (this repo) | Natural language → structured protections | Untrusted parsing |
| MCP server | Deterministic compilation → constraints + unsigned bundle | Trusted logic |
| Agent / wallet | Transaction signing | Key holder |
| On-chain enforcer | Post-condition verification | Trustless |

The on-chain enforcer is the only component that gates execution. The skill and MCP server are preparation layers — they cannot approve or block inclusion.

---

## MetaMask delegation angle

Delegation defines what an agent **may** do. IntentGuard defines what outcome is **allowed to happen**.

These are two different guarantees. Without both, a delegated agent is authorized but not safe.

```
MetaMask Delegation  →  permission / capability surface
IntentGuard          →  execution enforcement / economic outcome guardrail
```

A delegated agent that uses IntentGuard is both authorized and constrained. It can only produce outcomes the user pre-approved.

See [examples/03-metamask-delegated-execution.md](examples/03-metamask-delegated-execution.md).

---

## Demo scenarios

| File | What it shows |
|---|---|
| [01-basic-protected-swap.md](examples/01-basic-protected-swap.md) | Happy path — swap executes, constraints satisfied |
| [02-blocked-bad-outcome.md](examples/02-blocked-bad-outcome.md) | Protection triggered — MEV shifts price, tx blocked, no gas lost |
| [03-metamask-delegated-execution.md](examples/03-metamask-delegated-execution.md) | Delegated agent + outcome enforcement combined |
| [04-uniswap-protected-swap.md](examples/04-uniswap-protected-swap.md) | Protocol-specific: Uniswap swap with net balance enforcement |

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

Built by Jean-Loïc Mugnier
Founder, IntentGuard
X: [@theFrBrGuy](https://x.com/theFrBrGuy)

# intentguard-agent

**IntentGuard protects intent by translating it into explicit constraints that users can verify and the chain can enforce.**

A transaction can be signed, valid, and confirmed — and still be economically wrong.

IntentGuard turns user intent into enforceable on-chain constraints. Users express protection in natural language. The agent translates it into explicit balance constraints, presents them for user verification, and only then prepares protected execution. If any constraint is violated, the bundle is not included on-chain — the user transaction never executes, and no gas is paid.

```
User intent  →  explicit constraints  →  enforced at execution
If constraints not satisfied  →  transaction not included  →  no gas paid
```

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

IntentGuard does not build the underlying DeFi action. It takes an unsigned action transaction produced elsewhere (by a protocol skill or orchestrator), wraps it in a protection envelope, and returns the full 3-transaction signing package.

See [e2e_flow.md](e2e_flow.md) for the complete sequence diagram.

Every protected transaction becomes a bundle:

```
preTx      (nonce N)    →  snapshots token balances before execution
actionTx   (nonce N+1)  →  the action transaction, unmodified
postTx     (nonce N+2)  →  verifies outcome constraints after execution
```

The agent signs all three and submits atomically. If any constraint is violated, the bundle is not included on-chain — the action transaction never executes, and no gas is paid.

Failed outcomes are filtered before block inclusion, not after execution. Builders evaluate the bundle atomically — if the post-check would fail, the entire bundle is dropped. That's why the user pays no gas.

The bundle is submitted via **private routing** (Flashbots-compatible). Bots cannot sandwich what they cannot see.

---

## Intent to constraint translation

IntentGuard does not require users to write raw balance constraints directly.

Users express protection in natural language:

```
"Don't lose more than 1000 USDC"
"Receive at least 0.49 WETH"
"My DAI balance must not decrease"
```

The agent translates that into explicit, machine-checkable constraints and **presents them to the user for verification** before preparing protected execution:

```
I translated your intent into these enforceable protections:

  max spend:    1000 USDC  →  ΔUSDC ≥ -1000
  min receive:  0.49 WETH  →  ΔWETH ≥ +0.49

Please confirm before I prepare the protected transaction.
```

Only after confirmation does the agent proceed. The user sees exactly what will be enforced.

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

**Output:** transaction included on-chain only if all constraints satisfied — bundle dropped before inclusion otherwise, no gas paid

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

**Blocked:** MEV bot extracts value, WETH received = 0.41 → post-check fails, bundle is not included, user transaction never executes, no gas paid.

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

## Why this matters for AI-managed capital

Autonomous agents can construct, sign, and submit transactions — but they cannot guarantee that the resulting economic outcome matches the user's intent.

This creates a fundamental gap:

- Agents can be authorized to act
- But there is no native guarantee that execution outcomes are acceptable

IntentGuard closes this gap. It provides an execution enforcement layer where agents translate user intent into explicit constraints, users verify them, and execution only occurs if the final on-chain outcome satisfies them.

This makes IntentGuard a foundational primitive for:
- AI trading agents
- delegated portfolio management
- autonomous execution systems
- on-chain financial automation

IntentGuard is not just a safety layer — it is an execution primitive for AI-managed capital.

Without outcome enforcement, agentic execution is authorized — but not safe.

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
| [00-trigger-conversations.md](examples/00-trigger-conversations.md) | Routing contract — when to trigger, when not to, ambiguous cases |
| [01-basic-protected-swap.md](examples/01-basic-protected-swap.md) | Happy path — swap executes, constraints satisfied |
| [02-blocked-bad-outcome.md](examples/02-blocked-bad-outcome.md) | Protection triggered — MEV shifts price, tx blocked, no gas paid |
| [03-metamask-delegated-execution.md](examples/03-metamask-delegated-execution.md) | Delegated agent + outcome enforcement combined |
| [04-uniswap-protected-swap.md](examples/04-uniswap-protected-swap.md) | Protocol-specific: Uniswap swap with net balance enforcement |
| [05-blocked-token-drain.md](examples/05-blocked-token-drain.md) | Non-swap: protected assets must not move, regardless of calldata |

IntentGuard is not limited to swaps. The same enforcement primitive prevents unauthorized asset movement — whether from MEV, unexpected contract behavior, or malicious calldata — by enforcing that protected balances must not decrease.

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

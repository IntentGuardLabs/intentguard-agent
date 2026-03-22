# intentguard-agent

**IntentGuard protects intent by translating it into explicit constraints that users can verify and the chain can enforce.**

A transaction can be signed, valid, and confirmed — and still be economically wrong.

IntentGuard turns user intent into enforceable on-chain constraints. Users express protection in natural language. The agent translates it into explicit balance constraints, presents them for user verification, and only then prepares protected execution. If any constraint is violated, the bundle is not included on-chain — the user transaction never executes, and no gas is paid.

```
User intent  →  explicit constraints  →  enforced at execution
If constraints not satisfied  →  transaction not included  →  no gas paid
```

IntentGuard sits between intent creation and transaction submission. Not a wallet. Not a protocol. Not a UI. An execution enforcement layer that any agent can call.

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

## MetaMask

MetaMask defines what an agent is allowed to do.

IntentGuard defines what outcome is allowed to happen.

This allows safe delegated execution — agents can act, but only within enforced economic constraints.

See [examples/03-metamask-delegated-execution.md](examples/03-metamask-delegated-execution.md).

---

## Uniswap

IntentGuard is protocol-agnostic and works on top of Uniswap swaps.

While Uniswap enforces execution parameters (e.g. `amountOutMin`), IntentGuard enforces the final balance outcome.

This protects against:
- MEV
- multi-hop slippage
- unexpected price movements

See [examples/04-uniswap-protected-swap.md](examples/04-uniswap-protected-swap.md).

---

## Agent Service Model

Any agent that moves money can pay IntentGuard before execution to ensure the outcome is correct.

IntentGuard is exposed as a callable service endpoint (e.g. MCP or RPC) that agents invoke before execution. Before submitting a transaction, an agent calls IntentGuard to obtain protected execution guarantees and pays for that service via an agent-native payment mechanism such as [x402](https://x402.org).

```
Trading agent  →  x402 payment  →  IntentGuard service  →  protected bundle
```

The agent is both the caller and the economic actor. There is no human in the payment loop.

IntentGuard can be deployed as an agent service on Base, where agents can discover, invoke, and pay for protection before executing transactions.

**Agent consumer types:**
- **Trading agents** — autonomous systems executing swaps with outcome guarantees
- **Portfolio managers** — agents enforcing position constraints across multi-step workflows
- **Execution pipelines** — orchestrators that route actions through IntentGuard as a protection layer

**Why agents pay for outcome enforcement:**

Agents can be trusted to construct valid transactions. They cannot be trusted to guarantee acceptable outcomes — that requires an independent enforcement layer with on-chain finality.

IntentGuard is that layer. Agents pay for certainty: the transaction executes only if the outcome is acceptable. If it is not, nothing happens and no gas is consumed.

**Pricing model (intended):**
- Per protected transaction fee — pay only when you submit
- Subscription tier for high-frequency agents
- Pricing is usage-based and aligns with transaction risk and frequency

x402 is the intended billing interface. It is not yet implemented in v1 — protection is currently provided without a payment gate.

See [examples/06-agent-pays-for-protection.md](examples/06-agent-pays-for-protection.md).

---

## Why this fits Base — Agent Services on Base

IntentGuard is an outcome enforcement service designed to be consumed by agents, not humans.

| Property | Detail |
|---|---|
| Agent-native interface | MCP tools, structured JSON protections, no human UX required |
| Service model | Agents pay for execution guarantees — per-transaction billing via x402 |
| Protocol-agnostic | Works on any EVM contract — Uniswap, Aave, CoW, custom calldata |
| On-chain enforcement | Post-condition verified trustlessly on Base — no centralized gate |
| MEV protection | Private relay routing — bots cannot sandwich what they cannot see |
| No gas on bad outcomes | Bundle dropped before inclusion — agent pays only for successful execution |

IntentGuard fills a gap in the agentic execution stack: authorization (MetaMask Delegation) tells agents what they may do; IntentGuard tells the chain what outcomes are acceptable. Together they make autonomous financial agents safe to deploy.

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
| [06-agent-pays-for-protection.md](examples/06-agent-pays-for-protection.md) | Agent-to-agent service flow — autonomous execution with x402 payment framing |

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

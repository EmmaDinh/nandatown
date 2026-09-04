# Experiment: message_drop resilience in the marketplace scenario

## Choice

Scenario: **marketplace** (built-in, tier 1, seed 42, 50 buyers / 50 sellers).

Setting changed: `failures.message_drop`, from `0.0` to `0.05`. Everything else
(seed, agent counts, layers, `task.config`) is identical to the built-in
`marketplace.yaml`.

I picked `message_drop` over a `task.config` knob because it's the failure
axis most directly tied to this scenario's built-in validators
(`marketplace_all_responded`, `marketplace_price_agreement`), and because
failure injection uses a separate RNG stream from agent behavior — so any
change in outcome should trace back to actual message loss, not confounded
randomness from a shared seed.

## Hypothesis

Before running: I expected `success_rate` and total message counts to drop
by roughly the drop rate itself (~5%), and expected
`marketplace_all_responded` might start failing for a handful of requests.

## Evidence

| Metric | Baseline (`drop=0.0`) | Modified (`drop=0.05`) | Change |
|---|---|---|---|
| Attempted messages (correlation IDs) | 1000 | 593 | **−41%** |
| Explicitly dropped | 0 | 35 (~5.9% of attempts) | — |
| Completed sales | 266 | 174 | **−35%** |
| `marketplace_no_double_sell` | PASS | PASS | unchanged |
| `marketplace_all_responded` | PASS (500/500 answered) | **FAIL** (15 unanswered) | flipped |
| `marketplace_price_agreement` | PASS | PASS | unchanged |

Raw traces for both runs are included under `traces/`.

## Investigation — what surprised me

The drop rate itself landed right on target (35 of 593 attempts, ~5.9%), but
total system activity fell 41%, not ~5%. To understand why, I pulled the raw
JSONL and traced every event for a single dropped message: `buyer-13`'s
round-1 offer to `seller-31`.

- **Baseline**: `buyer-13` completes all 10 negotiation rounds (10 send/receive pairs).
- **Modified**: that one dropped message is `buyer-13`'s *only* event in the
  entire trace — it never attempts rounds 2 through 10.

The reference marketplace agent has no retry or timeout logic, and rounds
are strictly sequential per buyer: a single lost message permanently strands
the rest of that buyer's session. This compounds geometrically — the
probability a buyer survives all 10 rounds is `0.95^10 ≈ 60%` — which lines
up with the ~40% system-wide falloff actually observed. A flat per-message
drop rate becomes a much larger transaction-failure rate because there's no
resilience layer absorbing transient loss.

## Tools and help used

Claude Code (Sonnet 5) as a coding agent for the full workflow: installing
`nest-core` via `uv`, running `nest doctor` / `nest run` / `nest inspect`,
editing the scenario YAML, and pulling `docs/writing-a-scenario.md` and
`CONTRIBUTING.md` from this repo for reference. A short inline Python
snippet (not committed) parsed the JSONL trace to isolate one correlation
ID's full event history and compute the `0.95^10` compounding model above.
No other tutorials or human help were used beyond this repo's own README
and quickstart docs.

## What I'd build next

A pluggable retry/timeout wrapper for the negotiation layer: if a buyer's
offer isn't acknowledged within N ticks, it re-sends (capped retry count)
before giving up. This experiment showed the marketplace task has zero
resilience to transient message loss — a 5% drop rate produced a 40%
collapse in completed transactions purely from missing retry logic. A
resilience layer would let researchers separate "the protocol design is
fragile" from "the deployment has no fault tolerance," which seems useful
for anyone using Nanda Town to evaluate protocols under realistic network
conditions.

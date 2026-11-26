# Steady Flow Bot

[![make.fun](https://make.fun/badge/u-jackxyz72-pump-fun-sharpe-sprint-cleanest-03e4f532.svg)](https://make.fun/bounty/u-jackxyz72-pump-fun-sharpe-sprint-cleanest-03e4f532)
[![rank](https://make.fun/badge/u-jackxyz72-pump-fun-sharpe-sprint-cleanest-03e4f532/iqbalmaulana25/pump-fun-sharpe-sprint-bot/rank.svg)](https://make.fun/bounty/u-jackxyz72-pump-fun-sharpe-sprint-cleanest-03e4f532)
[![reward](https://make.fun/badge/u-jackxyz72-pump-fun-sharpe-sprint-cleanest-03e4f532/iqbalmaulana25/pump-fun-sharpe-sprint-bot/reward.svg)](https://make.fun/bounty/u-jackxyz72-pump-fun-sharpe-sprint-cleanest-03e4f532)

A disciplined momentum paper-trading bot for the [Pump.fun Sharpe Sprint — Cleanest Paper Bot Wins](https://make.fun/bounty/u-jackxyz72-pump-fun-sharpe-sprint-cleanest-03e4f532) bounty on make.fun. It trades 10 SOL of paper money on a replay of 6 hours of recorded pump.fun bonding-curve activity. Places go by hourly Sharpe ratio, best first; a placement needs net PnL above 0 after every fee.

## Strategy

Ranking is hourly Sharpe, not raw PnL — so the design goal is a smooth equity curve: many small, high-probability wins, losers cut fast, never a big swing either way:

- **Maturity filter** — at least 300 slots since the curve's `CreateEvent` (fresh launches are mostly bundled dumps), real SOL reserves between 2 and 60 SOL.
- **Confirmation, not every candle** — over a 50-slot window: net inflow ≥ 1.2 SOL, ≥ 5 buy trades from ≥ 5 distinct wallets, price up 3–40% and sitting at ≥ 94% of the window high, and the newer half of the window inflowing at least as much as the older half (momentum building, not fading). No single wallet may dominate more than 50% of the buys.
- **Small, disciplined sizing** — first buy 0.05 SOL, pyramid adds of 0.05 SOL only while the position is in profit, the price stepped up from the last add, and net SOL still flows in; at most 0.2 SOL per position, 5 positions, 1.5 SOL total exposure.
- **Fast exits** — take profit +6%, stop-loss −4%, trailing stop −8% from peak (armed at +5%), exit on fading flow (net < −0.5 SOL over 15 slots), and a 900-slot time limit unless the position is up ≥ 15%.
- **No chasing, no overtrading** — one entry signal per curve at a time, 150-slot cooldown after an exit, at most 2 entries per curve. Fees (1.25% curve + tx fees) punish churn, so the bot waits for confirmation.

## Files

- `main.ts` — entrypoint; the grader runs `node /bot/main.ts` (Node 22, stdlib only, no dependencies).
- `vendor/makefun-bot.ts` — the kit's `agent-fun/replay/v2` protocol shim, copied verbatim from the official kit.

## Local test

Grade locally the way the server does (process driver; docker is not installed here):

```sh
node <kit>/cli/makefun.mjs grade-local --kit <kit> --bot . --driver process
```

Dev-window result (kit's frozen dev dataset, run locally with `grade-local --driver process`, Node v24.22.1 — the server grades on Node v26.13.1 in docker, so this is indicative, not the server score): **net PnL −0.46 SOL** (return −4.6%, 90% interval −0.62…−0.30 SOL), 397 fills over 113 coins, 182 round trips, win rate 15.9%, max drawdown 4.6%, fees 0.25 SOL curve + 0.04 SOL tx. **Guardrails met** (113 coins hold a round trip ≥ 0.01 SOL, drawdown well under 50%). A revision round tightened the entry bar (net inflow 1.2→1.5 SOL, 6 trades / 6 distinct buyers, accel 1.1→1.2) and extracted the exits into a `shouldExit` helper with a cap of 2 new entries per batch; dev PnL was essentially unchanged (−0.4593 vs −0.4596 SOL) with fewer fills.

Honest read: the dev window is brutal for momentum — the kit's own pilot momentum bot scores −9.49 SOL with a 95.7% drawdown (fails the guardrail) on the same data, and the earlier jackxyz momentum entry scored −1.69 SOL / 16.9% drawdown. This bot loses far less than both and keeps drawdown at 4.6%, which is what a Sharpe ranking rewards: survival plus guardrail compliance, with upside if a few runners pay. A tuning round with a wider +10% take-profit and stricter entries scored slightly worse (−0.50 SOL) and was reverted. The final 6h window is different data, so no dev-window number is promised.

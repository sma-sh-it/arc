# ARC — Asymmetric Reaction Capture

## Build Plan v2.0

**Created:** 2026-03-13
**Updated:** 2026-03-13
**Author:** Robert
**Repository:** `sma-sh-it/arc`
**Local Path:** `~/Projects/arc/`

---

## Project Thesis

ARC is a **single-signal reactive trading system** that detects when a leading asset (e.g., WLFI) makes a **volume-confirmed directional reversal** before a following asset (e.g., BTC), then executes a reactive trade on the follower to capture the catch-up move.

The edge is **information latency asymmetry** — low-liquidity leading assets reflect market pressure minutes before it propagates to high-liquidity assets. ARC exploits that gap.

**Core Principles:**

- One trigger, one action. No fusion, no multi-strategy convergence.
- Trade the follower (deep liquidity, tight spreads), watch the leader (thin liquidity, fast moves).
- Both trigger and confirmation require volume conviction — price moves without order flow are noise.
- Fast in, trailing stop out. Ride the reaction, don't target a fixed exit.
- Positive expectancy through frequency, not magnitude.
- Pluggable confirmation strategies allow backtesting multiple approaches to find the strongest edge.

**ARC is NOT OFF.** OFF is multi-source signal fusion on 4H timeframes. ARC is single-signal reactive scalping on 15min–1H timeframes. They share no decision logic. However, ARC's higher-tier confirmation strategies borrow *detection concepts* (absorption, iceberg, OI shifts) proven in OFF's pipeline — not the code itself.

---

## Signal Model

### Trigger (Leader Side — e.g., WLFI)

The trigger fires when **BOTH** conditions are met simultaneously:

1. **Direction flip** — WLFI's short EMA slope changes sign (positive → negative = bearish flip, negative → positive = bullish flip). Alternatively, ROC zero-cross (configurable method).
2. **Volume spike** — WLFI volume on the flip bar exceeds X× its N-bar moving average. A direction flip on thin volume is noise — someone with size must be behind it.

### Confirmation (Follower Side — e.g., BTC)

Confirmation fires when **BOTH** conditions are met:

1. **Price move** — BTC moves ≥ N% in the same direction as WLFI's flip.
2. **Volume spike** — BTC volume exceeds Y× its N-bar moving average, confirming real order flow behind the move.

### Pluggable Confirmation Tiers

The confirmation logic is a swappable module. All tiers share the same trigger. Only the follower-side confirmation differs:

| Tier | Name              | Confirmation Criteria                                         | Position Sizing |
|------|-------------------|---------------------------------------------------------------|-----------------|
| 1    | Volume Threshold  | BTC ≥ N% + BTC volume spike                                  | Base size       |
| 2    | Absorption        | Tier 1 + absorption pattern detected on BTC                  | 1.5× base       |
| 3    | Full Conviction   | Tier 2 + OI/funding shift from CoinGlass derivatives data    | 2× base         |

Higher tier = stronger signal = larger position via the risk manager.

### State Machine

```
IDLE ──────────────────────────────────────────────────────► IDLE
  │                                                            ▲
  │  WLFI direction flip                                       │
  │  + WLFI volume spike                                       │
  │                                                            │
  ▼                                                            │
TRIGGERED ─── BTC ≥ N% same direction + BTC volume spike ─────┘
  │           (confirmation strategy fires)        (CONFIRMED)
  │                                                            │
  │  barsSinceSignal >= maxLeadBars ───────────────────────────┘
  │                                                (TIMEOUT)
  │                                                            │
  │  WLFI flips opposite direction + volume ───► TRIGGERED     │
  │                                        (override/reset)
```

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         ARC Pipeline                              │
│                                                                   │
│  ┌──────────────┐    ┌──────────────────┐    ┌────────────────┐  │
│  │ Leader        │───▶│  Divergence       │───▶│  Execution     │  │
│  │ Monitor       │    │  Detector         │    │  Engine        │  │
│  │ (price + vol) │    │  (direction flip  │    │  (Kraken Mkt   │  │
│  └──────────────┘    │   + vol trigger)  │    │   + Trailing)  │  │
│                       │                   │    └───────┬────────┘  │
│  ┌──────────────┐    │  ┌─────────────┐  │            │           │
│  │ Follower      │───▶│  │ Confirmation│  │    ┌───────▼────────┐  │
│  │ Monitor       │    │  │ Strategy    │  │    │ Risk Manager   │  │
│  │ (price + vol) │    │  │ (pluggable) │  │    │ (Sizing by     │  │
│  └──────────────┘    │  └─────────────┘  │    │  tier, limits, │  │
│                       └──────────────────┘    │  circuit break) │  │
│  ┌──────────────┐                             └───────┬────────┘  │
│  │ Config        │    ┌──────────────────┐            │           │
│  │ Loader        │───▶│ Session Manager   │    ┌───────▼────────┐  │
│  └──────────────┘    └──────────────────┘    │ Notifier        │  │
│                                               │ (Fast/Terse)    │  │
│  ┌──────────────┐                             └────────────────┘  │
│  │ CoinGlass     │ (Tier 3 only)                                  │
│  │ Derivatives   │                                                │
│  └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Component         | Technology                                    |
|-------------------|-----------------------------------------------|
| Language          | Python 3.12+                                  |
| Exchange          | Kraken (REST + WebSocket)                     |
| Leader Data       | Kraken WebSocket ticker / REST OHLCV          |
| Follower Data     | Kraken WebSocket ticker / REST OHLCV          |
| Derivatives Data  | CoinGlass API (Tier 3 confirmation only)      |
| Notifications     | Pushover (terse/fast) + Telegram              |
| Config            | YAML                                          |
| Secrets           | `~/.arc/secrets.env` (single-quoted values)   |
| Logging           | Python logging → file + console               |
| OS                | macOS (Mac Mini M4 Pro)                       |
| Version Control   | Git → GitHub (`sma-sh-it/arc`)                |

---

## Phase Plan

### Phase 1 — Foundation (ARC-001 → ARC-005)

Scaffold, configuration, secrets management, and project infrastructure.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-001 | Repository scaffold + .gitignore + README | DONE    |
| ARC-002 | ConfigLoader (YAML, pair config, TF)      | PLANNED |
| ARC-003 | SecretsLoader (`~/.arc/secrets.env`)      | PLANNED |
| ARC-004 | Logging framework (file + console)        | PLANNED |
| ARC-005 | Base exceptions and error hierarchy        | PLANNED |

**Gate:** Config loads, secrets resolve, logging writes to file. Go/No-Go.

---

### Phase 2 — Data Feeds (ARC-006 → ARC-010)

Real-time price AND volume feeds for leader and follower via Kraken WebSocket.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-006 | Kraken WebSocket client (ticker + volume) | PLANNED |
| ARC-007 | LeaderMonitor (price, volume, % change,   | PLANNED |
|         | EMA slope, volume MA)                     |         |
| ARC-008 | FollowerMonitor (price, volume, % change, | PLANNED |
|         | volume MA)                                |         |
| ARC-009 | OHLCV fallback (REST polling for low-liq) | PLANNED |
| ARC-010 | Data feed health checks + reconnect logic | PLANNED |

**Gate:** Both feeds streaming live price + volume data, derived metrics (EMA slope, volume MA) computing correctly, auto-reconnect on drop.

---

### Phase 3 — Divergence Detection + Confirmation Strategies (ARC-011 → ARC-022)

The core state machine and pluggable confirmation architecture.

#### 3A — Trigger Logic (Leader Side)

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-011 | Direction flip detector (EMA slope sign   | PLANNED |
|         | change + ROC zero-cross, selectable)      |         |
| ARC-012 | Leader volume spike validation (volume >  | PLANNED |
|         | X× N-bar MA on flip bar)                 |         |
| ARC-013 | Signal model (direction, magnitude, vol   | PLANNED |
|         | multiplier, timestamp, bar_index)         |         |
| ARC-014 | DivergenceDetector state machine          | PLANNED |
| ARC-015 | Signal timeout logic                      | PLANNED |

#### 3B — Confirmation Strategies (Follower Side)

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-016 | ConfirmationStrategy abstract base class  | PLANNED |
| ARC-017 | Tier 1: VolumeThreshold (BTC ≥ N% +      | PLANNED |
|         | BTC volume > Y× MA)                      |         |
| ARC-018 | Tier 2: Absorption (Tier 1 + absorption   | PLANNED |
|         | pattern detection on BTC order flow)      |         |
| ARC-019 | Tier 3: FullConviction (Tier 2 + OI/      | PLANNED |
|         | funding shift from CoinGlass)             |         |
| ARC-020 | Strategy registry + factory (--strategy   | PLANNED |
|         | CLI flag selects confirmation module)     |         |

#### 3C — Testing

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-021 | Unit tests: all state transitions         | PLANNED |
| ARC-022 | Unit tests: each confirmation strategy    | PLANNED |
|         | against known scenarios                   |         |

**Gate:** State machine + all 3 confirmation strategies produce correct signals against known test scenarios. All transitions and edge cases covered by tests.

---

### Phase 4 — Backtest Framework + Strategy Comparison (ARC-023 → ARC-031)

Validate the edge with historical data BEFORE building execution.
Run all 3 confirmation strategies against the same dataset for head-to-head comparison.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-023 | Historical data fetcher (Kraken OHLCV     | PLANNED |
|         | + volume for both leader and follower)    |         |
| ARC-024 | WLFI historical data sourcing strategy    | PLANNED |
| ARC-025 | CoinGlass historical OI/funding data      | PLANNED |
|         | fetcher (for Tier 3 backtest)             |         |
| ARC-026 | BacktestEngine (replay bars through       | PLANNED |
|         | DivergenceDetector + selected strategy)   |         |
| ARC-027 | P&L calculator (fees, slippage model)     | PLANNED |
| ARC-028 | BacktestReporter (win rate, expectancy,   | PLANNED |
|         | Sharpe, max drawdown, profit factor)      |         |
| ARC-029 | Parameter sweep (threshold optimization   | PLANNED |
|         | per strategy)                             |         |
| ARC-030 | Strategy comparison report (side-by-side  | PLANNED |
|         | Tier 1 vs Tier 2 vs Tier 3 results)      |         |
| ARC-031 | Minimum sample size validation (≥200      | PLANNED |
|         | signals per strategy)                     |         |

**CLI Usage:**
```bash
# Run individual strategy backtest
python -m arc.backtest.run --strategy volume_threshold
python -m arc.backtest.run --strategy absorption
python -m arc.backtest.run --strategy full_conviction

# Run all strategies and produce comparison report
python -m arc.backtest.run --compare-all
```

**Gate:** At least one strategy confirms positive expectancy after fees across ≥200 signals on 15min+ timeframe. The winning strategy is selected for paper trading. If ALL strategies are negative → project pauses for thesis review.

**THIS IS THE CRITICAL GATE.** Do not proceed to execution without statistical validation.

---

### Phase 5 — Execution Engine (ARC-032 → ARC-036)

Kraken market orders with trailing stop exits.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-032 | KrakenAdapter (market order, cancel,      | PLANNED |
|         | status query — forked from OFF)           |         |
| ARC-033 | TrailingStopManager (dynamic stop-loss    | PLANNED |
|         | that follows price)                       |         |
| ARC-034 | OrderStateTracker (open, filled, stopped, | PLANNED |
|         | expired)                                  |         |
| ARC-035 | Execution timeout (max hold duration)     | PLANNED |
| ARC-036 | Dry-run mode (log orders, don't execute)  | PLANNED |

**Gate:** Orders execute on Kraken sandbox/paper, trailing stops trigger correctly, state tracking is accurate.

---

### Phase 6 — Risk Management (ARC-037 → ARC-042)

Position sizing (tiered by confirmation strategy), loss limits, and circuit breakers.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-037 | PositionSizer (Kelly criterion or fixed%, | PLANNED |
|         | multiplied by confirmation tier weight)   |         |
| ARC-038 | Max concurrent positions limit            | PLANNED |
| ARC-039 | Daily loss circuit breaker                | PLANNED |
| ARC-040 | Per-trade max loss (hard stop beneath     | PLANNED |
|         | trailing stop)                            |         |
| ARC-041 | Cooldown period after consecutive losses  | PLANNED |
| ARC-042 | Risk config in YAML (per-tier sizing)     | PLANNED |

**Gate:** Risk manager blocks oversized positions, tier multipliers apply correctly, circuit breaker halts trading after daily limit hit.

---

### Phase 7 — Notifications (ARC-043 → ARC-046)

Fast, terse alerts optimized for scalper context.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-043 | Notifier (Pushover + Telegram, forked     | PLANNED |
|         | from OFF)                                 |         |
| ARC-044 | Alert templates: TRIGGER, ENTRY, EXIT,    | PLANNED |
|         | TIMEOUT, CIRCUIT_BREAK (include tier)     |         |
| ARC-045 | Pushover emergency for circuit breaker    | PLANNED |
| ARC-046 | Apple Watch glanceable format             | PLANNED |

**Gate:** All alert types fire correctly, emergency alerts escalate to Apple Watch.

---

### Phase 8 — Paper Trading (ARC-047 → ARC-052)

End-to-end paper mode — full pipeline, no real money.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-047 | PaperExecutor (simulated fills)           | PLANNED |
| ARC-048 | SessionManager (session ID, log files,    | PLANNED |
|         | P&L accumulator, strategy label)          |         |
| ARC-049 | SessionReporter (per-session stats,       | PLANNED |
|         | strategy performance breakdown)           |         |
| ARC-050 | Pipeline runner (orchestrates all          | PLANNED |
|         | components, --strategy flag)              |         |
| ARC-051 | Health endpoint                           | PLANNED |
| ARC-052 | Launch Agent (macOS launchd)              | PLANNED |

**Gate:** Paper session runs unattended for 48+ hours. All components healthy. P&L tracking accurate.

---

### Phase 9 — Observation & Tuning (ARC-053 → ARC-055)

Run paper, watch, adjust. Minimum 2-week observation.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-053 | 2-week paper observation window           | PLANNED |
| ARC-054 | Parameter tuning based on live paper data | PLANNED |
| ARC-055 | Go/No-Go report for live cutover          | PLANNED |

**Gate:** 2-week paper results confirm backtest expectations (within 1 standard deviation). Go/No-Go decision.

---

### Phase 10 — Live Cutover (ARC-056 → ARC-059)

Real money. Small position sizes. Gradual ramp.

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-056 | Live config (real API keys, real sizing)  | PLANNED |
| ARC-057 | Gradual ramp plan (25% → 50% → 100%      | PLANNED |
|         | position sizing over 2 weeks)             |         |
| ARC-058 | Live monitoring dashboard                 | PLANNED |
| ARC-059 | Kill switch (emergency shutdown)          | PLANNED |

**Gate:** Live P&L positive after 1-week ramp at 25% sizing.

---

### Phase 11 — Post-Launch Enhancements (Future)

| Ticket  | Task                                      | Status  |
|---------|-------------------------------------------|---------|
| ARC-060 | Additional leader-follower pairs          | PLANNED |
| ARC-061 | Multi-pair simultaneous monitoring        | PLANNED |
| ARC-062 | Adaptive thresholds (auto-tune based on   | PLANNED |
|         | recent volatility)                        |         |
| ARC-063 | Web dashboard for live monitoring         | PLANNED |
| ARC-064 | Performance attribution reporting         | PLANNED |
| ARC-065 | Cross-tier A/B paper testing (run Tier 1  | PLANNED |
|         | + Tier 2 simultaneously on live data)     |         |

---

## Directory Structure

```
~/Projects/arc/
├── arc/
│   ├── __init__.py
│   ├── config/
│   │   ├── __init__.py
│   │   └── loader.py                  # ConfigLoader (YAML)
│   ├── data/
│   │   ├── __init__.py
│   │   ├── kraken_ws.py               # Kraken WebSocket client (price + vol)
│   │   ├── leader_monitor.py          # Leader feed + EMA slope + vol MA
│   │   ├── follower_monitor.py        # Follower feed + vol MA
│   │   └── coinglass_client.py        # CoinGlass OI/funding (Tier 3)
│   ├── detection/
│   │   ├── __init__.py
│   │   ├── detector.py                # DivergenceDetector state machine
│   │   ├── direction_flip.py          # Direction flip detection (EMA/ROC)
│   │   ├── volume_validator.py        # Volume spike validation logic
│   │   ├── models.py                  # Signal model (dataclasses)
│   │   └── confirmations/
│   │       ├── __init__.py
│   │       ├── base.py                # ConfirmationStrategy (abstract)
│   │       ├── volume_threshold.py    # Tier 1: price ≥ N% + volume spike
│   │       ├── absorption.py          # Tier 2: Tier 1 + absorption pattern
│   │       ├── full_conviction.py     # Tier 3: Tier 2 + OI/funding shift
│   │       └── registry.py            # Strategy registry + factory
│   ├── execution/
│   │   ├── __init__.py
│   │   ├── kraken_adapter.py          # Kraken REST order management
│   │   ├── trailing_stop.py           # TrailingStopManager
│   │   ├── paper_executor.py          # Simulated fills for paper mode
│   │   └── order_state.py             # OrderStateTracker
│   ├── risk/
│   │   ├── __init__.py
│   │   ├── position_sizer.py          # Kelly / fixed %, tier-weighted
│   │   ├── circuit_breaker.py         # Daily loss limit + cooldown
│   │   └── risk_config.py             # Risk parameter models
│   ├── notify/
│   │   ├── __init__.py
│   │   └── notifier.py                # Pushover + Telegram (terse)
│   ├── session/
│   │   ├── __init__.py
│   │   ├── manager.py                 # SessionManager
│   │   └── reporter.py                # SessionReporter + P&L
│   ├── backtest/
│   │   ├── __init__.py
│   │   ├── engine.py                  # BacktestEngine (bar replay)
│   │   ├── data_fetcher.py            # Historical OHLCV + volume fetcher
│   │   ├── coinglass_fetcher.py       # Historical OI/funding (Tier 3)
│   │   ├── pnl_calculator.py          # Fee + slippage model
│   │   ├── reporter.py                # BacktestReporter (stats)
│   │   └── comparator.py             # Strategy comparison report
│   ├── pipeline/
│   │   ├── __init__.py
│   │   ├── runner.py                  # Pipeline orchestrator
│   │   ├── health.py                  # Health endpoint
│   │   └── cli.py                     # CLI entry point (--strategy flag)
│   └── utils/
│       ├── __init__.py
│       ├── secrets.py                 # SecretsLoader (~/.arc/secrets.env)
│       ├── logging.py                 # Logging framework
│       └── exceptions.py              # ARC exception hierarchy
├── tests/
│   ├── __init__.py
│   ├── test_detector.py
│   ├── test_direction_flip.py
│   ├── test_volume_validator.py
│   ├── test_confirmations.py
│   ├── test_risk.py
│   ├── test_execution.py
│   └── test_backtest.py
├── scripts/
│   └── fetch_historical.py            # One-off historical data pulls
├── config/
│   ├── config.yaml                    # Main config
│   ├── config_paper.yaml              # Paper trading config
│   └── pairs.yaml                     # Leader-follower pair definitions
├── docs/
│   ├── BUILD_PLAN.md                  # This document
│   └── RUNBOOK.md                     # Operational runbook
├── indicators/
│   └── Lead_Lag_Signal_Detector_v2_0.pine  # TradingView indicator
├── data/
│   └── .gitkeep                       # Historical data (gitignored)
├── logs/
│   └── .gitkeep                       # Session logs (gitignored)
├── .env.example
├── .gitignore
├── README.md
├── requirements.txt
└── pyproject.toml
```

---

## Glossary

| Term                       | Definition                                                        |
|----------------------------|-------------------------------------------------------------------|
| **Leader**                 | The asset that moves first (e.g., WLFI)                          |
| **Follower**               | The asset that reacts later (e.g., BTC) — the traded asset       |
| **Direction Flip**         | Leader's short EMA slope or ROC changes sign (the trigger event) |
| **Volume Spike**           | Volume exceeding X× the N-bar moving average on a given bar      |
| **Divergence**             | Leader has flipped direction with volume; follower has not yet    |
|                            | moved ≥ N% in that direction                                     |
| **Confirmation**           | Follower moves ≥ N% in the leader's new direction with a         |
|                            | volume spike (Tier 1), or with additional order flow / OI        |
|                            | validation (Tier 2–3)                                            |
| **Confirmation Strategy**  | Pluggable module that defines what qualifies as follower          |
|                            | confirmation — swapped via --strategy CLI flag                    |
| **Tier 1 (Volume)**        | BTC ≥ N% + volume spike. Base position size.                     |
| **Tier 2 (Absorption)**    | Tier 1 + absorption pattern on BTC. 1.5× position size.          |
| **Tier 3 (Full Convict.)** | Tier 2 + OI/funding shift from CoinGlass. 2× position size.      |
| **Timeout**                | Follower never confirmed within the max lead window               |
| **Trailing Stop**          | Dynamic stop-loss that follows price, locking in gains as the    |
|                            | reaction move extends                                            |
| **Circuit Breaker**        | Emergency halt triggered by daily loss limit                      |
| **Lead Window**            | Maximum bars to wait for follower confirmation                    |
| **Asymmetry**              | The information latency gap between leader and follower           |
| **Reaction Capture**       | The profit extracted from the follower's catch-up move            |
| **Paper Mode**             | Full pipeline with simulated execution (no real orders)           |
| **Live Mode**              | Full pipeline with real Kraken orders                             |
| **Kelly Criterion**        | Position sizing formula based on win rate and win/loss ratio      |
| **ROC**                    | Rate of Change — % price change over N bars. Zero-cross = flip.  |
| **EMA Slope**              | Direction of a short exponential moving average. Sign change =    |
|                            | direction flip.                                                   |

---

## Changelog

| Version | Date       | Changes                                                         |
|---------|------------|-----------------------------------------------------------------|
| v2.0    | 2026-03-13 | Major revision: direction-flip trigger (replaces magnitude      |
|         |            | threshold), volume validation on both trigger and confirmation, |
|         |            | pluggable confirmation strategy architecture (Tier 1–3),        |
|         |            | strategy comparison backtest, updated directory structure,       |
|         |            | renumbered all tickets.                                         |
| v1.0    | 2026-03-13 | Initial build plan. Magnitude-based trigger. Single             |
|         |            | confirmation approach.                                          |

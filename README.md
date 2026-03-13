# ARC — Asymmetric Reaction Capture

A single-signal reactive trading system that detects when a leading asset makes a significant directional move before a following asset, then executes a reactive trade on the follower to capture the catch-up move.

## Thesis

Low-liquidity assets (e.g., WLFI) reflect market pressure minutes before high-liquidity assets (e.g., BTC). ARC monitors the leader for threshold-crossing moves, detects the divergence window where the follower hasn't reacted yet, and executes a market order with a trailing stop to capture the reaction.

**The edge is information latency asymmetry.**

## Status

**Phase 1 — Foundation** (In Progress)

## Architecture

```
Leader Monitor → Divergence Detector → Execution Engine → Risk Manager → Notifier
Follower Monitor ↗                                           ↓
                                                      Session Manager
```

## Quick Start

```bash
# Clone
git clone git@github.com:sma-sh-it/arc.git
cd arc

# Virtual environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Secrets
mkdir -p ~/.arc
cp .env.example ~/.arc/secrets.env
# Edit ~/.arc/secrets.env with your API keys

# Run (paper mode)
python -m arc.pipeline.cli --config config/config_paper.yaml
```

## Documentation

- [Build Plan](docs/BUILD_PLAN.md)
- [Operational Runbook](docs/RUNBOOK.md) *(Phase 8+)*

## License

Private. Not for distribution.

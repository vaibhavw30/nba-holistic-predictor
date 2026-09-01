# Benchwarmer — NBA Prediction, Trading Engine, and Signal Research

Three connected components: a machine-learning model that predicts NBA game
outcomes, a C++20 engine that trades those predictions as fair values on Kalshi
event markets (paper-only), and a research harness that measures whether the
signal is actually any good.

> **📖 Full map: [Project Overview](docs/PROJECT-OVERVIEW.md)** — how the three
> pieces fit together end to end.

## Components at a glance

| Component | What it is | Status |
|---|---|---|
| **NBA prediction platform** (`backend_ml/`, `frontend_web/`) | XGBoost + Ridge ensemble over engineered team features; predictions served to a React dashboard via Supabase | **Working, accuracy below target.** Full pipeline runs end to end; 61.55% on the held-out split vs a >65% goal |
| **C++20 trading engine** (`trading_engine/`) | Kalshi WebSocket ingest, order-book reconstruction, strategy → risk → paper execution, deterministic replay harness | **Working, paper-only.** 33 tests passing; live socket path never run against a real account |
| **Signal-research harness** (`backend_ml/signal_research/`) | Calibration (Brier/ECE/isotonic/Platt), closing-line-value analysis, market capture, settlement | **Working.** Part of a 193-test Python suite |

Nothing here places real orders, and no real money is at risk. The only
execution path is a simulated paper venue.

---

## 1. NBA prediction platform

**Status: working end to end; predictive accuracy is the open problem.**

An XGBoost + Ridge ensemble over 18 engineered features — Elo ratings, EWMA'd
four-factor stats, rest/fatigue, momentum, and home-court context. Feature
construction is causal by design: every rolling statistic is `shift(1)`-ed
before its window so a game never sees its own outcome
(`backend_ml/data_engine.py:66-78`), and Elo is recorded pre-game
(`backend_ml/elo_engine.py:54-56`).

What works:

- **Data collection** — `data_engine.py` pulls from the NBA Stats API and caches
  a 13,182-game training set spanning 2015–2026.
- **Training** — `train_model.py` does a chronological 85/15 split with
  `GridSearchCV` hyperparameter tuning and the scaler fit on train only.
- **Recency weighting** — an exponential half-life weighting scheme
  (`recency.py`) with a walk-forward sweep to pick the half-life
  (`halflife_sweep.py`). The sweep currently selects *uniform* weighting: no
  finite half-life cleared the acceptance margin, so it ships dormant.
- **Prediction** — `predict.py` generates predictions and upserts them to
  Supabase (`predict.py:510-533`).
- **Automated retraining** — a launchd job runs nightly, gated on measured
  model drift, deploying a new model only if held-out accuracy holds within
  tolerance (`scheduled_retrain.py`, `backend_ml/scripts/*.plist`).
- **Frontend** — React dashboard reading real predictions from Supabase
  (`frontend_web/src/App.jsx:39`), with team search and confidence filtering.

**The honest gap:** held-out accuracy is **61.55%** on the 85/15 chronological
test split (`MODEL_PARAMETERS.md:150`), against a >65% target. The model beats
the 50% coin flip but has not hit the bar that would make it interesting to
trade on. Calibration work (below) exists partly to understand why.

---

## 2. C++20 trading engine

**Status: working, paper-only; live socket path unverified.**

See **[`trading_engine/README.md`](trading_engine/README.md)** for full detail.

Maintains per-instrument order books from an authenticated, TLS-verified Kalshi
WebSocket feed, prices them against the model's published fair values, and
routes decisions through a risk gate into a simulated venue. Includes a
deterministic replay harness: recorded frames are re-run through the identical
ingest path and must produce byte-identical telemetry.

- **33 tests across 17 suites, all passing.**
- Failure-closed: refuses to trade on stale fair values, crossed books, or a
  tripped kill switch; malformed fair-value files never clobber good state.
- **Paper execution only** — `PaperVenue` crosses the spread against in-memory
  book depth with partial fills and weighted-average-cost realized P&L.

Known limits, stated plainly: the live WebSocket path has never been run against
a real Kalshi account; there is **no market-data recorder** (the replay fixture
is hand-authored); aggregate-exposure and order-rate limits are configured but
unenforced; fees gate signals but aren't deducted from P&L; and there is no
slippage or latency modelling.

---

## 3. Signal-research harness

**Status: working.**

`backend_ml/signal_research/` measures whether the model's probabilities are
trustworthy and whether they'd have had edge:

- **Calibration** — Brier score, log loss, reliability tables, and expected
  calibration error (`calibration.py:20-56`).
- **Recalibration** — isotonic regression and Platt scaling with held-out
  evaluation (`recalibration.py:65-93`).
- **Closing-line value** — captures Kalshi and sportsbook prices at fixed
  moments and compares entry prices to the closing line (`market_capture.py`,
  `clv.py`, `model_clv.py`), including model-Brier vs closing-price-Brier at
  settlement.
- **Leakage guards** — research modules are explicitly forbidden from importing
  the live prediction path, which would leak the future into the past
  (`signal_research/dataset.py:11-13`, `halflife_sweep.py:10-12`).

This is post-hoc analysis over captured snapshots, not a P&L simulator — it
answers "did the model have edge and are its probabilities honest," not "what
would this strategy have earned."

---

## Repository layout

```
benchwarmer-nba/
├── backend_ml/                # Python ML + research
│   ├── data_engine.py         # NBA data collection & causal feature engineering
│   ├── train_model.py         # Ensemble training (chronological split, GridSearchCV)
│   ├── predict.py             # Inference → Supabase
│   ├── elo_engine.py          # Sequential pre-game Elo ratings
│   ├── recency.py             # Exponential recency weights
│   ├── halflife_sweep.py      # Walk-forward half-life selection
│   ├── scheduled_retrain.py   # Drift-gated nightly retrain
│   ├── backtest.py            # Recent-window accuracy report
│   └── signal_research/       # Calibration, CLV, market capture, settlement
├── trading_engine/            # C++20 Kalshi engine (see its own README)
│   ├── src/                   # market_data, strategy, risk, execution, telemetry
│   ├── tests/                 # 33 GoogleTest tests
│   └── tools/paper_session.cpp# Offline replay driver
├── frontend_web/              # React + Vite + Tailwind dashboard
├── tests/                     # Cross-cutting Python tests
└── docs/                      # Specs, plans, project overview
```

## Tech stack

**ML/backend:** Python 3.10+, XGBoost, scikit-learn, pandas, NumPy, Supabase
**Trading engine:** C++20, Boost.Beast/Asio, OpenSSL, simdjson, nlohmann/json, GoogleTest, CMake
**Frontend:** React 18, Vite, Tailwind CSS, Supabase JS client

---

## Getting started

### Prerequisites
- Python 3.10+, Node.js 18+
- C++20 compiler, CMake ≥ 3.20, system OpenSSL and Boost (for the trading engine)
- A Supabase project (free tier is fine)

### Database
Run `SUPABASE_SCHEMA.sql` in your Supabase project's SQL editor, then note the
project URL and anon key.

### Backend
```bash
cd backend_ml
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cat > .env << 'EOF'
SUPABASE_URL=your_supabase_url
SUPABASE_KEY=your_supabase_service_key
EOF
```

### Frontend
```bash
cd frontend_web
npm install
cp .env.example .env    # add VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY
npm run dev             # http://localhost:3000
```

### Trading engine
```bash
cd trading_engine
cmake -S . -B build && cmake --build build -j
./build/te_tests        # 33 tests
```

### Running the tests
```bash
python -m pytest backend_ml tests -q     # 193 Python tests
cd trading_engine && ./build/te_tests    # 33 C++ tests
```

---

## Usage

**Train a model** (chronological split, grid search, writes the 4 artifacts):
```python
from train_model import train_and_optimize_model
train_and_optimize_model()
```

**Generate predictions** for a given day (`0` = today):
```python
from predict import predict_games
predict_games(day_offset=0)
```

**Replay the trading engine offline** (no network, no credentials):
```bash
cd trading_engine
./build/paper_session fair_values.json tests/fixtures/replay_sample.jsonl T
```

---

## Roadmap

### Phase 1: Data collection — ✅ complete
- [x] Project structure
- [x] NBA data fetching (`data_engine.py`, NBA Stats API)
- [x] Historical game data cached (13,182 games, 2015–2026)
- [x] Team and player statistics (incl. `player_impact_engine.py`)
- [x] Causal feature engineering (18 features; `shift(1)` before every window)

### Phase 2: Model development — ⚠️ built, target not met
- [x] Load and prepare training data
- [x] Train XGBoost baseline
- [x] Hyperparameter tuning (`GridSearchCV`, `TimeSeriesSplit`)
- [x] Evaluate on a chronological held-out split
- [x] Ensemble with Ridge; recency-weighted training with walk-forward selection
- [ ] **>65% accuracy target — currently 61.55%**

### Phase 3: Prediction pipeline — ⚠️ mostly complete
- [x] Load trained model for inference
- [x] Fetch today's games
- [x] Generate predictions
- [x] Store predictions in Supabase
- [x] Automated *retraining* (nightly, drift-gated)
- [ ] Automated *daily prediction* runs (retraining is scheduled; inference is still manual)

### Phase 4: Frontend integration — ⚠️ mostly complete
- [x] Connect frontend to Supabase
- [x] Display real predictions (live `game_predictions` query)
- [x] Working filters (team search, status, confidence threshold)
- [x] Confidence levels surfaced
- [ ] Prediction details modal

### Phase 5: Trading engine — ✅ v1 complete (paper-only)
- [x] Kalshi WebSocket ingest with TLS hostname verification and RSA-PSS auth
- [x] Order-book reconstruction from snapshot + incremental deltas
- [x] Strategy pipeline (arb, edge-taker, market-maker quoting)
- [x] Risk gate: order size, per-market position caps, daily-loss kill switch
- [x] Paper execution with partial fills and realized P&L
- [x] Deterministic replay harness (33 tests passing)
- [ ] Live path verified against a real Kalshi account
- [ ] Aggregate-exposure and order-rate limits enforced
- [ ] Market-data recorder (replay fixture is currently hand-authored)
- [ ] Two-legged arb execution (currently YES leg only)

### Phase 6: Signal research — ✅ core complete
- [x] Brier / log loss / reliability / ECE
- [x] Isotonic and Platt recalibration
- [x] Market capture and closing-line-value analysis
- [x] Settlement lookup and model-vs-closing-line comparison
- [ ] Sustained live capture across a full season

### Phase 7: Advanced — not started
- [ ] Real-time game updates
- [ ] Historical prediction tracking dashboard
- [ ] Model performance dashboard
- [ ] User accounts and favorites

---

## Performance targets

| Metric | Target | Current |
|---|---|---|
| Accuracy | >65% | **61.55%** (85/15 chronological split) |
| High-confidence accuracy | >70% at 75%+ confidence | not yet measured |
| Calibration | well-calibrated probabilities | measured via Brier/ECE; recalibration available |
| ROC-AUC | >0.70 | not yet at target |

## License

MIT — free to use for learning or personal projects.

---

**Status:** Trading engine and signal-research harness are functional
(paper-only, 33 + 193 tests passing). The NBA prediction pipeline runs end to
end but has not yet reached its accuracy target.

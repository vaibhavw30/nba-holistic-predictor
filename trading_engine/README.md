# trading_engine

A C++20 market-data ingest and paper-trading engine for [Kalshi](https://kalshi.com)
binary event markets, with a deterministic offline replay harness.

It maintains per-instrument order books from a live authenticated WebSocket feed,
prices them against fair values published by the NBA model, and routes the
resulting decisions through a risk gate into a simulated (paper) execution venue.
Every decision is emitted as one JSON line of telemetry.

**No real orders are ever placed.** The only execution path is `PaperVenue`
(`src/execution/paper_venue.cpp`), which simulates fills against the in-memory
book. There is no live order-entry venue in this codebase.

---

## Architecture

```
                   ┌───────────────── live path ─────────────────┐
Kalshi WSS ──TLS──▶ MarketDataGateway::run()      gateway_run.cpp:84
  (RSA-PSS           │  reconnect + backoff, resubscribe
   signed)           ▼
                  handle_raw(frame)               gateway.cpp:4
                     │
                     ├─▶ parse_ws_message()       kalshi_messages.cpp:17   (simdjson)
                     │      returns ParsedMsg by value — nothing mutated yet
                     ▼
                  OrderBook::apply_snapshot/apply_delta   order_book.cpp:3,8
                     │
                     ▼  UpdateCb
                  StrategyEngine::on_book_update  strategy_engine.cpp:8
                     │   kill switch ▸ fair-value/stale/crossed guards
                     ├─▶ detect_arb               arb.cpp:4
                     ├─▶ detect_take              edge_taker.cpp:3
                     └─▶ make_quote               market_maker.cpp:5   (logged only)
                            │
                            ▼
                     RiskManager::check           risk_manager.cpp:8
                            │
                            ▼
                     PaperVenue::place_against    paper_venue.cpp:6
                            │
                            ▼
                     Telemetry::event             telemetry.hpp:10   (JSON lines)

                   └───────────────── replay path ────────────────┘
recorded .jsonl ──▶ handle_raw(line)  ── identical components from here down
                    (test_replay.cpp:60, tools/paper_session.cpp:62)
```

The replay path re-enters the pipeline at exactly the same function the live
socket calls (`handle_raw`), so recorded frames exercise the real gateway, book,
strategy, risk and venue code — not a parallel test double.

---

## Ingest path

### Transport and authentication

`MarketDataGateway::run()` (`src/market_data/gateway_run.cpp:84`) opens a
TLS 1.2 WebSocket to `api.elections.kalshi.com/trade-api/ws/v2` using
Boost.Beast over Boost.Asio's SSL stream (`gateway_run.cpp:115`).

Certificate verification is deliberately strict: beyond `verify_peer` and the
default CA paths, an explicit hostname-verification callback is installed
(`gateway_run.cpp:112`), because CA-trust alone would accept any valid
certificate for *any* domain — an MITM gap. SNI is set at `gateway_run.cpp:120`.

Authentication is RSA-PSS/SHA-256 over a `timestamp + method + path` message,
base64-encoded and sent as WebSocket upgrade headers
(`KALSHI-ACCESS-KEY` / `-TIMESTAMP` / `-SIGNATURE`, `gateway_run.cpp:126-131`).
The signer is `KalshiSigner` (`src/market_data/kalshi_auth.hpp:6`), backed by
OpenSSL `EVP_PKEY`. Credentials come from `KALSHI_KEY_ID` and
`KALSHI_PRIVATE_KEY_PATH` (a PEM private key) — never from source.

After the handshake it subscribes to the `orderbook_delta` channel for the
configured watchlist (`gateway_run.cpp:135-141`) and enters a blocking read
loop, forwarding each text frame to `handle_raw` (`gateway_run.cpp:148-153`).

> **Not live-verified.** The file's own header comment
> (`gateway_run.cpp:3-8`) records that this path has never been run against a
> real Kalshi account — there were no credentials or network access in the
> environment where it was written. The parsing, book, strategy, risk and venue
> layers below are covered by tests; the socket itself is not.

### Parsing

`parse_ws_message` (`src/market_data/kalshi_messages.cpp:17`) uses **simdjson**
on-demand with a `thread_local` parser. It classifies each frame as
`Snapshot`, `Delta`, or `Other` (`kalshi_messages.hpp:5`) and extracts the
ticker, price levels, or `(side, price, delta)` triple.

### Applying incremental deltas

`OrderBook` (`src/market_data/order_book.cpp`) keeps two `std::map<Cents,int>`
price ladders — YES bids and NO bids — which is what Kalshi's binary markets
quote natively. Asks are derived by complement rather than stored:
`best_yes_ask() == 100 - best_no_bid()` (`order_book.cpp:19-21`).

- **Snapshot** (`order_book.cpp:3`) clears both ladders and repopulates,
  dropping any level with `qty <= 0`. A snapshot is a full state replacement.
- **Delta** (`order_book.cpp:8`) adds `delta_qty` to the level and **erases the
  price level entirely when the resulting quantity reaches zero or below**
  (`order_book.cpp:11`), so emptied levels never linger as phantom liquidity.

`crossed()` (`order_book.cpp:29`) reports whether the book is in the
inconsistent state `best_yes_bid >= best_yes_ask`, which the strategy layer
treats as untradeable (see below).

---

## Failure-closed behaviour

The engine degrades to *not trading* rather than to trading on bad state. Four
distinct mechanisms, each in a different layer:

**1. Parse-then-commit: a bad frame cannot half-mutate a book.**
`parse_ws_message` builds a complete `ParsedMsg` and returns it **by value**;
only afterwards does `handle_raw` apply it to the book
(`gateway.cpp:5-10`). A frame that fails to parse therefore never reaches
`apply_snapshot`/`apply_delta`, so there is no partially-applied book state to
recover from. Note the honest limit: `handle_raw` itself has **no** try/catch,
so a malformed frame that makes simdjson throw propagates out of it — in the
live path that exception is caught by the reconnect handler below; in the
replay path it would terminate the run.

**2. Fair-value swap is atomic, and malformed input is not clobbering.**
`FairValueProvider::load_from_file` (`src/fair_value/fair_value.cpp:14`) parses
into a **fresh** map, and on any exception returns early, explicitly
*retaining the existing map rather than clobbering it*
(`fair_value.cpp:25-27`). Rows with an unparseable `asof` timestamp are skipped
instead of being inserted with a bogus 0-epoch (`fair_value.cpp:21`). Parsing
happens **outside** the lock; only the `map_.swap(fresh)` is guarded
(`fair_value.cpp:28-31`), so the concurrent reader in the gateway thread can
never observe a half-mutated map. This matters because a background thread
reloads this file every `fair_value_refresh_secs` while trading is live
(`src/main.cpp:32-38`).

**3. Reconnect resnapshots rather than trusting stale books.**
Any connect/handshake/read exception is caught (`gateway_run.cpp:157`), logged,
and retried after a backoff that starts at 1s, doubles to a 30s cap, and resets
only after a connection gets far enough to complete a clean subscribe
(`gateway_run.cpp:96-98, 145, 166-169`). No explicit book-clearing is needed on
reconnect: a fresh session delivers a new `orderbook_snapshot` per ticker, which
replaces book state wholesale (`gateway_run.cpp:90-93`).

**4. The strategy refuses to act on suspect state.**
`StrategyEngine::on_book_update` (`strategy_engine.cpp:8`) returns without
trading — emitting a `skip` or `killed` telemetry event — when the kill switch
is tripped, the fair value is missing, the fair value is older than
`fair_value_max_age_secs`, or the book is crossed (`strategy_engine.cpp:9-24`).
`RiskManager::check` re-asserts the same conditions at the order gate and
returns `{allow=false}` with a reason string (`risk_manager.cpp:9-11`). The kill
switch trips either from a file flag polled every tick
(`risk_manager.hpp:24-27`) or automatically when cumulative realized P&L breaches
`max_daily_loss_cents` (`risk_manager.cpp:4-7`).

> **Scope caveat, stated in the source itself** (`risk_manager.hpp:9-13`):
> `max_aggregate_exposure_cents` and `orders_per_sec_budget` are loaded from
> config but **not enforced** in v1. Only `max_order_size`,
> `max_contracts_per_market` and `max_daily_loss_cents` gate orders. Do not read
> "failure-closed" as covering aggregate exposure or order rate.

---

## Deterministic replay harness

Recorded WebSocket frames are re-run through the identical ingest path, and the
resulting telemetry must be **byte-identical** across runs.

`tests/test_replay.cpp` reads `tests/fixtures/replay_sample.jsonl`
(`test_replay.cpp:52`), feeds each line to the real
`MarketDataGateway::handle_raw` (`test_replay.cpp:60-63`), and wires the
gateway's callback to a real `StrategyEngine` with a real `RiskManager`,
`PaperVenue` and `Telemetry` (`test_replay.cpp:39-48`). The whole replay runs
twice and the two telemetry strings are compared with `EXPECT_EQ`
(`test_replay.cpp:71-73`).

Determinism is achieved by removing the two nondeterministic inputs:

- **Fixed clock.** `kFixedNowMs` (`test_replay.cpp:19`) is passed as `now_ms`
  to every callback, so fair-value staleness never depends on wall-clock time.
- **Pinned fair value.** A fixed `replay_fv.json` is written and loaded before
  the run (`test_replay.cpp:33-37`).

The test also asserts the fixture actually exercised multiple strategy branches
— the log must contain `"type":"take"`, `"type":"quote"` and `"type":"skip"`
(`test_replay.cpp:77-79`) — so a passing test can't degenerate into
"empty == empty".

The same replay is available as a standalone binary, `paper_session`
(`tools/paper_session.cpp`), for comparing how a different fair value changes
paper decisions on identical market data:

```bash
./build/paper_session <fair_values.json> <fixture.jsonl> [ticker]
```

It prints the per-event telemetry followed by a summary line:

```json
{"type":"session_end","ticker":"T","position":50,"realized_pnl_cents":450}
```

> **What this is and isn't.** This is a deterministic replay *harness*, not a
> historical backtester. There is **no market-data recorder in this repo** — the
> single fixture (`tests/fixtures/replay_sample.jsonl`, 9 frames, one synthetic
> ticker `"T"`) is hand-authored, as its own test comment states
> (`test_replay.cpp:9`). There is no multi-day or multi-market driver, no
> historical data ingestion, and no performance reporting beyond end-of-run
> position and realized P&L.

---

## Execution and risk model

`PaperVenue` (`src/execution/paper_venue.cpp:6`) simulates marketable orders
only:

| Behaviour | Implementation |
|---|---|
| Fill price | Crosses the spread: Buy fills at `best_yes_ask()`, Sell at `best_yes_bid()` (`paper_venue.cpp:14-20`). Returns `"noliq"` if no touch exists. |
| Partial fills | Quantity capped at displayed touch liquidity: `qty = min(o.qty, avail)` (`paper_venue.cpp:25`). The unfilled remainder is dropped, not rested. |
| Position | Signed per-ticker integer (`paper_venue.hpp:28`), mirrored into `RiskManager` (`strategy_engine.cpp:40,60`). |
| Realized P&L | Weighted-average-cost accounting, booked only on the closing portion of a fill (`paper_venue.cpp:36-55`). |
| Sizing / limits | `max_order_size` and `max_contracts_per_market` clamp quantity; `max_daily_loss_cents` trips the kill switch (`risk_manager.cpp:12-18`, `:4-7`). |

Not modelled: unrealized/mark-to-market P&L, slippage or book-walking beyond the
depth cap, latency, resting/maker orders, and order time-in-force. Fees
(`fee_cents_per_contract`) are used **only** to raise the required edge before a
signal fires (`pricing.cpp:9-10`, `arb.cpp:6-16`) — they are **not** deducted
from realized P&L. Market-maker quotes are computed and logged but never
executed, as noted at `strategy_engine.cpp:69`. The arb path executes only the
YES leg, which the source flags as a naked directional fill rather than a
riskless lock (`strategy_engine.cpp:31-34`).

---

## Tests

**33 tests across 17 suites, all passing.** Counted from
`./build/te_tests --gtest_list_tests`, not estimated.

| Suite | Tests | Suite | Tests |
|---|---:|---|---:|
| `Risk` | 4 | `Pricing` | 2 |
| `OrderBook` | 3 | `EdgeTaker` | 2 |
| `Paper` | 3 | `MarketMaker` | 2 |
| `Parse` | 3 | `Auth` | 2 |
| `StrategyEngine` | 3 | `FairValue` | 2 |
| `Arb` | 1 | `Config` | 1 |
| `Gateway` | 1 | `MarketMap` | 1 |
| `Replay` | 1 | `Telemetry` | 1 |
| `Scaffold` | 1 | | |

```
[==========] 33 tests from 17 test suites ran. (15 ms total)
[  PASSED  ] 33 tests.
```

---

## Build

Requires a C++20 compiler, CMake ≥ 3.20, and a system OpenSSL and Boost
(header-only Beast/Asio are used, so no compiled Boost libraries are needed).
GoogleTest, nlohmann/json and simdjson are fetched automatically by CMake.

```bash
cd trading_engine
cmake -S . -B build
cmake --build build -j
```

### Running the tests

The replay test opens its fixture by a relative path, so the working directory
must be `trading_engine/` — which is what `gtest_discover_tests` sets
(`CMakeLists.txt:34`) and what running the binary directly from this directory
gives you:

```bash
cd trading_engine
./build/te_tests            # or: ctest --test-dir build --output-on-failure
```

### Running the replay harness offline

No network and no credentials required:

```bash
cd trading_engine
./build/paper_session fair_values.json tests/fixtures/replay_sample.jsonl T
```

### Running the live engine

Requires Kalshi credentials and reads `config/engine.json`,
`config/watchlist.json` and `fair_values.json` from the working directory:

```bash
export KALSHI_KEY_ID=...
export KALSHI_PRIVATE_KEY_PATH=/path/to/key.pem
./build/te_engine
```

Touch a file named `KILL` in the working directory to halt trading live
(`src/main.cpp:25`). Note there is no SIGINT handler wired yet
(`src/main.cpp:51`).

---

## Dependencies

| Dependency | Version | Source | Used for |
|---|---|---|---|
| GoogleTest | v1.15.2 | `FetchContent` (`CMakeLists.txt:7-9`) | Test suite |
| nlohmann/json | v3.11.3 | `FetchContent` (`CMakeLists.txt:10-12`) | Config, watchlist, fair values, telemetry, WS subscribe payload |
| simdjson | v3.10.1 | `FetchContent` (`CMakeLists.txt:13-15`) | Hot-path WebSocket frame parsing (`kalshi_messages.cpp:11`) |
| OpenSSL | system | `find_package` (`CMakeLists.txt:16`) | RSA-PSS request signing, TLS |
| Boost | system, header-only | `find_package` (`CMakeLists.txt:17`) | Beast WebSocket + Asio SSL transport (`gateway_run.cpp:36-42`) |

## Configuration

`config/engine.json` (loaded by `Config::load`, `src/core/config.hpp:15`):

| Key | Default | Enforced? |
|---|---:|---|
| `max_contracts_per_market` | 100 | yes |
| `max_order_size` | 25 | yes |
| `max_daily_loss_cents` | 20000 | yes |
| `max_aggregate_exposure_cents` | 500000 | **no — v1 gap** |
| `orders_per_sec_budget` | 5 | **no — v1 gap** |
| `fee_cents_per_contract` | 1 | signal threshold only, not deducted from P&L |
| `base_edge_cents` | 2 | yes |
| `confidence_k` | 8.0 | yes |
| `fair_value_refresh_secs` | 60 | yes |
| `fair_value_max_age_secs` | 1800 | yes |

## Known limitations

- Live WebSocket path is **not verified against a real account** (`gateway_run.cpp:3-8`).
- **No market-data recorder**; the sole replay fixture is hand-authored, 9 frames, one ticker.
- Aggregate-exposure and order-rate limits are configured but unenforced (`risk_manager.hpp:9-13`).
- Arb executes the YES leg only — directional, not a riskless lock (`strategy_engine.cpp:31-34`).
- Market-making quotes are logged, never executed (`strategy_engine.cpp:69`).
- No unrealized P&L, slippage, latency, resting orders, or time-in-force.
- Fees gate signals but are not deducted from realized P&L.
- `handle_raw` has no exception guard of its own (see Failure-closed §1).

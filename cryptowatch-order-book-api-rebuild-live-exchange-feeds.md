---
title: "The Cryptowatch Order Book API: What It Returned, Why It Went Dark, and How to Rebuild It on Live Exchange Feeds"
date: "2026-11-19"
description: "A field-by-field reconstruction of the retired Cryptowatch REST and WebSocket order book API — endpoints, seqNum semantics, the credit allowance model, and the exchange-native calls that reproduce each response today, with measured slippage walks on live $BTC depth."
tags: ["Cryptowatch", "Order Book", "Market Data", "Crypto APIs", "Market Microstructure", "FinTer", "Rust", "Trading Terminal", "Liquidity"]
---

# The Cryptowatch Order Book API: What It Returned, Why It Went Dark, and How to Rebuild It on Live Exchange Feeds

![Cryptowatch-style multi-exchange crypto order book terminal with stacked bid and ask depth ladders, price levels flowing from three exchange icons into one aggregated book, dark background #121613, neon green accents #C4E855, isometric technical illustration, clean minimal financial software aesthetic](https://image.pollinations.ai/prompt/Cryptowatch-style%20multi-exchange%20crypto%20order%20book%20terminal%20with%20stacked%20bid%20and%20ask%20depth%20ladders%2C%20price%20levels%20flowing%20from%20three%20exchange%20icons%20into%20one%20aggregated%20book%2C%20dark%20background%20%23121613%2C%20neon%20green%20accents%20%23C4E855%2C%20isometric%20technical%20illustration%2C%20clean%20minimal%20financial%20software%20aesthetic?width=800&height=400&nologo=true)

**Short answer:** Cryptowatch's market data API is gone. Kraken retired Cryptowatch Web, Desktop, Mobile, Social and the **Cryptowatch API** on 30 September 2023. The hosts `api.cryptowat.ch`, `docs.cryptowat.ch` and `stream.cryptowat.ch` now return no address record at all, and `cryptowat.ch` itself answers with a 301 to `kraken.com`. What survives is the *shape* of the data — a `[price, amount]` L2 ladder wrapped in a `result` envelope with a `seqNum` cursor and a credit `allowance` — and that shape can be reproduced against exchange-native endpoints in about sixty lines of code.

This article reconstructs the retired API from archived documentation, then rebuilds each piece on live feeds. It includes a working slippage-walk implementation, a discovery about depth truncation that costs real money if you get it wrong, and the design FinTer uses to hold a book in a native desktop process instead of polling a metered endpoint.

---

## What Cryptowatch Actually Was

Cryptowatch launched in 2014 as a browser-based multi-exchange charting terminal, founded by Artur Sapek. Kraken acquired it in March 2017 — the announcement cited "thousands of traders" charting "over 150 markets in real time" — and ran it for six more years before announcing the sunset. The SEC's November 2023 complaint against Kraken dates the retirement precisely: "From 2017 until it was retired on September 30, 2023, Kraken offered an application named Cryptowatch which it described as a 'premium trading terminal.'"

The reason traders still search for its API is specific. Cryptowatch did not merely chart exchanges; it **normalised** them. One endpoint pattern covered dozens of venues, one envelope format covered every response, and one order book schema covered spot and derivatives alike. When you write against thirty exchange APIs, you spend your life on schema drift. Cryptowatch sold you the abstraction.

That abstraction is what people are actually looking for when they search for `cryptowatch api orderbook` today — not the brand, but the uniform order book contract.

---

## The Retired Surface Area

From archived snapshots of `docs.cryptowat.ch` (February 2023), the market data REST API exposed this structure:

```text
https://api.cryptowat.ch
├── /markets/:exchange/:pair
│   ├── /price                 → { price }
│   ├── /summary               → 24h OHLC, volume, change
│   ├── /trades                → most recent trades
│   ├── /orderbook             → L2 snapshot + seqNum
│   ├── /orderbook/liquidity   → aggregated depth statistics
│   ├── /orderbook/calculator  → execution projection for a size
│   └── /ohlc                  → candles keyed by period seconds
├── /markets                   → all (exchange, pair) pairs
├── /exchanges                 → venues, each with a markets route
└── /assets, /pairs            → taxonomy
```

Three design decisions in that list are worth stealing. Exchanges have since borrowed the second one — Binance and Bybit both publish update IDs — but the first and third remain rare, and they are the two that change how you build.

### 1. The calculator endpoint

`/orderbook/calculator?amount=N` did not return a book. It returned an **execution projection** — what a market order of size N would actually cost, walked through the visible depth:

```bash
# Project cost to buy 50 BTC (archived docs example)
curl --silent -G https://api.cryptowat.ch/markets/binance/btcusdt/orderbook/calculator \
  -d 'amount=50' | jq ".result.buy.spend"

# Project average slippage, in basis points, for buying 150 BTC
curl --silent -G https://api.cryptowat.ch/markets/binance/btcusdt/orderbook/calculator \
  -d 'amount=150' | jq ".result.buy.avgDeltaBps"
```

The response fields were `buy.spend`, `buy.avgPrice`, `buy.avgDeltaBps` and the `sell` mirror (`proceeds`, `avgPrice`, `avgDeltaBps`). `avgDeltaBps` is the number that matters operationally: it is the average fill price expressed as basis points away from the top of book. A 150 BTC buy at `avgDeltaBps = 12` means your execution is 12 bps worse than the best ask — on $BTC at $63,900 that is roughly $77 per coin of immediate cost, before fees.

Exchanges give you the book. They do not give you the size-aware cost of taking it. That gap is why quants wrote wrappers around Cryptowatch.

### 2. The seqNum contract

Every order book response carried a `seqNum`, and the archived docs stated its purpose directly: *"Each response includes a seqNum which is intended for use with the WebSocket API; you can use this to resynchronize your order book and replay deltas received over the live feed which have a higher seqNum."*

The streaming side was split into three independent streams per market:

- `markets:<id>:book:snapshots` — full book broadcast **once per minute**
- `markets:<id>:book:deltas` — incremental price-level changes, **real-time**
- `markets:<id>:book:spread` — best bid and ask only

Deltas were expressed as operations, not absolute values:

```json
{
  "marketUpdate": {
    "market": { "exchangeId": "4", "currencyPairId": "180", "marketId": "87" },
    "orderBookDeltaUpdate": {
      "seqNum": 3922,
      "bids": { "removeStr": ["8048.4"] },
      "asks": { "set": [{ "priceStr": "8093.7", "amountStr": "0.8" }] }
    }
  }
}
```

`set` writes a level, `removeStr` deletes a price. A one-minute snapshot cadence plus real-time deltas is a bandwidth trade: the server sends the expensive full book twelve times an hour instead of on every update, and the client is responsible for applying deltas in sequence order.

### 3. The allowance model

Cryptowatch metered access in **credits**, not requests per second. Anonymous and free accounts received the equivalent of 10 credits per rolling 24-hour window. Per the archived rate-limit page:

| Endpoint | Credits per call |
| --- | --- |
| Price | 0.005 |
| Trades | 0.01 |
| Order Book | 0.01 |
| Order Book Calculator | 0.015 |
| OHLC Candlesticks | 0.015 |

Every response echoed a self-report, so a client could throttle itself without guessing:

```json
{ "allowance": { "cost": 0.003, "remaining": 9.572 } }
```

Exhausting the allowance returned HTTP 429. This is a billing model dressed as rate limiting, and it is the single most important reason a *local-first* terminal will beat any metered API for order book work: a book that updates 20 times per second is not a request you want to pay for per call.

---

## Reproducing the Order Book Response

Here is what the retired endpoint returned, reconstructed from archived examples:

```json
{
  "result": {
    "asks": [[10340.01, 3.09216998], [10341.01, 6.714], [10341.29, 0.14237424]],
    "bids": [[10340.00, 0.81266905], [10339.47, 0.0181], [10339.00, 0.0027]],
    "seqNum": 2326491
  },
  "allowance": { "cost": 0.01, "remaining": 9.98 }
}
```

Asks ascending, bids descending, `[price, amount]` per level, `seqNum` as the cursor. Now the same structure against three live venues, verified today:

| Concern | Cryptowatch | Binance spot | Bybit v5 | Kraken |
| --- | --- | --- | --- | --- |
| Endpoint | `/markets/:ex/:pair/orderbook` | `GET /api/v3/depth` | `GET /v5/market/orderbook` | `GET /0/public/Depth` |
| Max visible depth | no stated cap (`limit`/`span`/`depth` params) | **5,000 levels/side** | **1,000 levels/side** | `count` param, **silently falls back to 100 above 500** |
| Cost model | 0.01 credits/call | request *weight*: 5 / 25 / 50 / 250 by limit band | rate limit per IP | rate limit per IP |
| Sequence cursor | `seqNum` | `lastUpdateId` | `u` (update id), `seq`, `cts` | none in public Depth |
| Side ordering | asks asc, bids desc | asks asc, bids desc | asks asc, bids desc | asks asc, bids desc |

Two things fall out of that table immediately.

**Depth caps are not uniform.** Bybit tops out at 1,000 levels per side for spot and contracts; Binance returns up to 5,000, and its documentation is explicit that a larger `limit` is silently clamped: *"If limit > 5000, only 5000 entries will be returned."* Kraken's public `Depth` is stranger: `count=500` returns 500 levels per side, but `count=1000` and `count=2000` both silently fall back to **100** levels per side — verified against the live endpoint while writing this. A client that asks for "deep" Kraken depth and does not check the returned array length will quietly operate on a book one-tenth the size it believes it has. Kraken's levels also arrive as `[price, volume, timestamp]` triples, not pairs. Any normalisation layer has to handle all of this.

**Only some venues give you a resync cursor.** Binance and Bybit expose update IDs that let you reconcile a REST snapshot against a WebSocket diff stream. Kraken's public `Depth` does not, which means Kraken books must be maintained purely from the socket, or rebuilt on a timer.

---

## The Calculator, Reimplemented

The projection maths is a single walk down the ladder. This implementation is deliberately honest about partial fills, because a book that cannot fill your size should say so rather than return a number that implies it did:

```python
def walk(levels, amount):
    """Walk price levels until `amount` base units are filled.

    Returns (notional, filled, worst_price). Partial fills are reported,
    not silently treated as complete.
    """
    remaining = amount
    notional = 0.0
    worst = None
    for price, qty in levels:
        take = min(remaining, qty)
        notional += take * price
        remaining -= take
        worst = price
        if remaining <= 1e-12:
            break
    return notional, amount - remaining, worst


def project(bids, asks, amount):
    best_bid, best_ask = bids[0][0], asks[0][0]
    buy_notional, buy_filled, _ = walk(asks, amount)
    sell_notional, sell_filled, _ = walk(bids, amount)

    buy_avg = buy_notional / buy_filled if buy_filled else float("nan")
    sell_avg = sell_notional / sell_filled if sell_filled else float("nan")

    # Cryptowatch's avgDeltaBps: slippage against the top of book.
    buy_bps = (buy_avg - best_ask) / best_ask * 10_000
    sell_bps = (best_bid - sell_avg) / best_bid * 10_000
    return buy_avg, buy_bps, sell_avg, sell_bps
```

Run against live snapshots captured on 19 November 2026, projecting a **10 BTC market order** — deliberately small, because that is where the interesting behaviour lives:

```text
binance BTCUSDT (limit=5000)
  levels        bid=5000 ask=5000
  best bid/ask  63,909.42 / 63,909.43
  spread        0.01  (0.002 bps)
  buy  10       spend=639,179.90 avg=63,917.99 slip=1.339 bps
  sell 10       proceeds=639,088.71 avg=63,908.87 slip=0.086 bps

bybit BTCUSDT spot (limit=200)
  levels        bid=200 ask=200
  best bid/ask  63,912.50 / 63,912.60
  spread        0.10  (0.016 bps)
  buy  10       spend=639,452.70 avg=63,945.27 slip=5.112 bps
  sell 10       proceeds=638,924.88 avg=63,892.49 slip=3.131 bps

kraken XXBTZUSD (count=500)
  best bid/ask  63,834.70 / 63,834.80
  buy  10       avg=63,856.88 slip=3.459 bps
  sell 10       avg=63,823.51 slip=1.752 bps
```

A caveat that the numbers deserve: these three snapshots were taken seconds apart and the venues were genuinely trading at different prices ($63,909 / $63,912 / $63,835). This is **not** a like-for-like venue comparison — it is three illustrations of the same computation. The intra-snapshot asymmetric result on Binance is real, though: buying 10 BTC cost 1.339 bps of slippage while selling cost only 0.086 bps in the same instant, because the ask side of that particular book was thinner near the touch. A terminal that shows only mid price hides this entirely.

---

## The Depth Truncation Trap

Here is the part that surprises people. Take **one** Binance snapshot, hold price and time constant, and replay the same 10 BTC order against progressively truncated books:

```text
snapshot lastUpdateId=103180597199  full depth=5000x5000 levels
best bid/ask 63792.65 / 63792.66  spread=0.01

 visible levels      buy avg  buy slip bps     sell avg  sell slip bps  buy filled
             10    63,792.77         0.018    63,792.48          0.027        2.27
             20    63,793.17         0.080    63,791.90          0.118        2.65
             50    63,795.99         0.522    63,787.25          0.846       10.00
            100    63,795.99         0.522    63,786.95          0.893       10.00
           1000    63,795.99         0.522    63,786.95          0.893       10.00
```

Read the `buy filled` column. Truncated to the top **10 levels** — the number of rows a small order book pane shows when it is fitted to a short window — a 10 BTC market order **cannot be filled at all**. Only 2.27 BTC of the requested size exists in that window. The reported slippage of 0.018 bps is not a price; it is an artefact of measuring a 2.27 BTC fill as if it were a 10 BTC fill. Any widget that computes slippage without checking whether the walk completed will report a number like this one with total confidence.

At 50 levels the order completes, and from 100 levels onward the projection is **bit-identical**. Ten times more depth buys exactly zero additional information for this order size.

That has a clean operational consequence: *depth display is a sizing decision, not a fidelity decision.* You need enough levels that the largest order you would ever send can complete, and not one row more — extra levels cost bandwidth and render time for output that will never be consumed. Cryptowatch's calculator endpoint existed precisely because a book widget showing 30 rows tells you nothing about whether your size fits.

---

## Maintaining a Book Between Snapshots

A REST snapshot is a photograph. A book that stays correct is a video. The reconciliation pattern is the same on every venue that publishes an update ID, and Binance documents it as the canonical local order book procedure:

```text
1. Open the diff stream first, buffer every event.
2. Fetch the REST snapshot.
3. Drop buffered events whose u <= lastUpdateId.
4. First valid event must satisfy:  U <= lastUpdateId + 1  <=  u
5. Apply: for each bid/ask, quantity 0 means delete that price level,
   otherwise overwrite the level's absolute quantity.
6. Track the previous event's u and require the next event's pu to equal it.
7. On any gap: discard the local book, refetch the snapshot, restart at 1.
```

The critical detail is step 5: these are **absolute** quantities, not deltas. Exchange diff streams tell you what the resting size at a price *is*, not how much it changed. Treating them as increments is the classic bug, and it silently produces a book that looks plausible while being wrong.

Step 7 matters more than it looks. A book that has silently desynchronised is worse than no book, because every downstream calculation — mid price, spread, imbalance, slippage projection — inherits the error without any signal that something is off. Resynchronising aggressively on gap detection is the only safe policy.

This is also where Cryptowatch's once-per-minute snapshot cadence looks archaic. Modern exchange diff streams are fast enough that a client which fetches one snapshot at startup and then applies diffs stays correct indefinitely — provided it handles sequence gaps and reconnects. The snapshot is a bootstrap, not a heartbeat.

---

## How FinTer Holds a Book Locally

[FinTer](https://github.com/coin-watch/FinTer) — Coinwatch's open-source native terminal in Rust on GPUI — takes the opposite architectural position to a metered API: the book is a local object, replaced wholesale by the newest snapshot, and never fetched per render.

The gateway client parses a normalised payload into a canonical type:

```rust
struct GatewayOrderBook {
    bids: Vec<[f64; 2]>,      // [price, quantity], best first
    asks: Vec<[f64; 2]>,
    timestamp: Option<i64>,
}
```

That becomes an `OrderBook` — `exchange`, `symbol`, `bids: Vec<PriceLevel>`, `asks: Vec<PriceLevel>`, `timestamp_ms` — with the derived quantities the calculator needs already available as methods:

```rust
pub fn best_bid(&self) -> Option<f64> { self.bids.first().map(|l| l.price) }
pub fn best_ask(&self) -> Option<f64> { self.asks.first().map(|l| l.price) }

pub fn spread(&self) -> Option<f64> {
    match (self.best_bid(), self.best_ask()) {
        (Some(bid), Some(ask)) => Some(ask - bid),
        _ => None,
    }
}

pub fn mid_price(&self) -> Option<f64> {
    match (self.best_bid(), self.best_ask()) {
        (Some(bid), Some(ask)) => Some((bid + ask) / 2.0),
        _ => None,
    }
}
```

The delivery path is where the design shows. Rather than queueing every book, FinTer keeps **only the latest**:

```text
broadcast::Receiver<Arc<OrderBook>>
      │  (Tokio background task)
      ▼
Arc<Mutex<Option<Arc<OrderBook>>>>   (latest snapshot only)
      │  (GPUI render thread, on change)
      ▼
OrderBookPanel::render()
```

The drain loop overwrites a single slot on every message. Its lag handling is explicit rather than fatal:

```rust
Err(broadcast::error::RecvError::Lagged(n)) => {
    warn!("OrderBookFeed: lagged by {n} snapshots — some data was lost");
    // Continue — do not exit on lag.
}
```

Dropping intermediate books is the *correct* behaviour for a depth ladder. A trader needs to see the current state of the book, not a replay of every state it passed through. Queueing them all would build latency into the display in exchange for information nobody consumes — the same reasoning that makes 100 depth levels no better than 50 once your order already fits.

The panel then polls on a 100 ms timer and does the cheapest possible change check, comparing `Arc` pointers rather than book contents:

```rust
// OrderBookPanel::new — 100 ms timer, weak entity so the task ends on close
let latest = feed_for_timer.snapshot();
let changed = match (&last, &latest) {
    (Some(previous), Some(current)) => !std::sync::Arc::ptr_eq(previous, current),
    (None, None) => false,
    _ => true,
};
if this.update(cx, |_, cx| {
    if changed || credit_changed { cx.notify(); }
}).is_err() { break; }
```

Since the drain loop replaces the `Arc` rather than mutating in place, pointer inequality is a complete and free "did the book change" test. Ten times per second at most, and the render thread never blocks on the network.

The broadcast channel that feeds it holds 64 slots (`OB_BROADCAST_CAPACITY = 64`), against 1,024 for ticks and trades — a deliberate split. A dropped tick is invisible; a dropped book is not, because the *next* snapshot fully replaces it. That difference is why the book channel can be small and lossy while the trade channel is large and near-lossless.

---

## What the Migration Actually Costs

Anyone replacing a Cryptowatch integration should budget for four kinds of work, in ascending order of unpleasantness:

1. **Schema normalisation.** Three venues, three shapes. Kraken returns triples, Binance and Bybit return pairs. Symbol conventions disagree (`BTCUSDT`, `BTC/USDT`, `XXBTZUSD`) — FinTer's `normalize_symbol` strips separators and case, and treats `BINANCE:BTCUSDT`, `BTC/USDT` and `BTCUSDT` as the same instrument for mark-price purposes.
2. **Depth caps and cost tiers.** Binance charges request weight in bands: 5 weight for 1–100 levels, 25 for 101–500, 50 for 501–1,000, and **250** for 1,001–5,000. Asking for the full book is 50× the weight of asking for a hundred levels — and as measured above, often returns identical projections.
3. **Sequence handling.** Venues with cursors need the buffer-snapshot-replay dance; venues without them need trusted socket-only maintenance or timed refresh.
4. **Rate limits that are not credits.** The allowance model was uniform and legible. Per-IP weight buckets, per-endpoint limits and connection caps are not, and they differ per venue.

What you *gain* by moving off the retired API is the thing Cryptowatch could never give you: unmetered local computation. The calculator walk is microseconds of arithmetic. In FinTer it runs wherever you can reach an `OrderBook` — including from the built-in agent, which can read the top of book through a typed snapshot and reason about depth without a network round-trip to a billing service.

---

## Build and Run FinTer

FinTer is MIT-licensed and builds from source. Requirements are a stable Rust toolchain with Cargo plus the platform libraries GPUI needs.

**Ubuntu / Debian dependencies:**

```sh
sudo apt-get update
sudo apt-get install -y \
  libxkbcommon-dev libxkbcommon-x11-dev \
  libwayland-dev libvulkan-dev \
  libxcb-render0-dev libxcb-shape0-dev libxcb-xfixes0-dev \
  libssl-dev libdbus-1-dev pkg-config
```

**Clone and run:**

```sh
git clone https://github.com/coin-watch/FinTer.git
cd FinTer
cargo run --release
```

**Verify a checkout before sending a change:**

```sh
cargo test --workspace
cargo fmt --check
cargo clippy -- -D warnings
```

Guest mode needs no account and no backend. As of commit `48e4008`, the market WebSocket is derived from the API origin rather than a separate `ws.` host — `wss://<api-origin>/ws` — so REST and streaming share one origin and one configuration value. `FINTER_WS_URL` remains as an explicit override for local gateway work.

---

## Conclusion

The Cryptowatch API is not coming back, and the parts people miss most were never really about the endpoints. They were about a **uniform order book contract**: one schema, one sequence cursor, one way to ask "what would this order actually cost me."

Every piece of that contract is reproducible on live exchange feeds — the ladder, the cursor, the projection walk. The projection walk in particular is trivial arithmetic that belongs in your process, not in a metered request, and running it locally is what makes it usable twenty times a second instead of once per credit.

The one measurement worth carrying away: on a single Binance BTCUSDT snapshot, a ten-level book could fill **2.27 of 10 BTC**, while anything from fifty levels up produced an identical projection. Depth is a sizing parameter. Configure it for the largest order you would send, then stop paying for rows nothing will ever read.

Explore the code, star the repository, or contribute:

- **GitHub Repository:** [https://github.com/coin-watch/FinTer](https://github.com/coin-watch/FinTer)
- **Download & Live Market App:** [https://coinwat.ch](https://coinwat.ch)
- **Architecture Deep Dive:** [FinTer Architecture: Building a High-Performance, Local-First Crypto Market Terminal in Rust and GPUI](https://coinwat.ch/blog/finter-rust-gpui-crypto-terminal-architecture)
- **Storage & Sync Deep Dive:** [Local-First Dashboards in FinTer: File-Backed Layouts, Last-Write-Wins Sync, and Agent-Controlled Pane Grids](https://coinwat.ch/blog/finter-local-first-dashboards-file-sync-agent-pane-control)
- **Security Deep Dive:** [Securing Crypto Exchange API Keys at Rest: AES-256-GCM, OS Keyrings, and In-Chat Charting in FinTer](https://coinwat.ch/blog/finter-crypto-terminal-api-security-aes-256-gcm-rust)
- **Related Market Analysis:** [Order Flow Imbalance (OFI) in Crypto Microstructure](https://coinwat.ch/blog/order-flow-imbalance-crypto-microstructure-algorithmic-trading), [MEV Bots and Crypto DEX Arbitrage](https://coinwat.ch/blog/mev-bots-crypto-arbitrage-dex), and [The Comprehensive Data Toolkit for Algorithmic Trading](https://coinwat.ch/blog/comprehensive-crypto-data-toolkit-algo-trading-en)

---

## Sources & Technical References

1. [Kraken — "Cryptowatch to sunset: Kraken Pro to integrate Cryptowatch features"](https://blog.kraken.com/product/cryptowatch-to-sunset-kraken-pro-to-integrate-cryptowatch-features) — official sunset announcement covering Cryptowatch Web, Desktop, Mobile, Social and API.
2. [SEC v. Payward, Inc. and Payward Ventures, Inc., Complaint (N.D. Cal., Nov. 2023)](https://www.sec.gov/files/litigation/complaints/2023/comp-pr2023-237.pdf) — states Cryptowatch was retired on 30 September 2023.
3. [Kraken — "Kraken Acquires Cryptowatch and Launches New Trade Platform" (1 March 2017)](https://blog.kraken.com/news/kraken-acquires-cryptowatch-and-launches-new-trade) — acquisition, founder, and scale at time of sale.
4. Archived Cryptowatch documentation, `docs.cryptowat.ch` snapshots of 3 February 2023: [Order Book](https://web.archive.org/web/20230203224919/https://docs.cryptowat.ch/rest-api/markets/order-book), [Rate Limit](https://web.archive.org/web/20230203224919/https://docs.cryptowat.ch/rest-api/rate-limit), [OHLC Candlesticks](https://web.archive.org/web/20230203224919/https://docs.cryptowat.ch/rest-api/markets/ohlc), [WebSocket Order Books](https://web.archive.org/web/20230203224919/https://docs.cryptowat.ch/websocket-api/data-subscriptions/order-books).
5. [Binance Spot REST API — Market Data Endpoints (`/api/v3/depth`)](https://developers.binance.com/docs/binance-spot-api-docs/rest-api/market-data-endpoints) — 5,000-level cap, request-weight bands.
6. [Binance Derivatives — How to manage a local order book correctly](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/How-to-manage-a-local-order-book-correctly) — snapshot/diff reconciliation procedure.
7. [Bybit V5 — Get Orderbook](https://bybit-exchange.github.io/docs/v5/market/orderbook) — 1,000-level cap, `u`/`seq`/`cts` fields.
8. [Kraken Public API — Get Depth](https://docs.kraken.com/api/docs/rest-api/get-order-book) — `count` parameter and `[price, volume, timestamp]` level format.
9. [FinTer GitHub Repository — coin-watch/FinTer](https://github.com/coin-watch/FinTer) — source for `src/data/bridge.rs`, `src/data/orderbook.rs`, `src/data/orderbook_feed.rs`, `src/data/trade_store.rs`, `src/components/ui/orderbook_panel.rs`, and `README.md`.
10. [GPUI Framework — Zed Industries](https://gpui.rs/) and [Tokio](https://tokio.rs/) — the runtime and UI substrate FinTer is built on.

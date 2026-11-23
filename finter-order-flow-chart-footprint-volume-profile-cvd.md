# Building an Order-Flow Chart in Rust: Footprint, Volume Profile, CVD, and a Liquidity Heatmap in FinTer

![FinTer order-flow chart engine with footprint bid x ask cells, a right-aligned volume profile histogram, a cyan delta and CVD subpane, and translucent liquidity heatmap bands behind candles, dark background #121613, neon green accents #C4E855, isometric technical illustration of layered chart panes, clean minimal financial software aesthetic](https://image.pollinations.ai/prompt/FinTer%20order-flow%20chart%20engine%20with%20footprint%20bid%20x%20ask%20cells%2C%20a%20right-aligned%20volume%20profile%20histogram%2C%20a%20cyan%20delta%20and%20CVD%20subpane%2C%20and%20translucent%20liquidity%20heatmap%20bands%20behind%20candles%2C%20dark%20background%20%23121613%2C%20neon%20green%20accents%20%23C4E855%2C%20isometric%20technical%20illustration%20of%20layered%20chart%20panes%2C%20clean%20minimal%20financial%20software%20aesthetic?width=800&height=400&nologo=true)

Six commits landed in **[FinTer](https://github.com/coin-watch/FinTer)** on 23 November 2026 that turn a candlestick chart into a full order-flow workstation: a real-time delta and cumulative-delta subpane, a bid × ask footprint, a right-aligned volume profile with point of control and value area, a resting-liquidity heatmap drawn behind the candles, order-book level grouping with wall detection, and a floating OHLCV card that follows the crosshair. Together they add 1,662 lines across 12 files.

This is an engineering account of how each piece is built — the threading model that keeps the render loop free of per-trade work, the integer price keys that make footprint aggregation safe, the value-area algorithm and where it diverges from the CME-style reference, and the four bugs the design deliberately avoids. Every claim below is traceable to a commit, a file, or a struct in the repository.

If you want the quantitative theory behind these signals — why cumulative delta is a one-dimensional measure and what order flow imbalance adds — the companion piece on [order flow imbalance and microstructure](https://coinwat.ch/blog/order-flow-imbalance-crypto-microstructure-algorithmic-trading) covers Cont, Kukanov and Stoikov's formulation. This article is about the other half: making the numbers appear at 60 frames per second without stalling the tape.

---

## What Shipped

| Commit | Subject | Primary files |
|---|---|---|
| `a2022d9` | Add real-time delta and CVD order-flow panel | `src/data/orderflow_feed.rs`, `src/components/ui/chart_panel.rs` |
| `e588beb` | Label the order-flow chart subpane | `src/components/ui/chart_panel.rs` |
| `b932a70` | Add footprint, volume profile, and order-flow chart controls | `src/chart/model.rs`, `src/data/orderflow_feed.rs` |
| `f87a8ec` | Add liquidity heatmap to the order-flow chart | `src/data/liquidity_feed.rs` |
| `10f702b` | Add order-book grouping, imbalance, and wall highlights | `src/components/ui/orderbook_panel.rs` |
| `939e126` | Add crosshair data card and chart-type menu | `gpui-component-patched/src/chart/candlestick_chart.rs` |

Four of these are visible features. One of them — `a2022d9` — is mostly plumbing, and it is the one that determines whether the other three are usable on a busy $BTC pair.

---

## The Core Constraint: The Render Thread Never Touches a Trade

A trade tape on a liquid venue delivers thousands of messages per second. A naive implementation wires the socket handler directly to a UI redraw: every trade mutates chart state and requests a frame. That works until it doesn't, and the failure mode is the ugliest one in trading software — the display silently falls behind the market, so you are reading a chart that is 400 ms stale and does not say so.

FinTer's answer is a three-layer split encoded in `src/data/orderflow_feed.rs`. The module's own doc comment states the contract: *"The bridge delivers trades on a broadcast channel. This feed drains them on a Tokio task, buckets them by bar and aggressor side, and exposes an immutable snapshot to the GPUI render thread. The render thread never does per-trade work."*

The three layers:

**1. Ingest, off the render thread.** `OrderFlowFeed::new(receiver, timeframe_ms)` takes a `tokio::sync::broadcast::Receiver<Arc<UnifiedTick>>` and spawns `drain_loop` on the shared Tokio runtime obtained from `crate::data::client::tokio_runtime()`. The loop is a two-arm `tokio::select!`: one arm awaits `cancel.cancelled()`, the other `receiver.recv()`. Each `Ok(trade)` takes the state mutex, calls `State::ingest`, and bumps a version counter. Nothing here knows what a frame is.

**2. State behind an `Arc<Mutex<State>>`.** The state is a `BTreeMap<i64, FlowBar>` keyed by bar open time, a `HashSet<String>` of seen trade IDs, and a `VecDeque<String>` of insertion order for eviction. The `BTreeMap` is not incidental: `snapshot()` walks `self.bars.values()` and gets ascending timestamps for free, which is exactly the order the chart's series need.

**3. A monotonic version counter as the only cross-thread signal.** `version: Arc<AtomicU64>` is incremented with `Ordering::Relaxed` on every ingest and on every lag event. The render side reads it and compares. That is the entire synchronization protocol — no channel from the data task back to the UI, no wake-up per trade.

The wiring in `src/components/ui/pane_grid/renderer.rs` shows how a pane gets one:

```rust
let orderflow = crate::data::orderflow_feed::OrderFlowFeed::new(
    trade_rx,
    crate::data::bridge::BridgeConfig::timeframe_to_ms(&timeframe),
);
let liquidity = crate::data::liquidity_feed::LiquidityFeed::new(book_rx);
```

And `ChartPanel::set_orderflow_feed` turns the counter into repaints with a 250 ms poll:

```rust
let mut last = u64::MAX;
loop {
    let failed = this.update(cx, |panel, cx| {
        let current = version.load(std::sync::atomic::Ordering::Relaxed);
        if current != last {
            last = current;
            if panel.show_delta || panel.show_footprint {
                cx.notify();
            }
        }
    }).is_err();
    if failed { break; }
    cx.background_executor().timer(Duration::from_millis(250)).await;
}
```

Two decisions in five lines are worth naming. First, the repaint is **capped at four per second** regardless of trade rate: the counter can advance a thousand times between polls and the UI still draws one frame. A chart pane is a few hundred pixels wide; there is no information in the 40th update within a 250 ms window that the 41st does not carry. Second, `cx.notify()` is gated on `show_delta || show_footprint`. A user who has both panels off pays the atomic load and nothing else — the feed keeps aggregating, because `snapshot()` is cheap and the user may switch the pane on at any moment, but no frames are requested.

This is the same lesson that a commit from the previous week (`a013314`, *perf(tape): stop throttling trades on the global TradeStore mutex*) paid for in measurement rather than design. That commit message records the numbers: `set_mark_price` was doing an `iter().find()` plus a `retain()` over a map keyed by the already-canonical symbol, allocating one `String` per stored symbol per trade. At 128 symbols that measured **~30 µs/trade**, capping the tape at **~33,000 trades/s** independent of network speed; the replacement `get()` measures **~0.29 µs**. The fix also moved the `TradeStore` lock outside the parse loop — build the batch, lock once.

The order-flow feed avoids that entire class of problem by not sharing a lock with anything on the hot path. Its mutex is private to the feed and held only for the duration of one `ingest` call, which is a handful of arithmetic operations and two map lookups.

---

## Bucketing Trades Into Bars

`State::ingest` is where the correctness lives. It is 55 lines and nearly every one of them defends against a specific real-world input.

**Validation first.** A trade with `timestamp_ms <= 0`, a non-finite price or amount, or a non-positive price or amount is dropped. This matters more than it looks: an exchange that sends a zero-quantity trade during a book reset, or a `NaN` price from a malformed decimal parse, would otherwise insert a bar at timestamp 0 or poison a `f64` sum permanently. `NaN` propagates through addition and never leaves.

**Dedupe with a bounded FIFO.** When the venue supplies a trade ID, the feed inserts it into a `HashSet<String>`. A duplicate returns immediately. The `VecDeque<String>` tracks insertion order so that when the set exceeds `MAX_SEEN_TRADE_IDS = 20_000`, the oldest ID is popped from both structures. This is what makes reconnect snapshots safe: a re-subscription that replays the last 100 trades does not double-count them, and the dedupe cost stays bounded at 20,000 entries instead of growing with session length.

**Floor division, not truncation.** The bar boundary is:

```rust
let timeframe_ms = timeframe_ms.max(1_000);
let bar_ts = trade.timestamp_ms.div_euclid(timeframe_ms) * timeframe_ms;
```

`div_euclid` rather than `/` is deliberate. Rust's `/` truncates toward zero, so a timestamp of `-1` would divide to `0` and bucket into bar 0 rather than bar `-timeframe`. The unit test `buckets_buy_sell_and_skips_unknown_side` pins the ordinary path; `dedupes_trade_ids_and_allows_late_bars` pins the odd one by feeding a trade at `timestamp_ms = 1_000` with a 60-second timeframe and asserting the bar lands at timestamp `0` with the sell volume intact. Late-arriving trades are accepted into their historical bar rather than dropped — a bar map is not a stream cursor.

**Signed volume with a third state.** The amount is signed by aggressor side, and here the codebase made a change that propagates beyond this module:

```rust
let amount = match trade.side {
    TradeSide::Buy => amount_with_positive(trade.amount),
    TradeSide::Sell => -amount_with_positive(trade.amount),
    TradeSide::Unknown => return,
};
```

`TradeSide::Unknown` returns. It does not default to sell. That is not a stylistic preference — the same commit (`a2022d9`) went through `src/components/ui/tape_panel.rs` and replaced every `is_buy()` boolean test with an exhaustive `match`:

```rust
match t.side {
    TradeSide::Buy => buy_vol += val,
    TradeSide::Sell => sell_vol += val,
    TradeSide::Unknown => {}
}
```

The old shape was `if t.side.is_buy() { buy } else { sell }`, which folds "we don't know" into "sell". On a venue or a frame where the side field is absent or unrecognised, that silently books every ambiguous trade as aggressive selling — and delta is exactly the metric that would hide the error, because it would look like persistent seller pressure rather than a parsing gap. The tape now renders those rows with a muted `—` instead of a false `SELL`, and `classify_audio_batch` uses `find_map` to skip unknowns when choosing the buy/sell tone.

The side itself arrives from the gateway as a string and is normalised in `src/data/bridge.rs`:

```rust
let raw_side = trade.side.as_deref().unwrap_or("").to_lowercase();
let side = if raw_side.contains("buy") || raw_side == "b" || raw_side == "bid" {
    TradeSide::Buy
} else if raw_side.contains("sell") || raw_side == "s" || raw_side == "ask" {
    TradeSide::Sell
} else {
    TradeSide::Unknown
};
```

The trap this normalisation sits on top of is documented by the venues themselves. Binance's aggregate trade stream carries `"m": true // Is the buyer the market maker?` — that is, the flag describes the *maker*, while a delta calculation needs the *taker*. Getting the polarity backwards inverts the entire signal while leaving a plausible-looking chart, which is why the field is normalised once at the bridge boundary into an explicit three-state enum rather than consumed as a boolean at each call site.

**Per-price rows.** Every trade also updates a price-level accumulator inside its bar:

```rust
let price_key = (trade.price * 1_000_000.0).round() as i64;
let price_row = bar.price_rows.entry(price_key).or_insert(PriceFlow {
    price: trade.price,
    ..PriceFlow::default()
});
```

`FlowBar::price_rows` is a `BTreeMap<i64, PriceFlow>` keyed by **price in millionths as an integer**, not by `f64`. This solves three problems at once. `f64` is not `Ord`, so it cannot be a `BTreeMap` key at all without a wrapper. Float keys that are equal in intent but differ in the last bit — `67000.1` arriving via two different parse paths — would create two rows for one price level. And an integer key gives a deterministic, total order, so iterating `price_rows.values()` yields prices ascending, which is the order the footprint and profile renderers both want.

The `PriceFlow` struct carries `price`, `buy_volume`, `sell_volume`, and two accessors:

```rust
pub fn delta(&self) -> f64 { self.buy_volume - self.sell_volume }
pub fn volume(&self) -> f64 { self.buy_volume + self.sell_volume }
```

`FlowBar` exposes the same two methods over its totals, so a bar-level delta and a price-level delta are the same expression at different granularities — which is what makes the profile's per-row delta internally consistent with the delta subpane.

**Bounded retention.** `MAX_BARS = 5_000`; the oldest key is removed whenever the map exceeds it. At a one-minute timeframe that is roughly 3.5 days of continuous trading. The eviction loop pulls `self.bars.keys().next().copied()` — the smallest key, i.e. the oldest bar — which is only cheap because the structure is ordered.

---

## Delta, CVD, and the `PARTIAL` Badge

`State::snapshot()` clones the bars into a `Vec` and builds cumulative delta in one pass:

```rust
let mut running = 0.0;
for bar in &bars {
    running += bar.delta();
    cvd.push((bar.timestamp, running));
}
OrderFlowSnapshot {
    bars,
    cvd,
    total_delta: running,
    partial: self.partial,
}
```

The chart consumes both series through `compute_indicators_for_chart`, which filters each against the set of candle timestamps currently loaded:

```rust
let candle_ts: HashSet<i64> = candles.iter().map(|candle| candle.timestamp).collect();
let delta_series: Vec<(i64, f64)> = orderflow.bars.iter()
    .filter(|bar| candle_ts.contains(&bar.timestamp))
    .map(|bar| (bar.timestamp, bar.delta()))
    .collect();
```

Both are emitted as `IndicatorLineSpec` values into one subpane at `PaneTarget::Subpane(subpane_idx)`: delta as `IndicatorStyle::Histogram`, CVD as `IndicatorStyle::Line`, both in `0x00F0FFFF` cyan. Emitting them as indicator lines rather than bespoke drawing code is the reason `e588beb` — *Label the order-flow chart subpane* — is 57 added lines instead of a new rendering path. The legend finds them by the `orderflow-` prefix on `instance_id`, renders a bold `"ORDER FLOW"` header, and looks up each series value at the active candle timestamp with `binary_search_by_key`. That binary search is only valid because the series are sorted, which they are because they inherit the `BTreeMap` ordering. The legend then positions itself by walking the accumulated `subpane_ratios` to find the pane's top edge.

One honest limitation is visible in the filter. Order-flow bars outside the currently loaded candle range are dropped from the pane. If you scroll the chart back beyond the candle feed's history, delta and CVD do not extend into that gap even if the trade feed had aggregated there. For a charting feature that is the correct trade-off — a delta histogram floating over empty price space would read as a data error — but it does mean the series always starts where the candles start.

The `partial` flag is the interesting one. `drain_loop` sets it when the broadcast receiver reports `RecvError::Lagged(skipped)`:

```rust
Err(broadcast::error::RecvError::Lagged(skipped)) => {
    let _ = skipped;
    if let Ok(mut state) = inner.lock() {
        state.partial = true;
    }
    version.fetch_add(1, Ordering::Relaxed);
}
```

The `Lagged` variant is Tokio's slow-consumer signal: the consumer fell behind and *n* messages were dropped from the ring buffer rather than blocking the sender. That is precisely the condition under which a delta histogram becomes a lie — the aggregation is missing trades it has no way to recover. FinTer's response is to stop claiming completeness. The chart renders a bold `PARTIAL` badge in the theme's warning colour next to the order-flow legend, driven directly off `snapshot.partial`.

The design choice worth stating plainly: **`partial` is sticky for the life of the feed.** Nothing in `src/data/orderflow_feed.rs` resets it to `false` — the only assignment in the file is `= true`. Once the feed has missed messages, every snapshot from that feed reports `partial: true` until the pane is recreated (which `renderer.rs` does when the exchange, symbol, or timeframe changes). This is defensible: without a way to backfill the dropped trades, the feed genuinely cannot know whether the bars are whole, so a badge that cleared itself after the next successful receive would be asserting something it cannot verify. It is also not free — a single transient lag spike on a five-hour session leaves a permanent warning. Both readings are reasonable; the code picks the conservative one and says so in the UI.

---

## Footprint: 30 Price Rows per Bar

A footprint chart replaces the candle body with a grid: for each bar, a column of price rows, each row showing buy volume × sell volume executed at that price. It is the standard way to see *where inside a bar* the aggression happened. The reference implementations — Sierra Chart's Numbers Bars, TradingView's footprint charts — draw one row per price tick by default.

FinTer's `b932a70` takes a different approach, and the difference is worth understanding before reading the code: **rows are fixed buckets, not ticks.**

`ChartPanel::compute_footprint_cells` establishes the vertical scale from the visible candles:

```rust
let visible_candles = &candles[candles.len().saturating_sub(120)..];
let candle_ts: HashSet<i64> = visible_candles.iter().map(|c| c.timestamp).collect();
let mut min_price = f64::INFINITY;
let mut max_price = f64::NEG_INFINITY;
for candle in visible_candles {
    min_price = min_price.min(candle.low);
    max_price = max_price.max(candle.high);
}
const ROWS: usize = 30;
let step = (max_price - min_price) / ROWS as f64;
```

Every `price_rows` entry for every bar in the candle set is then folded into one of 30 buckets:

```rust
let idx = (((row.price - min_price) / step).floor() as isize)
    .clamp(0, ROWS as isize - 1) as usize;
buckets[idx].0 += row.buy_volume;
buckets[idx].1 += row.sell_volume;
```

The `.clamp()` is load-bearing. Trades execute outside the high-low range of the bar that contains them — a print that arrives with an exchange timestamp from the previous tick can be aggregated into a bar whose candle has not yet extended to that price. Without the clamp, those `f64`-to-`isize` casts would index out of bounds, and on a negative index they would panic rather than silently misplace. Clamping folds them into the nearest edge row.

The consequence of 30 fixed rows over a wide range is that a row can span several price ticks. At a $500 range that is roughly $16.67 per row; at $50 it is $1.67. This is a deliberate resolution trade: 30 rows fit a pane without text clipping, and the payload per cell is two formatted volumes, which need horizontal room. Sierra Chart's Numbers Bars solve the same problem with a user-selectable volume-at-price multiplier for exactly this reason. The honest characterisation is that FinTer renders a **bucketed footprint**, not a per-tick one, and the bucket width is a function of the visible price range.

Formatting is compacted to fit:

```rust
let fmt_volume = |value: f64| -> String {
    if value >= 1_000_000.0 { format!("{:.1}M", value / 1_000_000.0) }
    else if value >= 1_000.0 { format!("{:.1}K", value / 1_000.0) }
    else if value >= 1.0 { format!("{value:.1}") }
    else { format!("{value:.2}") }
};
// ...
text: format!("{}×{}", fmt_volume(buy), fmt_volume(sell)).into(),
```

The `×` separator is the conventional bid/ask delimiter; Sierra Chart makes it configurable, FinTer hard-codes it.

Rendering lives in `gpui-component-patched/src/chart/candlestick_chart.rs` as a new `FootprintCellSpec { timestamp_ms, price_low, price_high, text, background, text_color }`, and it contains one decision that explains a user-visible behaviour:

```rust
if cell_w >= 26.0 {
    paint_text_line_centered(cell.text.clone(), cell.text_color, bounds, window, cx);
}
```

Text is skipped below 26 px of column width. A 120-bar window on a 1,200 px chart gives 10 px slots, so at the default zoom the footprint renders as a colour field with no numbers — the buy/sell dominance is legible, the magnitudes are not. Zoom in past the threshold and the numbers appear. That is a reasonable progressive-disclosure choice, and it is also the kind of thing that generates bug reports from users who expect numbers at every zoom level.

The second decision is in `chart_panel.rs`:

```rust
let mut chart = CandlestickChart::new(candles)
    .show_candles(self.show_main_series && !self.show_footprint)
```

Enabling footprint **turns the candles off**. Candle bodies and footprint grids occupy the same screen rectangle; drawing both produces a colour field with wicks through it. So footprint is a mode, not a layer. The toolbar reflects that — footprint is a toggle button (`toolbar_btn_footprint`, tooltip "Bid x ask footprint") rather than an overlay checkbox.

---

## Volume Profile: POC, VAH, VAL, and a 70% Value Area

The volume profile answers a different question from the footprint: not "what happened inside this one bar" but "across everything on screen, at which prices did the market actually transact". FinTer draws it as horizontal bars anchored to the right edge, with three reference lines — point of control, value area high, value area low.

`ChartPanel::compute_profile_bars` uses `ROWS = 24` and, unlike the footprint, derives its range from **all** loaded candles rather than the last 120:

```rust
for candle in candles {
    if candle.low.is_finite() && candle.high.is_finite() {
        min_price = min_price.min(candle.low);
        max_price = max_price.max(candle.high);
    }
}
```

and accumulates across all order-flow bars with no candle-timestamp filter:

```rust
for bar in &orderflow.bars {
    for row in bar.price_rows.values() {
        let idx = (((row.price - min_price) / step).floor() as isize)
            .clamp(0, ROWS as isize - 1) as usize;
        volume[idx] += row.volume();
        delta[idx] += row.delta();
    }
}
```

That asymmetry with the footprint is intentional in effect if not in comment: the footprint is a per-bar view and follows what is visible, while the profile is a session statistic and follows what is loaded. It also means the profile can legitimately be denser than the visible candles — if the trade feed retained bars the candle feed has since evicted, their volume still counts.

Bar widths are normalised against the maximum row and capped at 18% of the plot width:

```rust
let width = (value / max_volume).clamp(0.0, 1.0) as f32 * 0.18;
let bar_color = if delta[idx] >= 0.0 { color.opacity(0.36) } else { color.opacity(0.22) };
```

So a row's length encodes relative volume, and its opacity encodes whether that price's net delta was positive or negative — a two-channel encoding that lets you see "high volume, net selling" as a long dim bar in one glance. Bars are drawn before the candles in the paint order, so price action stays readable through the translucency.

### The value area algorithm, and where it diverges

The point of control is the highest-volume row:

```rust
let poc = volume.iter().enumerate()
    .max_by(|(_, a), (_, b)| a.total_cmp(b))
    .map(|(idx, _)| idx)
    .unwrap_or(0);
```

`total_cmp` rather than `partial_cmp().unwrap()` — the `NaN`-safe total order, so a single corrupted row cannot panic the render. There is no midpoint tie-break for equal-volume rows, which is a documented rule in the CME-style methodology (CQG's MPVA study resolves ties toward the middle of the range). In practice exact volume ties across two price rows are rare, but they are not impossible on a thin book.

The value area is where the implementation is worth reading closely, because the comment and the code describe different algorithms. The comment reads:

```rust
// Value area: grow outward from POC by highest-volume rows until 70%.
```

The code is:

```rust
let mut order: Vec<usize> = (0..ROWS).collect();
order.sort_by(|&a, &b| volume[b].total_cmp(&volume[a]));
let target = total_volume * 0.70;
let mut accumulated = 0.0;
let mut included = std::collections::HashSet::new();
included.insert(poc);
accumulated += volume[poc];
for idx in order {
    if accumulated >= target { break; }
    if included.insert(idx) {
        accumulated += volume[idx];
    }
}
let low_idx = *included.iter().min().unwrap_or(&poc);
let high_idx = *included.iter().max().unwrap_or(&poc);
```

Rows are sorted by volume descending and taken greedily until the accumulated share reaches 70%. The included set is a `HashSet`, so it can be **non-contiguous** — and VAH and VAL are simply the maximum and minimum indices in that set.

This differs from the standard algorithm in a way that has a visible consequence. CQG's documented method expands the value area one price at a time outward from the POC, always adding whichever adjacent price has more volume, which keeps the area a contiguous band by construction. FinTer's greedy global sort can reach down to a single high-volume row five buckets below the POC and thereby stretch VAH–VAL across a low-volume gap. The three lines still bracket roughly 70% of traded volume, but the *range* they draw can include prices that barely traded — which is exactly the inference a value-area trader is trying to avoid, since a contiguous area is supposed to represent a region of accepted value.

The fix is small (walk outward from `poc` comparing `volume[poc - k]` against `volume[poc + k]` and stop at 70%), and the comment already describes it. As shipped, the POC is correct and the 70% containment is correct; the contiguity property is not. That is a precise, verifiable difference and it is worth knowing before you trade off those lines.

---

## Liquidity Heatmap: 600 Snapshots of a 50-Level Book

Delta tells you what crossed the spread. The heatmap tells you what was sitting there waiting — resting limit liquidity, sampled over time, drawn as translucent bands behind the candles. It is the visual counterpart to the resting-depth half of the microstructure picture.

`src/data/liquidity_feed.rs` is `orderflow_feed.rs`'s twin, structurally. Same `Arc<Mutex<State>>`, same `Arc<AtomicU64>` version, same `CancellationToken` + `JoinHandle` pair cancelled in `Drop`, same `tokio::select!` drain loop over a `broadcast::Receiver`. Only the payload differs: `Receiver<Arc<OrderBook>>` in, `VecDeque<LiquiditySample>` out, `MAX_SAMPLES = 600`.

`State::ingest` flattens both sides of the book into one level list:

```rust
let mut levels = Vec::with_capacity(book.bids.len() + book.asks.len());
levels.extend(book.bids.iter().map(|level| LiquidityLevel { price: level.price, quantity: level.quantity }));
levels.extend(book.asks.iter().map(|level| LiquidityLevel { price: level.price, quantity: level.quantity }));
levels.retain(|level| level.price.is_finite() && level.quantity.is_finite());
if levels.is_empty() { return; }
```

Three details. Bids and asks are merged without a side label — the heatmap encodes *quantity at price*, and on a book where best bid is below best ask there is no ambiguity about which side a price belongs to. Non-finite levels are dropped before they can contaminate the max. And an empty book returns early rather than pushing an empty sample, so a malformed snapshot cannot carve a blank vertical stripe through the heatmap.

The timestamp fallback is the other defensive detail:

```rust
let timestamp_ms = if book.timestamp_ms > 0 {
    book.timestamp_ms
} else {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .map(|duration| duration.as_millis() as i64)
        .unwrap_or(0)
};
```

Some feeds send book snapshots without an exchange timestamp. Rejecting those would lose real data; accepting them at `0` would pile every untimestamped sample at the far left of the chart. Stamping at receipt time is the least-wrong option, and it is stable because `OrderBook` documentation notes that an exchange sequence gap means the client should discard the snapshot and wait for the next one — so a timestamp of zero is a "this snapshot is fresh but unlabelled" signal, not a stale one.

`compute_liquidity_cells` bins each sample into 30 price rows over the visible candle range and shades by relative size:

```rust
let alpha = 0.04 + 0.24 * (quantity / max_level).clamp(0.0, 1.0);
```

The floor of 0.04 matters. A linear `0.24 * ratio` alone would render a thin level at 1% of the max as 0.24% opacity — invisible. The additive floor guarantees any nonzero resting size is faintly visible, so a gap in the heatmap means "no liquidity", not "liquidity too small to see".

Width is where the honest caveat lives:

```rust
let next_ts = samples.get(index + 1).map(|next| next.timestamp_ms).unwrap_or(end_ts);
let start_ms = sample.timestamp_ms.max(first_ts);
let end_ms = next_ts.min(end_ts).max(start_ms + 1);
```

Each snapshot's liquidity band extends from its own timestamp to the *next* snapshot's timestamp. That makes the heatmap a **step function, not an interpolation**: between two samples the drawing asserts that the book looked like the earlier snapshot. When the book is updating many times per second the bands are thin and the error is negligible; when it goes quiet the last known state is extended, and a band that persists across a quiet period is showing you a book that may no longer exist. There is no visual degradation to signal staleness — the `partial` flag exists on the snapshot type and is set on `Lagged`, but no renderer reads it.

The bridge asks for real depth to make this worthwhile. In `renderer.rs` when a chart pane is created:

```rust
bridge_config.trades = true;
bridge_config.trade_limit = 5000;
bridge_config.trade_since_current_bar = true;
bridge_config.orderbook = true;
bridge_config.orderbook_depth = 50;
```

`trade_since_current_bar` is a `a2022d9` addition that rewrites the subscription to request a trade snapshot from the current bar's open rather than an arbitrary count:

```rust
let trade_since = if config.trade_since_current_bar {
    let frame_ms = BridgeConfig::timeframe_to_ms(&config.timeframe).max(1_000);
    let now = now_ms();
    Some(now - now.rem_euclid(frame_ms))
} else { None };
```

`now - now.rem_euclid(frame_ms)` is the bar open — the same floor-division identity as the bar bucketing, applied to the current wall clock. Asking for "trades since this bar opened" is the correct request for a delta panel, because the panel's first bar must be complete. Asking for "the last 100 trades" on a one-minute chart gives you a partially populated bar whose CVD starts mid-bar and looks like a divergence.

The `Requirement::Trades` variant carries the new field as `Option<i64>` with `#[serde(skip_serializing_if = "Option::is_none")]`, so an older gateway that does not understand `since` receives the same frame it always did. That is a wire-format decision worth copying: additive optional fields with skip-serialisation give you forward compatibility without a version handshake.

---

## The Order Book Gets Grouping, Imbalance, and Walls

`10f702b` is the one commit in this set that changes the depth ladder rather than the chart, and it addresses the oldest complaint about crypto order books: at tick precision on a major pair, the ladder is unreadable noise.

Grouping is a `group_size: u32` on `OrderBookPanel` that cycles `1 → 2 → 5 → 10 → 1` on click:

```rust
fn grouped_levels(levels: &[PriceLevel], group_size: u32) -> Vec<PriceLevel> {
    let group_size = group_size.max(1) as usize;
    if group_size == 1 { return levels.to_vec(); }
    levels.chunks(group_size).map(|chunk| {
        let price = chunk.first().map(|level| level.price).unwrap_or(0.0);
        let quantity = chunk.iter().map(|level| level.quantity).sum();
        PriceLevel::new(price, quantity)
    }).collect()
}
```

Price is taken from the **first** level of each chunk — for bids that is the highest price in the group, for asks the lowest, which means a grouped row's price is always the most aggressive price in it. That is the convention that keeps the ladder's spread arithmetic correct: the top grouped bid still sits closest to the spread, so `spread()` and `mid_price()` computed downstream remain meaningful. The unit test `grouping_sums_adjacent_levels` pins the quantity sum and the first-price rule.

Grouping interacts with the requested depth in a way that is easy to miss:

```rust
let raw_depth = (depth * self.group_size as usize).min(100);
let asks = Self::grouped_levels(book.top_asks(raw_depth), self.group_size);
```

The panel asks the book for `depth × group_size` levels so that grouping still fills the visible rows. But the ladder's own bridge is started from `BridgeConfig::default_dev`, which sets `orderbook_depth = 20` — and nothing in the order-book pane's setup path overrides it (only the chart pane at `renderer.rs:1996` sets `orderbook_depth = 50`). So the book holds 20 levels per side regardless of the requested `raw_depth`: at `5x` grouping the top of the ladder collapses to 4 rows, and at `10x` to 2. The `.min(100)` cap never binds, because the feed runs out first. Grouping therefore trades resolution for row count against a fixed 20-level budget, which is the opposite of what a trader reaching for `10x` wants — they want the same number of rows at coarse resolution, and they get two rows at coarse resolution.

Wall detection reuses the existing size-bar fraction and adds a left border:

```rust
let is_wall = bar_frac >= 0.60;
// ...
.when(is_wall, |row| row.border_l_2().border_color(price_color))
```

Because `bar_frac` is normalised against the maximum quantity among the **grouped** levels for the current render, changing `group_size` changes what counts as a wall. Grouping ten levels sums their quantity, so the grouped maximum rises and previously prominent single levels drop below the 60% threshold. That is coherent — the threshold is relative to what is on screen — but it means the wall highlight is not a stable property of the book, it is a property of the current view configuration.

Imbalance sits in the spread row and is computed over the raw top ten:

```rust
let bid_qty: f64 = book.top_bids(10).iter().map(|level| level.quantity).sum();
let ask_qty: f64 = book.top_asks(10).iter().map(|level| level.quantity).sum();
let total_qty = bid_qty + ask_qty;
let imbalance_str = if total_qty > 0.0 {
    format!("Imb: {:.0}%", bid_qty / total_qty * 100.0)
} else { "Imb: —".to_string() };
```

Two things follow from `top_bids(10)` being called on the ungrouped book. First, the imbalance percentage does not change when you change `group_size` — it always reports the top ten raw levels, which is the right behaviour for a metric you want to compare across sessions. Second, 50% is the neutral point, and the colour is `bullish` whenever `bid_qty >= ask_qty`, so exactly-balanced books read as bullish. That is a one-line judgement call rather than a bug, but it does mean the colour channel has 51 bullish states and 50 bearish ones.

---

## The Crosshair Card

`939e126` moves the OHLCV readout from a fixed inline row at the top of the chart to a floating card next to the cursor. It sounds cosmetic; it is actually where the arithmetic gets fiddly.

Three lines of text, each measuring itself in estimated character widths:

```rust
let text_w = time_label.len()
    .max(ohlc_line.len())
    .max(change_line.len()) as f32 * CHAR_WIDTH_EST;
let max_card_w = (chart_w - 16.0).max(80.0);
let min_card_w = 180.0_f32.min(max_card_w);
let card_w = (text_w + 18.0).clamp(min_card_w, max_card_w);
let card_h = 52.0;
```

`CHAR_WIDTH_EST = 6.0` is a constant already used elsewhere in the file for axis labels. Estimating width from character count rather than measuring shaped text is the pragmatic choice inside a paint pass — there is no text layout system available at that point — but it means a label rendered in a wide glyph set can overrun the card. The `min_card_w = 180.0` floor guarantees the card is never a sliver, and the `.max(80.0)` on `max_card_w` guarantees the clamp bounds never invert on a very narrow pane.

Placement flips rather than overflows:

```rust
let preferred_x = if cursor_x_in_chart + 16.0 + card_w <= chart_w {
    cursor_x_in_chart + 16.0
} else {
    cursor_x_in_chart - card_w - 16.0
};
let preferred_y = if cursor_rel.y.as_f32() + 16.0 + card_h <= candle_h {
    cursor_rel.y.as_f32() + 16.0
} else {
    cursor_rel.y.as_f32() - card_h - 16.0
};
let card_x = preferred_x.clamp(4.0, (chart_w - card_w - 4.0).max(4.0));
let card_y = preferred_y.clamp(4.0, (candle_h - card_h - 4.0).max(4.0));
```

Preferred position is down-right of the cursor with a 16 px offset; if that would leave the plot, it goes up-left. The final clamp catches the corners, where both preferences can fail — cursor in the bottom-right quadrant flips left and up, but a card 60% of the pane width could still start at a negative x, and the clamp pulls it back inside the plot rect. Note the clamps use `4.0` padding against `chart_w` and `candle_h`, not against the pane bounds, so the card is confined to the candle area and never floats over the volume subpane.

The card itself is a `paint_quad` fill at `cx.theme().background.opacity(0.94)` plus a 1 px border built as an explicit `PathBuilder` rectangle, and the three text rows are drawn with `paint_text_line` clipped to `card_bounds`. The 0.94 alpha is what makes the card readable over candles without hiding them entirely — at full opacity the card would occlude the very bars it is describing.

The values come from the same `high_v`/`low_v`/`vol_v` closures the chart already resolves. The diff shows those bindings were previously prefixed with underscores (`_high_v`, `_low_v`, `_vol_v`) because nothing read them; the card is now their first consumer. Change percentage is computed with an explicit epsilon guard:

```rust
let change_pct = if open_v.abs() > f64::EPSILON { change / open_v * 100.0 } else { 0.0 };
```

Division by a zero open is the kind of thing that happens once per instrument (a newly listed pair, a bad bar) and produces `inf` or `NaN` in a formatted string forever after. The guard costs one comparison.

The same commit adds a chart-type dropdown with seven types — `Solid`, `HeikinAshi`, `Stroke`, `UpStroke`, `DownStroke`, `Ohlc`, `Area` — built as a `DropdownMenu` whose items carry `.checked(current_candle_type == kind)`. Selecting one calls `set_candle_type` inside `window.defer(...)` so the menu closes before the chart rebuilds, then restores focus to the panel's `focus_handle`. That focus restore is why the keyboard shortcuts keep working after a menu interaction.

---

## Four Booleans That Have to Survive a Restart

`src/chart/model.rs` gained four fields on `ChartLayout` across these commits:

```rust
/// Show the order-flow delta/CVD subpane.
#[serde(default = "default_show_delta")]
pub show_delta: bool,
/// Show the right-aligned volume profile.
#[serde(default)]
pub show_profile: bool,
/// Render the chart as a bid x ask footprint.
#[serde(default)]
pub show_footprint: bool,
/// Show resting-liquidity bands behind candles.
#[serde(default)]
pub show_liquidity: bool,
```

The serde attributes are the whole point. `show_delta` uses `default_show_delta()` returning `true`, because it shipped enabled — a user who saved a layout before the field existed should get the delta pane, not lose it. The other three use plain `#[serde(default)]`, which for `bool` is `false`, matching their shipped state. An old layout file deserialises cleanly and gets exactly the defaults a new pane would.

Both directions are wired. `load_layout` copies field to panel; `save_layout` copies panel back to field. Skipping either half produces the classic bug where a toggle works until restart and then silently reverts — and because these are four independent booleans, a single missed line would only affect one feature, which is exactly the kind of defect that ships.

`save_layout` rebuilds a fresh `ChartLayout::new(exchange, symbol, timeframe)` on every write and repopulates it field by field, so the persisted file is always a complete layout at the current `schema_version` rather than a patched older one. `load_layout` guards one field asymmetrically — `indicators` is only overwritten when the loaded vec is non-empty, while the four visibility booleans are always applied, because `false` is a meaningful value for them and an empty indicator list is usually a missing file rather than a deliberate clear.

The keyboard surface documents the same four: `d` delta, `p` profile, `v` footprint, `h` liquidity — alongside `g` grid, `i` indicator library, `l` scroll to latest, `f` reset view. The `v` binding is the one to note: footprint took `v` from what a reader might expect to be volume, because volume is already a native pane in `CandlestickChart` behind `show_volume_pane`.

---

## What This Is Not

Precision about boundaries is more useful than a feature list.

- **No per-tick footprint.** Rows are 30 fixed buckets over the visible price range, so a row can span multiple ticks. Sierra Chart's numbers bars default to one row per tick with a configurable multiplier for exactly this trade.
- **No contiguous value area.** VAH and VAL are the min and max of a greedily selected set, so the drawn range can include low-volume gaps. The POC and the 70% containment are correct; the contiguity property is not implemented.
- **No session split.** The profile aggregates every loaded candle and every retained order-flow bar into one distribution. There is no per-day or per-session breakdown, so "value area" here means "value area of whatever is loaded", not "of today's session".
- **`partial` is permanent.** Once the order-flow feed detects a broadcast lag, the badge stays until the pane is recreated. No backfill exists to clear it.
- **Stale heatmap bands are not flagged.** The `LiquiditySnapshot` carries a `partial` flag, but no renderer reads it, so a long quiet period extends the last known book without a visual warning.
- **Imbalance ignores grouping.** It is always computed from the raw top ten levels, which makes it comparable across views but detached from the grouped ladder beside it.
- **Delta starts where the candles start.** Order-flow bars outside the loaded candle range are filtered out of the subpane.

None of these are hidden. The commit that shipped the footprint added per-price rows to `FlowBar` in the same change, so the data model is already sufficient for per-tick rendering — the constraint is the 30-row layout, not the aggregation.

---

## Why This Belongs in a Native Terminal

Every one of these features is a per-frame recomputation over a growing data structure. The footprint touches up to 120 bars × 30 buckets; the profile walks every retained bar's price rows; the heatmap bins 120 samples; the ladder re-groups and re-normalises the book on every render. In a browser these would be DOM nodes and layout invalidation. Here they are `Vec<Spec>` values handed to `CandlestickChart`, which paints them with `paint_quad` and `paint_path` calls in a single pass.

That is the architectural payoff. `FootprintCellSpec`, `ProfileBarSpec`, `ProfileLineSpec`, and `HeatmapCellSpec` are plain data — no widgets, no retained tree, no style resolution. The chart component iterates them in paint order: heatmap, then profile bars, then profile lines, then footprint cells, then candles. Reordering those five blocks reorders the visual layers, which is why the heatmap can sit behind the candles while the profile stays readable through them, using the same mechanism.

It also means the expensive part — aggregation — is a pure function of the trade stream and can be tested without a window. The four unit tests in `orderflow_feed.rs` and `liquidity_feed.rs` cover exactly the cases that break in production: unknown aggressor sides, duplicate trade IDs, out-of-order bars, timeframe boundaries, and ring-buffer eviction at `MAX_SAMPLES + 5`. None of them need GPUI, a socket, or a running app.

---

## Build and Run FinTer

FinTer is open source under the [coin-watch/FinTer](https://github.com/coin-watch/FinTer) repository, version marker `0.2.0`.

```bash
git clone https://github.com/coin-watch/FinTer
cd FinTer
cargo run
```

Guest mode runs without an account — layouts persist locally and the terminal shows explicit empty states when market data is unavailable. Signed-in builds default to the hosted CoinWatch backend for account, profile, and settings data, while layouts stay on local disk. Prebuilt binaries for macOS, Windows, and Linux are on the [download page](https://coinwat.ch/download).

The order-flow panels are on by default only for delta; profile, footprint, and liquidity are one click or one keystroke away. On a $BTC 1-minute pane, delta and footprint together are the fastest way to see the difference between a move that was bought aggressively and a move that was sold into a bid that kept refilling.

---

## Summary

Six commits on 23 November 2026 added a complete order-flow stack to a Rust/GPUI terminal, and the interesting engineering is in the boundaries rather than the features:

1. **Layer separation beats optimisation.** A Tokio task drains trades into an `Arc<Mutex<State>>`, publishes an `AtomicU64` version, and the UI polls it at 250 ms. Render cost is decoupled from trade rate by construction.
2. **Three states, not two.** `TradeSide::Unknown` is not "sell". Folding it into the boolean case would have inverted delta on any frame with an unparsed side, and the chart would have looked plausible while being wrong.
3. **Integer keys for float prices.** `(price * 1_000_000.0).round() as i64` makes price rows totally ordered, duplicate-free, and safe as a `BTreeMap` key.
4. **Bounded everything.** `MAX_SEEN_TRADE_IDS = 20_000`, `MAX_BARS = 5_000`, `MAX_SAMPLES = 600`, `.min(100)` on grouped depth. Long sessions do not grow without limit.
5. **Comments can outrun code.** The value area comment describes contiguous expansion from the POC; the implementation takes a global greedy sort. POC and 70% containment hold, contiguity does not.
6. **Absence of a signal is a signal.** The `PARTIAL` badge exists because a lagged feed cannot claim to be complete — and it is sticky for exactly that reason.

The full source is at [github.com/coin-watch/FinTer](https://github.com/coin-watch/FinTer). For the quantitative background these panels are built to display, see [order flow imbalance and microstructure](https://coinwat.ch/blog/order-flow-imbalance-crypto-microstructure-algorithmic-trading); for the terminal's threading and rendering architecture, see [FinTer's Rust and GPUI architecture](https://coinwat.ch/blog/finter-rust-gpui-crypto-terminal-architecture); for how API credentials are protected while all of this runs locally, see [AES-256-GCM at rest in FinTer](https://coinwat.ch/blog/finter-crypto-terminal-api-security-aes-256-gcm-rust); for how these same order books are reconstructed from live exchange feeds at the gateway, see [rebuilding the Cryptowatch order book API](https://coinwat.ch/blog/cryptowatch-order-book-api-rebuild-live-exchange-feeds); and for how the panes holding these charts persist and sync, see [FinTer's local-first dashboards](https://coinwat.ch/blog/finter-local-first-dashboards-file-sync-agent-pane-control).

---

## Sources & Technical References

1. https://github.com/coin-watch/FinTer — FinTer source repository (commits `a2022d9`, `e588beb`, `b932a70`, `f87a8ec`, `10f702b`, `939e126`, `a013314`)
2. https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html — Tokio broadcast channels and the slow-consumer / `Lagged` contract
3. https://docs.rs/tokio/latest/tokio/macro.select.html — `tokio::select!` macro reference
4. https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html — cancellation tokens used to terminate feed tasks on `Drop`
5. https://doc.rust-lang.org/std/collections/struct.BTreeMap.html — ordered map semantics used for bar and price-row storage
6. https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html — `Ordering::Relaxed` semantics for the version counter
7. https://github.com/binance/binance-spot-api-docs/blob/master/web-socket-streams.md — aggregate trade stream payload, including `"m": true // Is the buyer the market maker?`
8. https://www.sierrachart.com/index.php?page=doc/NumbersBars.php — Numbers Bars reference implementation of bid × ask footprint charts
9. https://help.cqg.com/cqgic/25/Documents/marketprofilevalueareasmpva.htm — CQG MPVA documentation: value area expansion from the POC and the 70% convention
10. https://www.tradingview.com/support/solutions/43000502040-volume-profile/ — volume profile: POC and value area definitions
11. https://www.tradingview.com/support/solutions/43000502284-footprint-charts/ — footprint chart conventions
12. https://docs.rs/gpui/latest/gpui/ — GPUI, the GPU-accelerated Rust UI framework FinTer is built on
13. https://www.rust-lang.org/ — Rust language reference

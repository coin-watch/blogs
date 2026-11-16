---
title: "Local-First Dashboards in FinTer: File-Backed Layouts, Last-Write-Wins Sync, and Agent-Controlled Pane Grids"
date: "2026-11-16"
description: "How FinTer replaced Supabase layout tables with one JSON file per dashboard, added last-write-wins cloud sync with a provable pull/push symmetry, and let an AI agent rearrange any pane grid across windows — all in Rust on GPUI."
tags: ["FinTer", "Rust", "GPUI", "Local-First", "Trading Terminal", "Architecture", "Cloud Sync", "AI Agents", "Open Source", "Market Microstructure"]
---

# Local-First Dashboards in FinTer: File-Backed Layouts, Last-Write-Wins Sync, and Agent-Controlled Pane Grids

![FinTer cryptocurrency trading terminal with a tiled pane grid dashboard, JSON file icons flowing into a local disk vault, bidirectional cloud sync arrows, and an AI agent cursor rearranging chart panes, dark background #121613, neon green accents #C4E855, isometric technical illustration, clean minimal financial software aesthetic](https://image.pollinations.ai/prompt/FinTer%20cryptocurrency%20trading%20terminal%20with%20a%20tiled%20pane%20grid%20dashboard%2C%20JSON%20file%20icons%20flowing%20into%20a%20local%20disk%20vault%2C%20bidirectional%20cloud%20sync%20arrows%2C%20and%20an%20AI%20agent%20cursor%20rearranging%20chart%20panes%2C%20dark%20background%20%23121613%2C%20neon%20green%20accents%20%23C4E855%2C%20isometric%20technical%20illustration%2C%20clean%20minimal%20financial%20software%20aesthetic?width=800&height=400&nologo=true)

A trading terminal's layout is not decoration. It is the trader's working memory made spatial: the 1-minute $BTC chart on the left, the depth ladder in the middle, the funding-rate pane underneath, the liquidation tape pinned to the right monitor. Rebuilding that arrangement after every restart is a small insult repeated daily. Losing it because a backend had an outage is worse.

That is why the most consequential change in **[FinTer](https://github.com/coin-watch/FinTer)** this week is not a chart feature. It is a storage decision. In three commits on 15 November 2026, Coinwatch's open-source Rust/GPUI terminal moved dashboards out of two Supabase tables and into **one JSON file per dashboard on local disk**, added an **optional** push/pull bridge to the cloud with a last-write-wins conflict rule, and gave its AI agent the ability to **rearrange any pane grid in any window** through a typed wire format.

This is a deep dive into the implementation: the file store and its atomic-write discipline, the migration path that avoided a flag day, the sync algorithm and the symmetry property that stops it ping-ponging, the GPUI `Global` registry that lets one conversation reach every dashboard, and a smaller but telling fix that makes forming candles advance on wall-clock time rather than on socket luck.

---

## The Last Cloud Dependency Was the Layout

FinTer's architecture has been local-first since its first release: market data streams directly from exchanges over WebSocket, indicators compute in-process, and guest mode runs without an account. The one thing that still required a network round-trip was the thing users touch most — the dashboard itself.

Before commit `68a4e54`, dashboards lived in two Supabase tables:

- `modules` — the saved pane trees, reached over `/api/modules`
- `layout_presets` — named layouts with a `is_public` flag, reached over `/api/layout-presets`

Every layout save was an HTTP request. Every application launch that wanted to restore your workspace depended on a reachable backend and a valid session. Guest mode could not manage dashboards at all, because the create and delete paths only mutated an in-memory list — a new dashboard vanished on restart, and a deleted one came back.

The new model inverts this. The local file is the source of truth. The server holds a copy, and only when you ask it to.

```text
+-----------------------------------------------------------------------------------+
|                        FINTER DASHBOARD STORAGE (AFTER)                           |
|                                                                                   |
|   ~/.config/finter/dashboards/                                                    |
|     0f8c2a91-....json   { id, name, tree, is_public, created_at, updated_at }      |
|     7b31d4e0-....json   { id, name, tree, is_public, created_at, updated_at }      |
|                                                                                   |
|            |  read/write (no network, no auth, no spinner)                        |
|            v                                                                      |
|   [ Dashboard Manager UI ]  <----  [ Guest Mode: fully functional offline ]        |
|            |                                                                      |
|            |  explicit toolbar action only                                        |
|            v                                                                      |
|   [ dashboard_sync::sync_all ]  --->  PATCH/POST /api/modules  --->  R2 envelope  |
+-----------------------------------------------------------------------------------+
```

---

## One JSON File Per Dashboard

The new store lives in `src/data/dashboard_files.rs`. The on-disk envelope is deliberately boring:

```rust
/// A dashboard persisted as a single JSON file.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DashboardFile {
    /// Stable identifier; also the file stem.
    pub id: String,
    /// Human-readable name, editable without renaming the file.
    #[serde(default)]
    pub name: String,
    /// Serialized `PaneNode` tree.
    #[serde(default)]
    pub tree: serde_json::Value,
    #[serde(default)]
    pub is_public: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}
```

The payload shape matters less than the identity rule. **The filename is the identity.** A dashboard is `~/.config/finter/dashboards/<uuid>.json`, and the display name lives inside the file. Renaming a dashboard therefore never moves a file, never invalidates a reference the agent is holding, and never creates a duplicate.

### Atomic writes: temp file, then rename

A layout save happens on nearly every keystroke-adjacent interaction — pane splits, symbol changes, indicator toggles. That is a lot of write traffic against a file the user's workspace depends on. A crash mid-write that leaves a truncated JSON document would cost the user their entire arrangement.

The store writes to a sibling temp file and renames:

```rust
fn write_file(path: &Path, dashboard: &DashboardFile) -> Result<()> {
    if let Some(parent) = path.parent() {
        std::fs::create_dir_all(parent)
            .map_err(|e| CoinWatchError::Storage(format!("mkdir {}: {e}", parent.display())))?;
    }
    let json = serde_json::to_string_pretty(dashboard)
        .map_err(|e| CoinWatchError::Deserialization(format!("serialize dashboard: {e}")))?;

    // Write to a sibling temp file then rename, so a crash mid-write cannot
    // leave a truncated dashboard behind.
    let tmp = path.with_extension("json.tmp");
    std::fs::write(&tmp, &json)
        .map_err(|e| CoinWatchError::Storage(format!("write {}: {e}", tmp.display())))?;
    std::fs::rename(&tmp, path)
        .map_err(|e| CoinWatchError::Storage(format!("rename to {}: {e}", path.display())))?;
    Ok(())
}
```

This leans on a POSIX guarantee that is easy to state and easy to get wrong: [`rename(2)`](https://man7.org/linux/man-pages/man2/rename.2.html) atomically replaces the destination, so *"there is no point at which another process attempting to access `newpath` will find it missing."* A reader either sees the old complete file or the new complete file. Never a half-written one.

The same property is what makes the pattern portable to Windows, where `std::fs::rename` maps onto the equivalent replace-existing semantics rather than a delete-then-create.

### Path traversal: validate the id, not the path

The store accepts an `id` from several directions — the UI, a migrated row, a remote sync response. That id becomes a filename, which makes it an attack surface. A crafted id like `../../.ssh/authorized_keys` would be catastrophic if joined naively.

The defense is an allowlist on the identifier, applied before the path is ever constructed:

```rust
fn dashboard_path(dir: &Path, id: &str) -> Result<PathBuf> {
    if !is_valid_id(id) {
        return Err(CoinWatchError::InvalidRequest(format!(
            "invalid dashboard id: {id:?}"
        )));
    }
    Ok(dir.join(format!("{id}.json")))
}

/// Dashboard ids are UUIDs; reject anything that could escape the directory.
fn is_valid_id(id: &str) -> bool {
    !id.is_empty()
        && id.len() <= 64
        && id
            .chars()
            .all(|c| c.is_ascii_alphanumeric() || c == '-' || c == '_')
}
```

No `.`, no `/`, no `\`, no null byte, bounded length. This is the allowlist-first approach that [OWASP's guidance on path traversal](https://portswigger.net/web-security/file-path-traversal) recommends: reject input that is not a known-good shape rather than trying to sanitize a bad one. Because the character set excludes the separator and the parent-directory token entirely, there is no encoding trick to defeat — the check happens on the decoded `String` that will actually be used.

### Corruption is contained, not fatal

Two failure modes get explicit handling, and both are about not letting one bad file cascade:

**Unparseable files are skipped, not fatal.** `list_in` logs a warning and continues:

```rust
match read_file(&path) {
    Ok(dashboard) => dashboards.push(dashboard),
    Err(e) => tracing::warn!("skipping unreadable dashboard {}: {e}", path.display()),
}
```

One corrupt dashboard must not hide the user's other nine. A listing that returns an error is a listing that renders an empty workspace, which looks exactly like data loss.

**Legacy bare-tree files are lifted.** An earlier build wrote the raw `PaneNode` tree into this directory with no envelope. Rather than orphan those dashboards on upgrade, `read_file` detects the absence of `id`/`tree` keys and wraps the value, using the file stem as the id and the file's mtime as both timestamps:

```rust
if value.get("id").is_some() && value.get("tree").is_some() {
    return serde_json::from_value(value)
        .map_err(|e| CoinWatchError::Deserialization(format!("parse {}: {e}", path.display())));
}
// ... otherwise lift the bare tree into the envelope
```

The store also exposes a directory-scoped API (`list_in`, `load_in`, `upsert_in`, `delete_in`) beneath the user-facing one, so tests can point at a temp directory instead of the real config path. That is what makes the path-traversal and corrupt-file cases testable at all.

---

## Migrating Without a Flag Day

The awkward part of any storage migration is the read path. If the migration reads legacy rows through the *new* client, and the new client reads the local file store, the migration reads its own output and produces nothing.

`src/data/dashboard_migration.rs` sidesteps this by talking to the original HTTP endpoints directly:

```rust
/// Raw `GET /api/modules` against the legacy table.
fn fetch_legacy_modules(client: &ApiClient) -> Result<Vec<Module>> {
    let user_id = client
        .user_id()
        .ok_or_else(CoinWatchError::Auth("Not logged in".into()))?;

    let rt = crate::data::client::tokio_runtime();
    let req = client.auth_get(&format!("/api/modules?user_id={user_id}"));
    // ...
}
```

The module doc comment states the reason plainly: it deliberately does **not** go through `data::modules` / `data::layout_presets`, "because those now read the local file store — using them here would make the migration read its own output."

Three properties make the migration safe to run more than once:

1. **Idempotency.** Rows whose id already exists on disk are skipped unless `overwrite` is set. Re-running is a no-op rather than a clobber.
2. **Missing endpoints are not errors.** A `404` from `/api/modules` logs *"legacy /api/modules not deployed; nothing to migrate"* and returns an empty vector. A user on a newer deployment does not see a failure.
3. **Preset rows win over module rows sharing an id**, because only presets carry `is_public`.

The migration preserves ids and the `is_public` flag, so a dashboard that was public before the migration is still public after it.

---

## The Seam That Kept the Change Small

The interesting engineering choice here is not the file format. It is that `data/modules.rs` and `data/layout_presets.rs` **kept their public method signatures** and now delegate to the file store.

That single decision meant roughly forty existing call sites did not change. The dashboard manager, the layout preset picker, and the save path all continued to call the same functions, which now happen to read JSON off disk instead of HTTP off a server.

It also produced the property that unlocks guest mode: those methods no longer require authentication. Previously, managing dashboards was implicitly a signed-in operation. Now it is a filesystem operation, and authentication is only needed for the optional sync.

The commit also fixed the guest branches in `mainview.rs` that had been mutating an in-memory list. Create and delete now write through the store, so a new dashboard survives a restart and a deleted one stays deleted. The per-keystroke layout save writes to the store instead of the old `guest_layout.json` / `cloud_layout.json` pair — one code path instead of two divergent ones.

Verification for this commit was done in the running app, not only in tests: a dashboard file is created on first launch, reused (not duplicated) on restart, and its layout renders. Alongside that, 10 unit tests in `dashboard_files` cover path-traversal rejection, corrupt-file tolerance, legacy-file lifting, and migration idempotency, plus 4 integration tests in `tests/dashboard_files_roundtrip.rs` covering create/list/delete, persistence across a fresh read, and migration fidelity.

---

## Sync: Last-Write-Wins, and Why Not a CRDT

With local files as the source of truth, the cloud becomes a peer rather than an authority. `src/data/dashboard_sync.rs` implements an explicit, user-triggered bridge that stores the same envelope in R2.

The conflict rule is **last-write-wins on `updated_at`**. Both sides carry the field and both bump it on edit, so the newer timestamp is the newer intent.

That is a deliberate rejection of the more sophisticated option. From the module documentation:

> That is deliberately simple — a dashboard is a single user's layout, and the alternative (merging two pane trees) produces layouts neither side asked for.

This is worth dwelling on, because the reflexive answer in local-first circles is a CRDT. Kleppmann et al.'s [local-first essay](https://www.inkandswitch.com/essay/local-first/) is explicit that CRDTs are the foundational technology for the field, and that merging concurrent edits cleanly is the hard problem the model has to solve. But a pane tree is not a text document. Merging two divergent split trees produces a layout with panes neither device had open — technically convergent, practically useless. Last-write-wins loses at most one edit's worth of layout on one device, and the user can redo it in seconds. The complexity budget is better spent elsewhere.

### Pure decision functions, and the symmetry property

The rule is not buried in the I/O. It lives in two pure functions that take timestamps and return an enum:

```rust
/// What to do with one dashboard during a pull.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PullAction {
    /// Not present locally — write it.
    Create,
    /// Remote is newer — overwrite the local copy.
    Overwrite,
    /// Local is same age or newer — leave it alone.
    Keep,
}

pub fn pull_action(local: Option<&DashboardFile>, remote: &DashboardFile) -> PullAction {
    match local {
        None => PullAction::Create,
        Some(local) if remote.updated_at > local.updated_at => PullAction::Overwrite,
        Some(_) => PullAction::Keep,
    }
}
```

`push_action` is its mirror. Because both are pure, the conflict rule is testable without a network or a filesystem — which is the only way to get real coverage of a distributed edge case.

Three details in that design are load-bearing:

**Symmetry prevents ping-pong.** The two functions are deliberately symmetric: if a pull would overwrite, the matching push skips. Without that, a single `sync_all` run would pull a remote dashboard down, then immediately push the same bytes back up, and the next run would do it again. A test asserts the symmetry directly.

**Equal timestamps mean nobody edited.** If both sides carry the same `updated_at`, neither writes. Rewriting on equality would churn every file on every sync — a full disk write and a full network round-trip for a dashboard that did not change.

**A null tree never overwrites a good local file.** The server returns rows in the old table shape for compatibility, so `RemoteRow::into_file` returns `None` for a row whose `module` is null:

```rust
fn into_file(self) -> Option<DashboardFile> {
    // A row without a tree cannot become a dashboard; skip rather than
    // writing an empty one over a good local file.
    if self.module.is_null() {
        return None;
    }
    // ...
}
```

A malformed response cannot blank out a working dashboard. That is a cheap check that prevents a data-loss bug class.

### Failure isolation and the no-auto-sync rule

`sync_all` pulls then pushes, so a single run converges both directions. Per-dashboard failures are collected into a `SyncReport` rather than aborting the run — one unreachable dashboard should not block the other nine.

```rust
pub struct SyncReport {
    pub pulled: usize,
    pub pushed: usize,
    pub local_wins: usize,
    pub remote_wins: usize,
    pub local_only: usize,
    pub remote_only: usize,
    /// Per-dashboard failures. A single failure does not abort the run.
    pub errors: Vec<String>,
}
```

And critically: **sync never runs automatically.** It is triggered by a toolbar button in the dashboard manager (`toolbar-sync`, tooltip *"Sync dashboards with the cloud"*) and executes on the background executor:

```rust
fn sync_dashboards(&mut self, cx: &mut Context<Self>) {
    if !self.is_authenticated {
        tracing::info!("Skipping dashboard sync: not signed in");
        return;
    }

    let client = self.api_client.clone();
    cx.spawn(async move |this, cx| {
        let result = cx
            .background_executor()
            .spawn(async move { crate::data::dashboard_sync::sync_all(&client) })
            .await;
        // ...
    })
    .detach();
}
```

A network failure can neither block the UI nor lose a local edit, because no local edit ever waits on the network in the first place.

The commit message is honest about the one gap: the sync button is wired but was not clicked through by hand, because the panel hosting it would not stay open under synthetic X11 clicks. The logic beneath it is covered by 12 unit tests, and the full library suite stood at 299 passing (up from 287), with two pre-existing `orderbook_feed` failures that reproduce on a clean tree. Recording that limitation rather than claiming a manual verification is the right call — and it is a useful reminder that UI automation of custom-rendered GPUI surfaces is genuinely harder than automating a DOM.

---

## The Agent Can Now Rearrange Your Workspace

The third commit, `fe4754e`, is the one with the longest reach. It gives FinTer's agent panel the ability to control **every pane grid, in every window**, through a typed wire format.

The design constraint was dependency direction. FinTer's `data` layer never imports `crate::components`. So the wire format lives in `src/data/pane_control.rs`, entirely free of UI types, and the component layer consumes it.

### Eleven actions, one fenced block each

The agent emits fenced `ui` blocks in its reply. Each block is one `UiAction`, addressed at a dashboard by id, name, `active`, or `all`:

```rust
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[serde(tag = "action", rename_all = "snake_case")]
pub enum UiAction {
    /// Replace a pane's module type, optionally merging a new config.
    SetModule {
        #[serde(default)]
        dashboard: Option<String>,
        pane: String,
        #[serde(alias = "type", alias = "module")]
        module_type: String,
        #[serde(default)]
        config: serde_json::Value,
    },
    /// Merge keys into a pane's existing config (exchange, symbol, interval…).
    UpdateConfig { /* ... */ },
    /// Split one pane in two along an axis.
    Split { /* ... */ },
    /// Replace a pane with an evenly distributed rows × cols matrix.
    SplitMatrix { /* ... */ },
    /// Remove a pane and collapse its parent split.
    Close { /* ... */ },
    /// Exchange the contents of two panes.
    Swap { /* ... */ },
    Maximize { /* ... */ },
    Restore { /* ... */ },
    Focus { /* ... */ },
    /// Move a split divider to `ratio` (0.05–0.95).
    Resize { /* ... */ },
    /// Replace the entire dashboard layout with a complete pane tree.
    SetLayout { /* ... */ },
}
```

A model reply looks like this — one action per fence, which keeps parsing trivial and failures attributable:

````text
```ui
{"action":"set_module","pane":"#2","module_type":"chart","config":{"exchange":"binance","symbol":"ETH/USDT","timeframe":"15m"}}
```

```ui
{"action":"split","pane":"#1","axis":"vertical"}
```
````

The `serde` aliases are a small but real usability decision. `module_type` accepts `type` or `module`; `Swap` accepts `with`, `other`, or `target`; `Resize` accepts `split`, `pane`, or `node`. An LLM that picks a slightly different key name still produces a valid action instead of a parse error, which matters when the cost of a retry is a full round-trip.

### Reference resolution: how the agent says "that one"

Models are unreliable at reproducing UUIDs. So `src/data/pane_tree_describe.rs` flattens the tree into a stable depth-first list of descriptors and resolves references in a documented precedence order:

```rust
/// Accepts, in order of precedence:
/// - an exact pane id,
/// - `#3`, `3` or `pane 3` — the 1-based depth-first index,
/// - a module type (`chart`, `ticker`, …) when exactly one pane matches,
/// - a unique id prefix (the first 8 characters of a UUID, for example).
pub fn resolve_pane_reference(&self, reference: &str) -> Option<String> { /* ... */ }
```

Each descriptor carries an `index`, an `id`, a `module_type`, its `config`, and a position path like `root/a` — the path being useful because it lets a model reason about geometry ("the left column") rather than only identity. The module-type match is gated on uniqueness: if two panes are charts, `"chart"` resolves to nothing rather than to an arbitrary one.

### The registry: one conversation, every window

Applying an action requires reaching a `HomePage` that may live in a tab of the main window or in a detached top-level window. FinTer solves this with a process-wide GPUI `Global` registry holding weak handles:

```rust
/// One open dashboard, reachable from any window.
#[derive(Clone)]
pub struct DashboardTarget {
    pub id: String,
    pub name: String,
    /// True when the dashboard is rendered in its own top-level window.
    pub detached: bool,
    pub home: WeakEntity<HomePage>,
}

#[derive(Default)]
pub struct DashboardRegistry {
    targets: Vec<DashboardTarget>,
}

impl Global for DashboardRegistry {}
```

`WeakEntity` rather than `Entity` is the correct choice: the registry must not keep a closed dashboard's view alive. `live()` filters to entries whose `home.upgrade()` still succeeds, so a closed window silently drops out of the addressable set instead of panicking on a stale handle.

Resolution handles the selectors a model actually emits:

```rust
match selector {
    Some("all") | Some("*") => live,
    None | Some("active") | Some("current") => {
        let selected: Vec<DashboardTarget> = live
            .iter()
            .filter(|target| Some(target.id.as_str()) == active)
            .cloned()
            .collect();
        if selected.is_empty() {
            live.into_iter().take(1).collect()
        } else {
            selected
        }
    }
    // ...
}
```

Note the fallback: if `"active"` resolves to nothing (the conversation's dashboard was closed mid-session), it degrades to the first live dashboard rather than failing. Every action routes through the same `set_tree` path the mouse controls use, so an agent-driven layout change and a hand-dragged one are the same operation with the same invariants.

### Receipts: apply once, never replay

The subtle correctness problem with agent-driven UI is idempotency. If a reply is re-rendered from history — after a restart, a tab switch, a scroll — does the layout change apply again?

No. Replies apply their directives **exactly once on arrival**, and the resulting `UiActionResult` receipts persist with the message, exactly like a trade receipt:

```rust
pub struct UiActionResult {
    pub action: String,
    pub ok: bool,
    #[serde(default)]
    pub dashboard: Option<String>,
    #[serde(default)]
    pub pane: Option<String>,
    pub message: String,
}
```

Reloading history re-renders the receipts; it does not re-execute the actions. That distinction is the whole reason the receipt exists as a persisted record rather than an ephemeral toast.

Failed actions report the panes that *do* exist, so the model can self-correct from the error rather than guessing again:

> If a reference does not resolve, the receipt names the panes that do exist — correct yourself from that and retry once.

### Context budgeting that protects the control block

An agent snapshot containing full candle arrays for several dashboards will blow any context budget. `shrink_context` trims it — but the order of trimming is a correctness decision, not a performance one:

> Candle arrays dominate the snapshot, so they are dropped first, then whole market-data blocks. The control block is never touched — without it the model cannot address a pane and every `ui` directive would fail.

Dropping the `control` catalog to save tokens would leave the model able to describe a layout but unable to address a single pane in it. The cheapest thing to keep is the thing that makes the feature work at all.

---

## A Smaller Fix With Real Consequences: Forming Candles on Wall-Clock Time

The fourth commit in this window, `5a1083f`, is a chart fix — and it illustrates a class of bug that only appears in live market data.

A forming candle is the in-progress bar at the right edge of the chart. Its open is fixed, its close is the latest price, and its high/low ratchet as trades arrive. When the interval boundary passes, that bar must close and a new one must open.

The failure mode: if the rollover depends on an incoming WebSocket message, then a quiet market produces no new bar. On a low-liquidity pair at 03:00 UTC, the 1-minute candle simply stops advancing. The chart looks frozen even though the clock is not.

The fix drives rollover from a clock tick independent of the feed. `drain_loop` selects on a 250 ms interval alongside the OHLCV and ticker receivers:

```rust
// Periodic wall-clock tick so the live candle rolls over even when the
// market is quiet and neither klines nor ticker ticks arrive.
let mut clock_interval = tokio::time::interval(std::time::Duration::from_millis(250));
clock_interval.tick().await;

loop {
    tokio::select! {
        _ = clock_interval.tick() => {
            let now_ms = std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)
                .map(|d| d.as_millis() as i64)
                .unwrap_or(0);
            let mut inner = buffer.lock().expect("CandleFeed mutex poisoned");
            if check_and_form_live_candle(&mut inner, interval_ms, &timeframe, now_ms) {
                inner.version = inner.version.saturating_add(1);
                inner.last_update_ms = now_ms;
            }
        }
        batch_res = rx.recv() => { /* ... */ }
    }
}
```

The same check also runs on UI snapshots, so a chart that is being rendered but receiving nothing still advances.

Two guards keep the backfill honest. First, a bounded gap bridge: if the feed has been idle for a very long time — or carries a synthetic or epoch timestamp — the code skips forward to the current bucket instead of appending thousands of empty candles:

```rust
const MAX_BACKFILL_PERIODS: i64 = 500;

// ...
let periods_behind = (current_bucket.saturating_sub(next_candle_ts)) / interval_ms;
if periods_behind > MAX_BACKFILL_PERIODS {
    next_ts = current_bucket;
}
```

Second, closed bars get reconciled against REST 1.5 seconds after they close, because the exchange needs a moment to finalize volume and OHLCV. The chart panel detects that a new candle appeared and schedules the refresh on the background executor:

```rust
// A new candle appeared → the previous one just closed. Ask upstream to
// re-fetch after a short delay so the exchange has finalized the bar.
if self.last_completed_candle_ts > 0 && latest_ts > self.last_completed_candle_ts {
    if let Some(refresh) = &self.on_refresh_latest {
        let refresh = refresh.clone();
        cx.background_executor()
            .spawn(async move {
                // Wait 1.5s for upstream exchange to finalize the closed bar
                tokio::time::sleep(std::time::Duration::from_millis(1500)).await;
                refresh();
            })
            .detach();
    }
}
```

There is also a comment in that file worth quoting for what it prevents. A previous approach fed the mark price from a venue-agnostic `TradeStore` into the forming candle:

> Do not overwrite it with `TradeStore::get_mark_price(&self.symbol)` — that lookup is venue-agnostic, so a ticker from another exchange could corrupt this chart, and ratcheting `last.low`/`last.high` on every render frame permanently stretches the wicks.

Two distinct bugs in one sentence: a cross-exchange price contaminating a venue-specific chart, and a per-frame ratchet that monotonically widens the wick until the candle is unreadable. Both are the kind of defect that survives casual testing because the chart still *renders*.

The commit passes all 21 `candle_feed` tests.

---

## Why This Matters Beyond the Terminal

Three of these changes generalize to anyone building a desktop application with a cloud component.

**Keep the local copy authoritative and make sync optional.** FinTer's sync is a button, not a daemon. That inverts the usual failure model: a network partition degrades a feature instead of blocking the application. It also means the sync algorithm can be simple, because it never has to be correct under continuous concurrent writes.

**Put the conflict rule in pure functions.** `pull_action` and `push_action` are testable without a network or a filesystem, which is why the symmetry property has an actual test rather than a comment asserting it. Distributed edge cases are exactly the ones that escape manual testing.

**Validate identifiers, not paths.** An allowlist on the id character set closes path traversal before a `PathBuf` exists. Trying to sanitize a path after joining is the harder and less reliable direction.

For traders, the practical payoff is narrower and more immediate: your workspace is a file. It survives an outage, it survives a backend that no longer exists, and if you want to back it up or check it into a private repo, you can — it is JSON in a directory you own. That is the [local-first](https://martin.kleppmann.com/papers/local-first.pdf) promise stated concretely: the bytes that comprise your data live on your machine, and the service is a peer rather than a landlord.

---

## Build and Run FinTer

FinTer is MIT-licensed and builds from source. Requirements are a stable Rust toolchain with Cargo, plus the platform libraries GPUI needs.

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
```

Guest mode requires no account and no backend. Dashboards are created, renamed, and deleted entirely on local disk under `~/.config/finter/dashboards/`. Cloud sync, when you want it, is a single explicit action from the dashboard manager toolbar.

---

## Conclusion

The headline features of a trading terminal are charts and order books. The features that determine whether you actually keep using it are smaller: does it remember my layout, does it work when the network does not, can I get my data out.

FinTer's 15 November commits answer all three by treating dashboard state as a file rather than a row. One JSON document per dashboard, written atomically through a temp-file rename, identified by a validated UUID, tolerant of corruption, migrated from the old tables without a flag day, and synced to the cloud only when asked — with a conflict rule simple enough to prove and symmetric enough not to thrash.

On top of that storage layer, the agent can now rearrange any pane grid in any window, because the wire format is typed, the registry reaches across windows through weak handles, and the receipts make every action apply exactly once.

Explore the code, star the repository, or contribute:

- **GitHub Repository:** [https://github.com/coin-watch/FinTer](https://github.com/coin-watch/FinTer)
- **Download & Live Market App:** [https://coinwat.ch](https://coinwat.ch)
- **Architecture Deep Dive:** [FinTer Architecture: Building a High-Performance, Local-First Crypto Market Terminal in Rust and GPUI](https://coinwat.ch/blog/finter-rust-gpui-crypto-terminal-architecture)
- **Security Deep Dive:** [Securing Crypto Exchange API Keys at Rest: AES-256-GCM, OS Keyrings, and In-Chat Charting in FinTer](https://coinwat.ch/blog/finter-crypto-terminal-api-security-aes-256-gcm-rust)
- **Related Market Analysis:** [Order Flow Imbalance (OFI) in Crypto Microstructure](https://coinwat.ch/blog/order-flow-imbalance-crypto-microstructure-algorithmic-trading), [Autonomous Trading vs Algorithmic Trading](https://coinwat.ch/blog/ai-agents-autonomous-crypto-trading), and [The Comprehensive Data Toolkit for Algorithmic Trading](https://coinwat.ch/blog/comprehensive-crypto-data-toolkit-algo-trading-en)

---

## Sources & Technical References

1. [FinTer GitHub Repository — coin-watch/FinTer](https://github.com/coin-watch/FinTer) — source for `src/data/dashboard_files.rs`, `src/data/dashboard_sync.rs`, `src/data/dashboard_migration.rs`, `src/data/pane_control.rs`, `src/data/pane_tree_describe.rs`, `src/components/ui/pane_control.rs`, `src/data/candle_feed.rs`, and `src/components/ui/chart_panel.rs`
2. [Kleppmann, M., Wiggins, A., van Hardenberg, P., & McGranaghan, M. (2019). Local-First Software: You Own Your Data, in Spite of the Cloud. Onward! '19, ACM SIGPLAN](https://martin.kleppmann.com/papers/local-first.pdf)
3. [Local-first software — Ink & Switch essay](https://www.inkandswitch.com/essay/local-first/)
4. [rename(2) — Linux Programmer's Manual](https://man7.org/linux/man-pages/man2/rename.2.html)
5. [Path Traversal — PortSwigger Web Security Academy](https://portswigger.net/web-security/file-path-traversal)
6. [Input Validation Cheat Sheet — OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
7. [File Upload Cheat Sheet — OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
8. [GPUI Framework — Zed Industries](https://gpui.rs/)
9. [The Rust Programming Language — rust-lang.org](https://doc.rust-lang.org/)
10. [Tokio Asynchronous Runtime for Rust — tokio.rs](https://tokio.rs/)

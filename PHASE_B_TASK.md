# Phase B — Re-verify, Risk Enforcement, Order Review/Execution, and Logging (Automated Daily Task)

_Part of [FriesTrader](https://github.com/roy-songzhe-li/FriesTrader), forked from YizhiSong/FriesTrader. MIT License._

Automated second half of this pipeline (see `README.md`), run every
weekday 8:35am Central (5 min after 9:30am ET open) as a cloud routine.
Performs **Steps 4–9**, consuming candidates from Phase A's
`pending_proposals.jsonl`.

Authorized to place real live orders under a narrow condition (see Step
6's "Live-order gate" — Step 8 reuses the identical gate for buys).
**That authorization must be given explicitly, in advance, by whoever
operates this pipeline — after being warned that an unattended scheduled
task has no human confirmation at the moment of execution.**
Do not add, remove, or loosen any gate condition on your own judgment.

**Moomoo MCP:** All broker calls go through the `moomoo-api-mcp` MCP server
(`uvx moomoo-api-mcp`). Stock codes use `"<MARKET>.<SYMBOL>"` format (e.g.
`"US.AAPL"`). Account is identified by `acc_id` and `trd_env` read from
`risk_rules.json`. **For REAL accounts (`trd_env="REAL"`)**: call
`unlock_trade()` at the very start of the session before any account or
trading tools — this unlocks REAL account access for the session. For
SIMULATE accounts: unlock is not needed.

## Step 0 — Load state (do this first, every run)

1. Read `risk_rules.json` **fresh** — never cache across runs. Extract
   `acc_id`, `trd_env`, `execution.mode`, and all risk parameters.
2. If `trd_env == "REAL"`, call `unlock_trade()` now, before any other
   account tool. This uses the `MOOMOO_TRADE_PASSWORD` environment variable
   set in the MCP config. If `unlock_trade` fails, abort the run and log
   the error — do not attempt REAL orders without confirmed unlock.
3. Call `get_accounts()` to verify the account and resolve the acc_id. Find
   the account matching `risk_rules.json`'s `acc_id`. If no match is found,
   abort and log — never fall back to account `"0"` silently.
4. Determine today's day of week mechanically (e.g.
   `TZ='America/Chicago' date +'%A'`) — don't infer from the date string.
   Needed for Step 7's weekend-gap check.
5. Read `pending_proposals.jsonl` (overwritten each Phase A run, holds only
   the latest run — use its `"stage": "thesis"` entries directly as today's
   candidates). If missing or empty, log a `cycle_summary` noting nothing
   to process and stop — don't error.
6. Read `trade_log.jsonl` (if present):
   - **Idempotency — key off the proposal's own `date`, not today's.**
     Skip a candidate if `trade_log.jsonl` already has a `risk_check`/
     `order` entry for that symbol with a matching `proposal_date`.
   - **Dry-run cycle count**: number of **distinct dates** with a
     `cycle_summary` entry where `mode: dry_run` — not raw entry count
     (same-day reruns count once). This represents validated days, not
     executions, and must be
     `>= execution.dry_run_min_cycles_before_live` before the
     live-order gate (Step 6 for sells, Step 8 for buys) can open.

## Step 4 — Classify candidates

Call `get_positions(trd_env=<trd_env>, acc_id=<acc_id>)` — this snapshot,
taken before any of this cycle's sells execute, is also what Step 5's
stop-loss/take-profit checks use. Positions have fields: `code` (e.g.
`"US.AAPL"`), `qty`, `cost_price` (average cost per share), `market_val`,
`pl_ratio`, `can_sell_qty`.

`direction: "avoid"` candidates aren't processed further. `exit_existing`
candidates (Phase A's recommendation to sell a currently-held position)
go straight into Step 6's sell-execution pass — selling is never gated. Split
the remaining `direction: "long"` candidates, using this snapshot, into:
- **new**: not a live open position — a genuine new entry, the only kind that
  consumes a slot.
- **held**: already a live open position — a potential top-up (Step 7).
  Top-ups never consume a slot and are always considered regardless of
  account fullness.

**This classification stays fixed for the rest of the cycle**, even if a
same-cycle sell later empties the position — so a symbol whose stop-loss
fires this cycle can't silently shift from `held` to `new` by the time
Step 7 runs.

## Step 5 — Sell-side risk enforcement

### Stop-loss check (always runs, independent of new candidates)

Gather inputs, then let the script decide — do not hand-compute the reference
price, drawdown, stdev, or clamp. For each open position (the snapshot pulled
in Step 4):

- Pull a fresh quote: `get_stock_quote(codes=["US.SYMBOL"])` → use `last_price`
  as `current_price`.
- Check `trade_log.jsonl` for whether any `take_profit` tier has fired for this
  position's current holding period (since qty last reached zero).
- If a tier has fired, pull daily `high` bars via `get_historical_klines`:
  ```
  get_historical_klines(
      code="US.SYMBOL",
      ktype="K_DAY",
      start="<entry_date>",
      end="<yesterday>",
      max_count=500,
      autype="QFQ"
  )
  ```
  Use the `high` field from each bar for `--daily-highs`.
- If `risk_rules.json`'s `stop_loss.mode` is `"volatility_scaled"` and the
  position is not currently showing a gain on average cost, pull the last
  `stop_loss.volatility_lookback_trading_days` trading days of daily closes
  via `get_historical_klines` (ktype="K_DAY", request ~30 calendar days back,
  use `close` field from each bar), oldest first through yesterday, for
  `--daily-closes`. Skip this pull on a gain — the script itself also skips
  the computation in that case.

Run:
```
python3 scripts/stop_loss.py \
  --average-cost <cost_price from get_positions> \
  --current-price <last_price from get_stock_quote> \
  --mode <stop_loss.mode> \
  --hard-stop-pct <stop_loss.hard_stop_pct> \
  --volatility-multiplier <stop_loss.volatility_stdev_multiplier> \
  --min-stop-pct <stop_loss.min_stop_pct> \
  --max-stop-pct <stop_loss.max_stop_pct> \
  --fallback-stop-pct <stop_loss.fallback_stop_pct> \
  --min-bars 10 \
  [--daily-closes <comma-separated close values>] \
  [--take-profit-tier-fired --daily-highs <comma-separated high values> --trailing-high-since <entry date>]
```
Use its JSON output directly (`stop_reference_basis`, `stop_reference_price`,
`drawdown_pct`, `stop_pct_used`, `stdev_20d` when computed, `fallback_reason`
when applied, `triggered`, `action`) rather than recomputing.

**If the script fails to run**: treat this position as if a loss-limit breach
applied this cycle (`entries_halted = true`) and log `"stage": "stop_loss"`
with `"stop_pct_used": null, "triggered": false, "action": "halt_entries_check_manually",
"notes": "stop_loss.py failed to run — verify this position's stop manually before next cycle"`.

If `triggered` is true — immediate full-position sell, no thesis review,
never blocked by a loss-limit halt, executed in Step 6's sell-execution pass.
Log `"stage": "stop_loss"` with the script's outputs.

### Take-profit check (always runs, independent of new candidates)

Using the same pull as stop-loss (no need to call again — `cost_price`,
`qty`, and fresh `last_price` as at start of this step), check
`trade_log.jsonl` for `"stage": "take_profit"` entries for this symbol at
each tier's exact `gain_pct`, logged since the position's quantity last
reached zero — collect the `gain_pct` values already fired this holding
period.

Run:
```
python3 scripts/take_profit.py \
  --average-cost <cost_price from get_positions> \
  --current-price <last_price from get_stock_quote> \
  --quantity <qty from get_positions> \
  --tiers <risk_rules.json take_profit.tiers as "gain_pct:sell_fraction" pairs, e.g. "0.15:0.25,0.30:0.25,0.50:0.25"> \
  [--already-fired <comma-separated gain_pct values already fired this holding period>]
```
Use its JSON output directly. If the script fails, treat it like a stop-loss
script failure: `entries_halted = true`, exclude this position from any sell
decision.

If `fired_this_cycle` is non-empty — each fired tier is executed in Step 6's
sell-execution pass, no thesis review, never blocked by a loss-limit halt.
Log one line per entry.

### Conviction-trim check (held positions only)

Skip entirely if `risk_rules.json`'s `conviction_trim.enabled` is `false`.
Otherwise, for every **held** position, gather this cycle's fresh `conviction`
and `target_size` (from Step 7's priority-order inputs — pull those first if
not yet available), look back through this symbol's `risk_check` entries in
`trade_log.jsonl`, most recent first, and count consecutive entries where
`conviction == "low"` and `current_position_value` exceeded `target_size` by
more than `conviction_trim.overweight_trigger_pct` (not including this cycle,
not crossing back over a prior full exit).

Run:
```
python3 scripts/conviction_trim.py \
  --conviction <this cycle's conviction> \
  --current-position-value <market_val from get_positions> \
  --target-size <target_size> \
  --overweight-trigger-pct <conviction_trim.overweight_trigger_pct> \
  --prior-consecutive-low-overweight-cycles <count from lookback> \
  --min-low-conviction-cycles <conviction_trim.min_low_conviction_cycles>
```
Use its JSON output directly. Script failure → `entries_halted = true`.

If `triggered` is true — sell `trim_dollar_amount` worth of the position
(round to integer shares), down to `target_size`, executed in Step 6's
sell-execution pass. Log `"stage": "conviction_trim"`.

## Step 6 — Sell-side execution

**Execute sells now — every stop-loss trigger, fired take-profit tier,
conviction-trim trigger, and Step 4's `exit_existing` candidates:** run this
procedure for each, before touching anything buy-side.

**Pre-flight check (replaces Robinhood's `review_equity_order`):** call
`get_max_tradable(order_type="NORMAL", code="US.SYMBOL", price=<bid_price>,
trd_env=<trd_env>, acc_id=<acc_id>)`. Returns `max_position_sell` and
`max_cash_buy`. Verify `max_position_sell >= qty` for sells. If the check
returns fewer shares than needed, log this as a blocking alert and do not
proceed to placement for that order — treat it the same as a blocking alert
from Robinhood's `review_equity_order` (skip placement, log the alert
verbatim). Pull the current `bid_price` from `get_stock_quote` to use as
the limit price for sell orders.

Branch on `execution.mode` (fresh from Step 0) and the dry-run cycle count:

**Live-order gate — ALL must be true:**
- `execution.mode == "live"`
- dry-run cycle count `>= execution.dry_run_min_cycles_before_live`
- `get_max_tradable` for this order returned no blocking alert

- **Gate open**: Call `place_order` with these parameters for a SELL:
  ```
  place_order(
      code="US.SYMBOL",          # e.g. "US.AAPL"
      price=<bid_price>,         # current bid from get_stock_quote
      qty=<integer shares>,
      trd_side="SELL",
      order_type="NORMAL",       # limit order at bid
      time_in_force="DAY",
      trd_env=<trd_env>,
      acc_id=<acc_id>
  )
  ```
  Returns `order_id` and `order_status`. Then confirm the real fill:
  1. Wait ~15 seconds.
  2. Call `get_orders(code="US.SYMBOL", trd_env=<trd_env>, acc_id=<acc_id>)`.
     Find the entry whose `order_id` matches. Check `order_status` for terminal
     states: `FILLED_ALL`, `CANCELLED_ALL`, `REJECTED`, `FAILED`, `DELETED`,
     `FILLED_PART`.
  3. If terminal, use that state. Otherwise wait ~15 more seconds and check
     once more via `get_orders` — never poll more than twice or block
     indefinitely waiting for a fill.
  4. For filled orders, call `get_deals(code="US.SYMBOL", trd_env=<trd_env>,
     acc_id=<acc_id>)` to get the actual execution price: find deals with
     matching `order_id`, use their `price` as `fill_price` and `qty` as
     `fill_quantity`.
  Log `"stage": "order", "mode": "live", "placed": true, "order_id": "<id>",
  "order_state": "<confirmed state>", "fill_price": <price if filled, else null>,
  "fill_quantity": <qty if filled, else null>` in addition to the pre-trade
  `quote_bid`/`quantity` estimate.

- **`execution.mode == "dry_run"`**: log
  `"stage": "order", "mode": "dry_run", "would_execute": true"` and stop.
  **Never call `place_order` here.**

- **`execution.mode == "live"` but cycle count under threshold**: log
  `"stage": "order", "mode": "live_blocked_insufficient_cycles",
  "would_execute": true, "placed": false"` with current vs. required count.

**Wash-sale flag on sells (informational only, never blocks a sell):** whenever
a sell realizes a loss, check `trade_log.jsonl` for purchases of that symbol
within `wash_sale_avoidance.lookback_window_days` days before today, in any
of the linked accounts (`wash_sale_avoidance.linked_accounts`). A qualifying
purchase alone is not enough — confirm via `get_positions` (called for each
`linked_accounts` acc_id) that at least one account still holds a nonzero
quantity sourced from a purchase inside the lookback window after this sale.
Only then add the wash-sale flag to that sell's `order` log entry. Purely
informational — never blocks, delays, or resizes the sell.

**Re-pull fresh account state:** call
`get_assets(trd_env=<trd_env>, acc_id=<acc_id>)` (for `total_assets` and `cash`)
and `get_positions(trd_env=<trd_env>, acc_id=<acc_id>)` (for the live open
position count and updated quantities) again — the earlier pull is now stale.
Map `total_assets` → `total_value` and `cash` → `cash` for the computations
below. In `dry_run` mode these won't have changed (nothing real executed),
which is expected.

## Step 7 — Buy-side risk enforcement and sizing

**Compute capacity:** `open_slots = max_concurrent_positions - len(get_positions(...))`.
Only **new**-group candidates consume a slot; fixed for the rest of this cycle.

**If `open_slots <= 0`**: no **new** candidate can be approved this cycle. Skip
the weekend-gap search and buy gate below for every **new**-group candidate —
log `"stage": "risk_check", "passed": false, "reason":
"no open slots this cycle (X of Y max already held/approved) — skipped without
staleness re-check"`. **held** group is unaffected.

**No same-cycle sell-then-buy**: if a symbol's stop-loss fired, any take-profit
tier fired, or a conviction-trim fired earlier this cycle, drop it from the
**held** group before continuing — log `"stage": "risk_check", "passed": false,
"position_action": "top_up", "reason": "stop-loss/take-profit fired this cycle —
not eligible for a same-cycle top-up"`. Normal top-up candidate again next cycle.

### Weekend gap (Monday runs only)

**If today is Monday:** before the price-staleness check, run one additional
targeted web search per pending proposal covering Saturday/Sunday (earnings,
M&A, guidance, macro) — separate from and not counted against
`cadence.news_search_budget_per_cycle`. If anything materially contradicts
the thesis/invalidation criteria, drop it — log `"stage": "risk_check",
"passed": false, "reason": "weekend news invalidated thesis: <what you found>",
"sources": ["Outlet Name: https://..."]`. Otherwise proceed to the price-based check.

### Buy gate (every day)

**Buys only — new entries and top-ups; never applies to
stop_loss/take_profit/conviction_trim/exit_existing sells.**

Pull a fresh quote: `get_stock_quote(codes=["US.SYMBOL"])` — re-verify against
this morning's open. Use `ask_price` as `fresh_ask`.

**Gather inputs, then let the script decide:**
- `--fresh-ask`: `ask_price` from `get_stock_quote`.
- `--thesis-price`: Phase A's thesis-time `current_price` (from
  `pending_proposals.jsonl` screened entry).
- `--daily-closes`: last `entry_extension.lookback_trading_days` trading days of
  daily closes via `get_historical_klines` (ktype="K_DAY", request ~30 calendar
  days back, use `close` field from each bar — all returned bars are
  split-adjusted, no interpolation flag to filter).
- Wash-sale inputs, only if `wash_sale_avoidance.enabled` is `true` (pass
  `--wash-sale-enabled` and `--wash-sale-lookback-days`): for every account in
  `wash_sale_avoidance.linked_accounts`, scan `trade_log.jsonl` for sell orders
  of this symbol that realized a loss (where `fill_price < entry_price` from the
  corresponding `stop_loss` or `take_profit` stage entry for that holding period),
  within the lookback window. Pass their dates as
  `--loss-sale-dates <comma-separated ISO dates>`. Also pass `--today <today's date, ISO>`.
- Sell re-entry lock inputs, whenever `trade_log.jsonl` has this symbol's most
  recent `order` entry as a sell that actually executed (in live mode: verify via
  `get_positions` that qty is genuinely lower; in dry_run: a logged dry-run sell
  never reduced a real position, so omit these inputs). If it did execute, pass
  `--last-sell-reason`, `--last-sell-price` (that entry's `quote_bid`),
  `--last-sell-date`, and `--last-sell-was-gain` (compare `fill_price` to the
  corresponding `stop_loss`/`take_profit` stage entry's `entry_price` for this
  symbol). If gain-closed, also pass `--reentry-lock-max-trading-days` and
  `--trading-days-since-sell`.

Run:
```
python3 scripts/entry_gate.py \
  --fresh-ask <ask_price> \
  --thesis-price <current_price from pending_proposals> \
  --entry-price-gap-max-pct <entry_price_gap.max_pct> \
  --daily-closes <comma-separated close values> \
  --max-extension-pct <entry_extension.max_extension_pct> \
  [--wash-sale-enabled --wash-sale-lookback-days <N> --loss-sale-dates <dates>] \
  --today <date> \
  [--last-sell-reason <reason> --last-sell-price <price> --last-sell-date <date> \
   --last-sell-was-gain <true|false> \
   [--reentry-lock-max-trading-days <N> --trading-days-since-sell <N>]]
```
Use its JSON output directly (`entry_price_gap`, `entry_extension`,
`wash_sale_avoidance`, `sell_reentry_lock`, `passed`, `blocking_conditions`,
`action`).

**If the script fails to run**: skip the buy this cycle and log
`"stage": "risk_check", "passed": false, "reason":
"entry_gate.py failed to run — buy skipped this cycle, verify manually"`.

**If `passed` is false**, skip the buy. Log one `"stage": "risk_check",
"passed": false` line per entry in `blocking_conditions` with the matching
reason text.

**If `passed` is true but `entry_price_gap.gap_pct` is non-trivial**: re-check
against the thesis's `invalidation` criteria via web search; if plausibly
invalidated, drop it with `"stage": "risk_check", "passed": false`.

**Loss-limit halt check (always runs, gates all new entries and top-ups):**

Compute realized P&L from `trade_log.jsonl` — this pipeline logs every sell's
`fill_price`, `fill_quantity`, and the corresponding `stop_loss`/`take_profit`
stage entry logs the `entry_price` (average cost). For each sell `order` entry
in `trade_log.jsonl`:
- **Daily P&L**: sum `(fill_price - entry_price) * fill_quantity` for all sell
  order entries whose `date == today`.
- **Weekly P&L**: same, for all sell order entries in the current Mon–Fri week.
Match each sell order to its corresponding `stop_loss`/`take_profit`/
`conviction_trim` stage entry for the same symbol and holding period to get
`entry_price`. If no matching stage entry can be found for a sell, skip it from
the P&L sum and log a warning — do not guess. **Fail safe: if P&L cannot be
determined cleanly, treat as breached** (`entries_halted = true`).

Run:
```
python3 scripts/pnl_pct.py \
  --daily-realized-usd <computed daily P&L> \
  --weekly-realized-usd <computed weekly P&L> \
  --starting-capital-usd <risk_rules.json starting_capital_usd> \
  --daily-limit-pct <loss_limits.daily_loss_limit_pct_of_account> \
  --weekly-limit-pct <loss_limits.weekly_loss_limit_pct_of_account>
```
Use its JSON output directly (`daily_pnl_pct`, `weekly_pnl_pct`,
`entries_halted`, `halt_reason`). Script failure → fail safe, treat as breached.
Log as `"stage": "loss_limit_check"`.

**Candidate priority order:** Merge **new** and **held** groups (excluding
candidates already rejected) into one list. Gather for every candidate:
`symbol`, `conviction`, `risk_flags`, `pct_below_52wk_high`, `group`
(`"new"` or `"held"`), and for **held** only: `current_position_value`
(`qty × last_price` using re-pulled values). Also use the re-pulled
`total_assets` and `cash` from `get_assets`, and `concurrent_positions_start`
from the `open_slots` computation.

Rank and size:
```
python3 scripts/rank_candidates.py | python3 scripts/position_sizing.py \
  --total-value <total_assets> \
  --cash-start <cash> \
  --concurrent-positions-start <concurrent_positions_start> \
  [--entries-halted] \
  --max-position-pct <position_sizing.max_position_pct_of_account> \
  --max-concurrent-positions <position_sizing.max_concurrent_positions> \
  --min-cash-buffer-pct <position_sizing.min_cash_buffer_pct> \
  --min-top-up-usd <position_sizing.min_top_up_usd> \
  --min-top-up-pct-of-target <position_sizing.min_top_up_pct_of_target> \
  --conviction-pct "high:0.20,medium:0.12,low:0.06"
```
`rank_candidates.py` sorts by conviction tier, then `risk_flags` count
ascending, then `pct_below_52wk_high` descending. `position_sizing.py`
processes in ranked order, compounding running cash/concurrency totals.
Pass `--entries-halted` whenever the loss-limit check or a script failure
halted entries this cycle.

Use the final script's JSON output directly. Script failure → reject every
pending candidate this cycle.

For each candidate, log `"stage": "risk_check"` with that candidate's
`results` entry fields verbatim. Every `risk_check` entry must include
`proposal_date` and, for `direction: "long"`, `risk_flags` and
`pct_below_52wk_high`. Top-up entries must also include
`"position_action": "top_up"`.

## Step 8 — Buy-side execution

**Execute approved buys:** for every candidate Step 7's sizing approved
(new entries and top-ups), in ranked order.

**Pre-flight check:** call `get_max_tradable(order_type="NORMAL",
code="US.SYMBOL", price=<ask_price>, trd_env=<trd_env>, acc_id=<acc_id>)`.
Returns `max_cash_buy`. Verify `max_cash_buy >= qty`. If not, log blocking
alert and skip placement.

**Same Live-order gate as Step 6. Gate open → BUY:**
```
place_order(
    code="US.SYMBOL",
    price=<ask_price>,       # current ask from get_stock_quote
    qty=<integer shares>,
    trd_side="BUY",
    order_type="NORMAL",     # limit order at ask
    time_in_force="DAY",
    trd_env=<trd_env>,
    acc_id=<acc_id>
)
```
Then confirm fill using the same `get_orders` → `get_deals` flow as Step 6
(wait ~15s, check once more if not terminal, use `get_deals` for actual fill
price). Log `quote_ask`/`quantity` as the pre-trade estimate alongside the
confirmed order state.

Never change `execution.mode` yourself. Every `order` entry must carry
`proposal_date`.

## Step 9 — Logging

Append every decision to `trade_log.jsonl` — one JSON line each: `stop_loss`,
`take_profit`, `conviction_trim`, `loss_limit_check`, `risk_check` (pass/fail,
including weekend-gap rejections, buy-gate rejections, top-up evaluations), and
`order` stages, matching the shape in `trade_log_template.jsonl`. Top-up
entries must include `"position_action": "top_up"`.

**Every line — including the final `cycle_summary` — needs a real `"timestamp"`**
(`HH:mm:ss`, e.g. via `TZ='America/Chicago' date +'%H:%M:%S'`). Separate from
`"date"`/`"proposal_date"` — for readability only, never used for idempotency
or dry-run count.

**Always append exactly one final line per run:**
```json
{"date": "YYYY-MM-DD", "timestamp": "HH:mm:ss", "stage": "cycle_summary", "mode": "dry_run|live", "candidates_considered": N, "orders_reviewed": N, "orders_placed": N}
```
Load-bearing — Step 0's dry-run cycle count depends on this line.

**After appending, regenerate `trade_log_recent.md`** (full overwrite) — a
short, plain-English recap of today's cycle for a quick read: date heading,
prose/bullet sections covering only what actually happened this cycle
(skip anything empty): the loss-limit check result; each held position's
stop-loss/take-profit status; each new-entry/top-up candidate considered and
its outcome; and any orders actually placed (symbol, buy/sell, dollar amount).
This is a readable render of what this cycle already decided — `trade_log.jsonl`
is still the source of truth.

## Hard rules

- Never change `execution.mode`, `trd_env`, or any `risk_rules.json` value.
- Never call `place_order` unless the live-order gate (Step 6 for sells,
  Step 8 for buys) is open at that moment.
- A "high conviction" thesis never overrides a failed mechanical check.
- If required data can't be retrieved (positions, assets, P&L), fail safe —
  treat the check as failed/halt new entries — and log exactly what failed.
- The wash-sale guard only ever blocks a buy. It must never block, delay, or
  resize a stop_loss/take_profit/conviction_trim/exit_existing sell — risk
  management never bends to a tax outcome.
- For REAL accounts: never call trading tools (`get_positions`, `get_assets`,
  `place_order`, etc.) without first confirming `unlock_trade` succeeded in Step 0.

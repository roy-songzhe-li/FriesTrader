# Phase A — Screening & Thesis Only (Automated Daily Task)

_Part of [FriesTrader](https://github.com/roy-songzhe-li/FriesTrader), forked from YizhiSong/FriesTrader. MIT License._

Automated subset of this pipeline (see `README.md`), run every weekday
4:30pm Central as a cloud routine.

Performs **ONLY Steps 1–3**. **NEVER** Step 4 (re-verify), 5 (risk
enforcement), 6 (dry run/order review), or 7 (`trade_log.jsonl`) — those
belong to Phase B. Order tools (`place_order`, `cancel_order`, `modify_order`)
are hard-blocked at the connector level; do not attempt them anyway.

**Moomoo MCP:** All broker calls in this phase go through the
`moomoo-api-mcp` MCP server (`uvx moomoo-api-mcp`). Stock codes use
`"<MARKET>.<SYMBOL>"` format throughout (e.g. `"US.AAPL"`, `"US.MSFT"`).
This session does **not** need `unlock_trade` — Phase A only reads market
data and positions, not trade-sensitive account data.

## Step 1 — Build the watchlist

Read `risk_rules.json`'s `universe.watchlist_name`, `moomoo_market`, `acc_id`,
and `trd_env` fresh each run — never cache or hardcode these values.

**Primary universe — Moomoo watchlist:** Call `get_user_security_group(group_type=1)`
to get the list of custom security groups. Find the group whose `group_name`
matches `universe.watchlist_name`. Then call
`get_user_security(group_name=<matched name>)` to fetch its symbols — each
returned as a dict with `code` (e.g. `"US.AAPL"`) and `name`. These are
your watchlist candidates. If no group matches the name exactly, log a
hard error and stop — do not guess or fall back to another group.

**Supplementary web scan** (additive, not a replacement): run a web search
for today's notable US market movers (e.g. query like `"US stock market
notable movers high volume <date>"`). From the search results, identify up to
`universe.supplementary_scan_max_candidates` US stocks with genuine notable
activity (high relative volume, material catalyst, earnings reaction) that
are **not** already on your watchlist and **not** already a held position.
Format each symbol as `"<moomoo_market>.<SYMBOL>"` (e.g. `"US.AMD"`).
Mark each with `"source": "market_scan"` on its `screened` line
(watchlist-sourced and held candidates get `"source": "watchlist"`).
This cap is **separate from and additive to** `watchlist_max_candidates` —
scan results never compete with watchlist candidates for the same slots.

Dedupe the combined list, then apply the universe filters below, and cap the
**watchlist-sourced, non-held** portion at `universe.watchlist_max_candidates`
(the scan's own separate cap above already bounds its contribution, so this
cap only applies to watchlist candidates).

**Pull market snapshots** for the full candidate list (before filtering) in
one batched call: `get_market_snapshot(codes=["US.AAPL", "US.MSFT", ...])`.
This returns per-stock: `last_price`, `highest_52weeks_price`,
`lowest_52weeks_price`, `total_market_val` (market cap in USD),
`volume` (today's volume), `volume_ratio` (current vs. avg ratio), `name`,
`turnover_rate`, and additional fundamental fields. Use `last_price` as
`current_price` in Steps 2–3.

**Pull fresh quotes** for the capped candidate list via
`get_stock_quote(codes=["US.AAPL", ...])`. Returns `last_price`, `ask_price`,
`bid_price`, `volume`, `open_price`, `high_price`, `low_price`,
`prev_close_price`. Use `last_price` as `current_price` in Steps 2–3 — this
is more real-time than snapshot for the final candidate list.

**Pull price history** per candidate via `get_historical_klines`:
```
get_historical_klines(
    code="US.AAPL",
    ktype="K_DAY",
    start="<today minus 430 calendar days>",   # enough for 200-bar MA + buffer
    end="<today>",
    max_count=350,
    autype="QFQ"   # forward split-adjusted, default
)
```
Returns list of bars with fields `time_key`, `open`, `high`, `low`, `close`,
`volume`, `turnover`, `change_rate`. Use `close` (not `close_price`) for
price series. No `interpolated` field — bars are already clean
split-adjusted data; use all returned bars.

**Compute `avg_volume_30_days`** from historicals: take the last 30 daily
bars' `volume` values and compute their mean. Do **not** use `volume_ratio`
back-calculation — compute directly from the K-line series so the figure is
auditable and consistent.

**Universe filters** — apply to each candidate using the data pulled above:

`universe.min_avg_daily_volume`: exclude if `avg_volume_30_days < min_avg_daily_volume`.

`universe.min_market_cap_usd` / `universe.max_market_cap_usd`: use
`total_market_val` from `get_market_snapshot`. Exclude if market cap is
below the floor or exceeds the ceiling. Log as
`"market cap $<X> exceeds universe.max_market_cap_usd ($<threshold>) — excluded per universe filters"`.

`universe.penny_stock_filter_enabled`: mechanical exclusion, active only when
true — exclude if `current_price < universe.penny_stock_price_threshold_usd`.
Log: `"penny stock (price $<X>, under $<threshold>) — excluded per universe.penny_stock_filter_enabled"`.

`universe.leveraged_etf_filter_enabled` / `universe.inverse_etf_filter_enabled`:
active only when true. In Moomoo, these are detected by checking the `name`
field from `get_market_snapshot` for identifying keywords (case-insensitive):
- **Leveraged ETF** keywords: `"2x"`, `"3x"`, `"ultra"`, `"ultrapro"`,
  `"leveraged"`, `"daily bull"`, `"bull 3x"`, `"bull 2x"`, `"2x bull"`,
  `"3x bull"`.
- **Inverse ETF** keywords: `"inverse"`, `"short"`, `"bear"`, `"daily bear"`,
  `"bear 3x"`, `"bear 2x"`, `"-1x"`, `"-2x"`, `"-3x"`.
If the `name` does not conclusively indicate leveraged/inverse type, run a
quick targeted web search to confirm the fund type before excluding. Log the
reason as `"leveraged/inverse ETF (name: \"<name>\") — excluded per universe.<filter>"`.

`universe.trend_filter_enabled` / `universe.trend_filter_lookback_trading_days`:
mechanical downtrend exclusion when enabled. Compute the simple moving average
of the trailing `trend_filter_lookback_trading_days` daily closes from the
historicals series. Exclude if `current_price < SMA`. Log as
`"200-day MA $<X>, current price $<Y> (<Z>% below trend) — excluded per universe.trend_filter_enabled"`.
If fewer than `trend_filter_lookback_trading_days` daily bars are available,
skip this check for that candidate and log
`"trend filter skipped — fewer than <trend_filter_lookback_trading_days> daily bars available"`.

**Always include every held position** in the final list.
Call `get_positions(trd_env=<trd_env>, acc_id=<acc_id>)` to get current
positions. Each position has `code` (e.g. `"US.AAPL"`), `qty`, `cost_price`
(average cost per share), `market_val`, `pl_ratio`. If a held position is
already in the candidate list via watchlist or scan, leave it as-is; don't
add a duplicate. `watchlist_max_candidates` caps **non-held** candidates
only — held positions are outside that count and can never crowd out a
non-held candidate or be silently dropped for failing filters. A held
position must stay eligible for a fresh thesis (including `exit_existing`) and
is always logged `"stage": "screened", "passed_filters": true,
"reason": "currently held — always included"` regardless of what filters
would have said.

(This just builds the candidate list — not a risk/stop-loss check; that's
Phase B's job.)

## Step 2 — Gather signals

Use the ~300-day price history per candidate already pulled in Step 1 — no
second pull needed. Take the most recent 60 **calendar** days' worth of bars
for the price-move signal. Use `close` field values throughout.

Whether a candidate qualifies for a news search is mechanical, against
`risk_rules.json`'s `signal_thresholds` — qualifies if it meets **any one**
of these three (no extra tool calls needed):

1. **60-day price move**: `abs(latest_close - close_60d_ago) / close_60d_ago >= signal_thresholds.price_move_60d_pct`.
   **"60 days" = 60 *calendar* days, not trading bars.** Get `close_60d_ago`
   as the earliest bar's `close` when `start = today minus 60 calendar days`.
   Do **not** pull a longer range and count back 60 bars (that drifts to
   ~85–90 calendar days). If fewer than 60 days of history exist, compute
   over the available window and note it rather than skipping.

2. **Volume spike**: `latest_volume / avg_volume_30_days >= signal_thresholds.volume_spike_multiple`.
   `latest_volume` from `get_stock_quote` `volume` field; `avg_volume_30_days`
   from the 30-bar mean computed in Step 1 above.

3. **Near a 52-week extreme**: `(highest_52weeks_price - current_price) / highest_52weeks_price <= signal_thresholds.pct_from_52wk_extreme`
   **or** `(current_price - lowest_52weeks_price) / lowest_52weeks_price <= signal_thresholds.pct_from_52wk_extreme`.
   `highest_52weeks_price` / `lowest_52weeks_price` from `get_market_snapshot`;
   `current_price` from `get_stock_quote` `last_price`.

**Log the raw inputs behind every ratio, not just the ratio** (see
`signal_check` format below) — otherwise it can't be sanity-checked
without re-pulling data.

If none apply, no search/thesis this run — log as `screened`-only.
Qualifying candidates' searches stay within
`cadence.news_search_budget_per_cycle` (per run, not per stock; held
positions draw from their own separate budget, not this one).

**If more candidates qualify than the budget allows**, prioritize by how
far each one exceeded the specific threshold it tripped — not conviction
or `risk_flags` (those don't exist yet; they're outputs of the search
this budget gates). Compute a **magnitude score** per qualifying candidate:
- Price move: `actual_price_move_60d_pct / price_move_60d_pct` (threshold).
- Volume spike: `actual_volume_spike / volume_spike_multiple` (threshold).
- 52-week extreme: `pct_from_52wk_extreme` (threshold) `/ actual_pct_from_52wk_extreme`
  (whichever of the two 52-week-extreme ratios triggered) — inverted, since
  smaller = closer to the extreme = more notable.
If a candidate qualifies under more than one criterion, use its **highest**
score. Process qualifying candidates in descending score order, spending the
budget as you go. Any candidate that would push spend past
`cadence.news_search_budget_per_cycle` is skipped this cycle — log
`"stage": "screened", "passed_filters": true, "reason": "news search budget exhausted this cycle (<N> of <cadence.news_search_budget_per_cycle> already spent on higher-magnitude signals) — no thesis this run"`.

**Every qualifying candidate's search** must explicitly check, in addition to
whatever catalyst-specific query satisfied Step 2:
1. Whether any active lawsuit/regulatory investigation naming the company has a
   scheduled ruling, hearing, trial date, or compliance deadline in the next
   ~90 days (the `active_litigation` risk_flags criterion below).
2. Whether the company has a confirmed accounting restatement, for-cause
   auditor dismissal/resignation, or an indictment/plea involving a current or
   former executive or employee tied to company operations, disclosed within
   the last 3 years (the `governance_history` risk_flags criterion below).
Don't rely on either surfacing incidentally from a catalyst-only search.

**Exception — held positions always get a fresh thesis**, signal or not.
Run one targeted news search per held position (separate budget from
`cadence.news_search_budget_per_cycle`, bounded by
`max_concurrent_positions`, same pattern as Phase B's Monday weekend-gap
searches) and produce a thesis every run — this is what makes
`exit_existing` reachable.

## Step 3 — Synthesize thesis

For each flagged candidate, produce the thesis record from `README.md`
(symbol, date, thesis, conviction, invalidation, direction).
- **No price targets.**
- **No forecasting as fact** — "this suggests..." not "this will...".

**`conviction` follows a fixed rubric, not open judgment** — the same
underlying facts must produce the same rating regardless of which day this
runs. Evaluate fresh each run using only what this run's own research found;
never carry forward or average against a prior day's conviction.

- **`high`** requires **all** of:
  - The catalyst is a specific, already-confirmed, company-disclosed event
    (an earnings result, a signed deal/contract, a completed regulatory
    approval, a disclosed structural risk) — not a rumor, analyst opinion,
    technical pattern, or sector/macro-wide move, and not still
    pending/anticipated (e.g. "ahead of earnings" caps at `medium`).
  - **For a non-held candidate**, that event is recent — its confirming date
    is within the last 15 trading days. **Held positions are exempt.**
  - The thesis explicitly names the strongest available counter-evidence and
    gives a concrete reason it doesn't change the read. The counter-case must
    be **fundamental or structural** — a competitive threat, demand/margin
    risk, execution risk, balance-sheet or liquidity concern, or
    regulatory/legal exposure. **Valuation and price action alone don't
    qualify** as the sole counter-case for `high`.
  - At least one cited source is **primary or wire** — a company filing or
    press release, a regulatory/court document, or a wire service (Reuters,
    AP, Bloomberg, Dow Jones).
  - `risk_flags` is empty.
  - No unresolved binary catalyst falls before this position's next likely
    review that could reverse the read.
- **`low`** applies if **any** of:
  - The thesis itself frames the evidence as mixed, offsetting, or unresolved.
  - The move is explained as technical, mechanical, or sentiment-driven in a
    way that discounts its fundamental significance.
  - Any `risk_flags` entry is present.
  - The catalyst is macro/sector-wide rather than company-specific.
- **`medium`** is everything else: a real, credible, company-specific catalyst
  exists and doesn't hit a `low` disqualifier, but the catalyst is still
  pending, or multiple contributing factors are listed without one clearly
  resolved as dominant, or no counter-case is explicitly engaged and dismissed.

**For a held position**, `direction` is `"long"` (still supports holding)
or `"exit_existing"` (no longer does) — never `"avoid"` (only for not-yet-held).

**Include `risk_flags`** for every `direction: "long"` candidate — an array
of zero or more tags from this fixed set:
- `"active_litigation"` — an active lawsuit or regulatory investigation with a
  specific scheduled ruling, hearing, trial date, or compliance deadline within
  the next ~90 days.
- `"governance_history"` — a confirmed accounting restatement, for-cause auditor
  dismissal/resignation, or indictment/plea involving a current or former
  executive tied to company operations, disclosed within the last 3 years.
- `"dilution_risk"` — a completed or pending equity/convertible raise, share
  offering, or ATM program disclosed in the last ~90 days.
- `"insolvency_or_liquidity_concern"` — bankruptcy rumor, going-concern language,
  or reliance on an external backer to remain solvent.
- `"leadership_turnover"` — a C-suite departure/replacement in the last ~90 days
  tied to operational or execution problems (not routine succession).
Empty array (`[]`) if none apply.

**Include `pct_below_52wk_high`** for every `direction: "long"` candidate:
`(highest_52weeks_price - current_price) / highest_52weeks_price` (e.g. `0.15`).
`highest_52weeks_price` from Step 1's `get_market_snapshot`,
`current_price` from Step 1's `get_stock_quote` `last_price`.

**Include a `sources` field** listing outlet name + URL for every search result
that informed this thesis. Prefer primary sources (company filings/press
releases, wire services like Reuters/AP) and major outlets (Bloomberg, WSJ,
CNBC) over aggregator/content-farm sites. A `high`-conviction thesis
additionally requires at least one primary or wire source.

## Output

**Overwrite `pending_proposals.jsonl` at the start of this run** — it holds
only today's candidates; history remains auditable via `trade_log.jsonl`.

Every line needs a real `"timestamp"` (`HH:mm:ss`, e.g. via
`TZ='America/Chicago' date +'%H:%M:%S'` — never guessed) alongside `"date"`.

Write:
- One `"stage": "screened"` line per candidate (`passed_filters`, `source`
  (`"watchlist"` or `"market_scan"`), `avg_volume`, `market_cap`, `reason`
  if rejected, plus `"signal_check"` noting which Step 2 threshold(s)
  triggered with **raw inputs** for each ratio):
  - `"price_move_60d: 0.2925 (close_60d_ago: 424.10 -> latest_close: 548.13)"`
  - `"volume_spike: 2.3x (latest_volume: 68000000 / avg_volume_30d: 29421634)"`
  - `"near_52wk_high: 0.02 (current_price: 314.86 / highest_52weeks_price: 321.00)"`
  - `"none"` if it didn't qualify.
- One `"stage": "thesis"` line per flagged candidate.

Do not touch `trade_log.jsonl`.

**After all `screened`/`thesis` lines, append one `"stage": "summary"` line
per decision bucket** (always all five, in order, even if empty):
```json
{"date": "YYYY-MM-DD", "timestamp": "HH:mm:ss", "stage": "summary", "decision": "rejected", "symbols": ["AMC", "ADDYY"]}
{"date": "YYYY-MM-DD", "timestamp": "HH:mm:ss", "stage": "summary", "decision": "no_signal", "symbols": ["US.TSLA", "US.NVDA"]}
{"date": "YYYY-MM-DD", "timestamp": "HH:mm:ss", "stage": "summary", "decision": "avoid", "symbols": ["US.SPCX", "US.LCID"]}
{"date": "YYYY-MM-DD", "timestamp": "HH:mm:ss", "stage": "summary", "decision": "long", "symbols": ["US.AAPL (medium)", "US.AMD (high)"]}
{"date": "YYYY-MM-DD", "timestamp": "HH:mm:ss", "stage": "summary", "decision": "exit_existing", "symbols": []}
```
`long` appends each symbol's `conviction` as `"<symbol> (<conviction>)"`.

## Hard stop

Do not call `place_order`, `cancel_order`, `modify_order`, or any trading tool,
or check `execution.mode`. `get_positions` is for Step 1's candidate list only —
no stop-loss/drawdown computation here; all risk enforcement is Phase B's job.

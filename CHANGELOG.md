# Portfolio Analyzer — Changelog

---

## v3.4 — 2026-05-02 — API efficiency + data accuracy fixes

### Fixed

**Alpha Vantage over-triggering**
`DividendYield=None` and `TrailingPE=None` were used as standalone triggers to call Alpha Vantage.
The flaw: `yfinance` returns `None` (not `0`) for non-dividend stocks and loss-making stocks — those
are expected absences, not missing data, and AV would also return `None` for them. This caused every
non-dividend stock (AMZN, GOOGL, META, GEV, AUR, …) to fire an AV call.

With `AV_MIN_GAP = 13s` and ~6 tickers triggering AV, the last worker waited 65+ seconds for the
lock — the progress bar appeared frozen. Multiple analysis runs exhausted AV's 25 calls/day free tier,
making subsequent runs show "AV key error" / LOW data quality on most stocks.

Fix: AV now only fires when `_yf_coverage < 3` (fewer than 3 of 5 key scoring fields populated by
yfinance). Reduces AV calls from ~5–6 per run to 0–2 for typical portfolios.

---

**DividendYield and ytdReturn double-division**
`yfinance 0.2+` returns `dividendYield` and `ytdReturn` as decimals (e.g. `0.107` for 10.7%).
The code was dividing by 100 again, storing `0.00107`, then multiplying back by 100 for display —
showing `0.11%` instead of `11%` for JEPQ, and ETF YTD returns 100× too small.

Fix: Removed the `/100` division from both fields in `yf_get_all()`. Updated the comment from
"percentages (e.g. 11.1 = 11.1%)" to "decimals (0.11 = 11%)".

---

**Finnhub always called for every ticker**
The secondary Finnhub call for a "fresher real-time price" ran unconditionally inside the
`else` branch (yfinance succeeded). This meant every ticker made a guaranteed Finnhub call even
when yfinance already had a valid real-time price and `change_pct`.

Fix: Wrapped in `if change_pct is None and fh_key:` — Finnhub is now only called when yfinance
returned a stale/cached price with no intraday change data.

---

**Duplicate tickers in portfolio**
The ticker parser used a plain list comprehension with no deduplication. A ticker appearing twice
in the URL (e.g. `BLK` at positions 18 and 20) resulted in two full fetch cycles, two cards in
the results, and a race condition in `cached_fetch` where both threads could miss the cache
simultaneously and make duplicate API calls.

Fix: `dict.fromkeys(...)` deduplicates while preserving insertion order.

---

**Cache wiped on every Analyze click**
`clear_cache()` was called unconditionally before every analysis run, making the 30-minute TTL
entirely useless. The footer caption "Scores cached 30 min" was misleading.

Fix: `clear_cache()` now only runs when the ticker set has actually changed. Re-running Analyze
on the same tickers reuses the cache, making re-analysis instant within the 30-minute window.

---

### Added

**VOOG to ETF_DB**
`VOOG` (Vanguard S&P 500 Growth ETF) was not in the built-in ETF database. It was detected as an
ETF via yfinance's `quoteType`, but fell back to `{"expense": None, "num_holdings": 0}`, scoring
Diversification 2/10 (worst bucket) despite holding ~231 stocks. Data quality was forced to LOW.

Added: `"VOOG": {"expense": 0.10, "num_holdings": 231, "category": "S&P 500 Growth", "multi_class_aum": True, "beta_fallback": 1.10}`

---

### Changed

**max_workers: 8 → 4**
Reduced `ThreadPoolExecutor` workers from 8 to 4. With 8 workers, all 8 threads slept 0.15s
then fired at Yahoo Finance simultaneously — a burst of 8 concurrent connections that triggered
throttling and empty responses on some tickers. 4 workers halves the burst load while keeping
analysis parallel.

---

## v3.3 — 2026-04-28 — Analyze on demand only + cross-platform save

- Fixed: Analysis no longer runs automatically on every Streamlit rerun
- Added: "Click Analyze" placeholder on first load
- Deploy: Push to GitHub + connect at share.streamlit.io

## v3.2 — 2026-04-28 — Named portfolios + data accuracy

- Added: Multiple named portfolios saved in URL `?saved=` param
- Added: Portfolio switcher dropdown
- Added: Delete button for named portfolios
- Added: Auto-sync URL on Analyze
- Added: Data accuracy % per ticker

## v3.1 — 2026-04-28 — Performance & UX improvements

- Changed: Moved `_TIER_OVERRIDES`, `_TECH`, `_IND` tuples to module level
- Added: "Last analyzed" datetime stamp
- Added: Live ticker count in Portfolio Settings expander label

## v3.0 — 2026-04-28 — Initial public release

- yfinance-first data architecture with Finnhub + Alpha Vantage supplements
- Parallel fetch via ThreadPoolExecutor
- Sector-aware valuation scoring (Tech / Industrial / Default tiers)
- ETF scoring with built-in expense/holdings database
- Named portfolio save/restore via URL params

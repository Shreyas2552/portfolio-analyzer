# CLAUDE.md — Portfolio Analyzer

Codebase reference for AI-assisted development sessions.

---

## What This Repo Is

A **single-file** Streamlit app (`portfolio_streamlit.py`) that scores a user-defined
portfolio of stocks and ETFs using live fundamental data from yfinance, Finnhub, and
Alpha Vantage. Deployed at https://portfolio-analyzer-shreyas.streamlit.app/

- **GitHub repo:** https://github.com/Shreyas2552/portfolio-analyzer
- **Entry point Streamlit watches:** `portfolio_streamlit.py`
- **Branch:** `main` — auto-redeploys on every push (~1–2 min)

---

## ⚠️ Scope Boundary — READ BEFORE ADDING FEATURES

This repo contains **only** the Portfolio Analyzer. It is a **single-file app** with no
`pages/` folder and no `modules/` folder.

**The Control Systems Simulator (PID, LQR/LQG, Bode plots, Kalman filter) is a completely
separate app in its own repo:**

| | Portfolio Analyzer | Control Systems Simulator |
|---|---|---|
| Repo | `Shreyas2552/portfolio-analyzer` | `Shreyas2552/controls-simulator` |
| Entry point | `portfolio_streamlit.py` | `app.py` |
| Dependencies | streamlit, requests, yfinance, pandas | streamlit, numpy, scipy, plotly |
| Has `pages/` | NO | YES (`1_Control_Simulator.py`, `2_LQR_LQG.py`) |
| Has `modules/` | NO | YES (plants, PID, analysis, filters, lqr_lqg) |

**Why they are separate:** On 2026-04-30 the simulator was merged here as a multi-page app
(commit `79747e6`) and reverted the same day (commit `12473eb`). The merge caused:
- numpy/scipy/plotly added to a finance app's requirements (unnecessary 200MB+ install)
- A single Streamlit deployment serving two completely unrelated tools
- Scope confusion in future development sessions

**Do not merge them again.** If asked to add control-theory features to this repo, decline
and point to `Shreyas2552/controls-simulator` instead.

---

## File Structure

```
portfolio_streamlit.py   Single-file Streamlit app — all logic in one place
requirements.txt         streamlit, requests, yfinance, pandas only
CHANGELOG.md             Version history with scope boundary note at top
CLAUDE.md                This file
portfolio_analyzer_5.py  Previous version kept for reference — not deployed
```

No `pages/`, no `modules/`, no subfolders are part of this app.

---

## Data Architecture

```
PRIMARY    → yfinance       price, fundamentals, analyst targets, ETF data (free)
SECONDARY  → Finnhub        real-time price — only called when yfinance lacks change_pct
SUPPLEMENT → Alpha Vantage  only when yfinance_coverage < 3 of 5 key fields (rarely fires)
PARALLEL   → ThreadPoolExecutor(max_workers=4) — all tickers fetched simultaneously
CACHE      → module-level dict, 30-min TTL, cleared only when ticker set changes
```

### Alpha Vantage rate limits
- Free tier: 5 req/min, **25 req/day**
- `AV_MIN_GAP = 13.0s` serialises calls through `_av_lock`
- AV only fires when `_yf_coverage < 3` — do NOT add `DividendYield is None` or
  `TrailingPE is None` as standalone triggers (they fire for every non-dividend /
  loss-making stock; see v3.4 fix notes in CHANGELOG.md)

### Finnhub rate limits
- Free tier: 60 req/min
- `FH_MIN_GAP = 0.25s` serialises calls through `_fh_lock`
- Only called as fallback when yfinance fails for price, OR when `change_pct is None`

---

## Key Functions

| Function | Purpose |
|---|---|
| `yf_get_all(ticker)` | Primary fetch — one `.info` call returns everything |
| `fetch_and_score(ticker, fh_key, av_key)` | Full pipeline: fetch → score → return result dict |
| `cached_fetch(ticker, fh_key, av_key)` | Wraps fetch with 30-min module-level cache |
| `score_stock(...)` | 12-criteria scoring for equities; sector-aware P/E thresholds |
| `score_etf(...)` | 8-criteria scoring for ETFs; uses ETF_DB for expense/holdings |
| `_field_accuracy(criteria)` | % of criteria with real (non-N/A) data |

---

## ETF_DB

Built-in dict at module level. If a ticker is not in ETF_DB but yfinance identifies it as
an ETF (`quoteType == "ETF"`), it falls back to `{expense: None, num_holdings: 0}` which
scores Diversification 2/10 and triggers a data quality warning.

**Add missing ETFs to ETF_DB** — do not rely on the fallback for known ETFs.
Currently missing entries would score incorrectly (e.g. VOOG was missing until v3.4).

---

## Sector Tier System

`score_stock()` uses three scoring tiers with different P/E and gross margin thresholds:

| Tier | P/E good / bad | Gross margin good / bad | Typical tickers |
|---|---|---|---|
| Tech | 18 / 60 | 65% / 25% | NVDA, AMZN, GOOGL, META, MSFT, TSLA |
| Industrial | 10 / 28 | 35% / 5% | GEV, PCAR, CAT, F, BA, LMT |
| Default | 13 / 38 | 55% / 15% | Everything else |

Override a ticker's tier in `_TIER_OVERRIDES` dict (module level) when yfinance's sector
label doesn't match its actual business model (e.g. AMZN is "Consumer Cyclical" but scores
as Tech).

---

## Deployment

```bash
# Local dev
streamlit run portfolio_streamlit.py

# Deploy (Streamlit Cloud watches main branch)
git add portfolio_streamlit.py
git commit -m "..."
git push origin main   # auto-redeploys in ~1-2 min
```

Secrets (Finnhub key, AV key) go in Streamlit Cloud dashboard under App Settings → Secrets:
```toml
FINNHUB_KEY = "your_key_here"
AV_KEY      = "your_key_here"
```
The app falls back to hardcoded demo keys if secrets are not set, but those keys are shared
and will hit rate limits under any real load. Set proper secrets for production use.

---

## Known Gotchas

**yfinance dividendYield is a decimal, not a percentage**
`info["dividendYield"]` returns `0.107` for 10.7% in yfinance 0.2+. Do NOT divide by 100
before storing — the display code multiplies by 100. See v3.4 fix in CHANGELOG.md.

**AV trigger must stay as `_yf_coverage < 3` only**
Do not add `DividendYield is None` or `TrailingPE is None` back as triggers. Both return
None legitimately for non-dividend and loss-making stocks. See v3.4 for the full explanation.

**Finnhub secondary call must stay conditional**
`fh_get()` inside the `else` branch must stay wrapped in `if change_pct is None and fh_key:`.
Removing that condition fires 22 Finnhub calls per analysis run for no benefit.

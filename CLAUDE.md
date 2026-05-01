# Codebase Reference — Portfolio Analyzer + Control Simulator

## What This Is

A single Streamlit multi-page app combining two tools:
1. **Portfolio Analyzer** — real-time stock fundamentals via yfinance / Finnhub / Alpha Vantage
2. **Control Systems Simulator** — interactive PID, LQR, Kalman Filter, LQG educational tool

Deployed on Streamlit Community Cloud — no local setup needed for end users.

- **Live URL:** https://portfolio-analyzer-shreyas.streamlit.app/
- **GitHub repo:** https://github.com/Shreyas2552/portfolio-analyzer
- **Entry point Streamlit watches:** `portfolio_streamlit.py`
- **Auto-redeploys** on every push to `main` (~1–2 min)

---

## File Structure

```
portfolio_streamlit.py          Home page — Portfolio Analyzer (unchanged from original)
portfolio_analyzer_5.py         Old version, kept for reference, not deployed as a page

pages/
  1_Control_Simulator.py        PID simulator — the full control_simulator/app.py
  2_LQR_LQG.py                  LQR + Kalman + LQG — from control_simulator/pages/2_LQR_LQG.py

modules/                        Shared computation layer (no UI code here)
  __init__.py                   Package marker
  plants.py                     6 transfer-function plant models + PLANT_MODELS catalogue
  pid_controller.py             PID TF with derivative filter; Tustin (bilinear) discretisation
  analysis.py                   TF algebra, step/ramp/control responses, Bode, root locus,
                                 stability margins, pole analysis, performance metrics
  filters.py                    Analog + digital filter design (Butterworth, Chebyshev, Bessel)
  lqr_lqg.py                    State-space plants, LQR/Kalman Riccati solvers, Euler simulations

requirements.txt                All dependencies for both tools (merged)
.gitignore                      Excludes __pycache__, *.pyc, .streamlit/secrets.toml
```

### How Streamlit multi-page works
- Streamlit auto-discovers everything in `pages/` when you run the main file.
- File naming: `N_Display_Name.py` — the number sets sidebar order, underscores become spaces.
- Each page runs as an independent script; each can call `st.set_page_config()` (must be the very first `st.*` call in that file).
- `modules/` imports work because Streamlit always runs from the **repo root**, so `from modules.X import Y` resolves correctly on both local and Streamlit Cloud.

---

## Module Details

### `modules/plants.py`
- `PLANT_MODELS` dict — keys are display names, values have `tf_display`, `physical_context`, `params` (with default/min/max/step for sliders).
- `get_plant_tf(plant_name, params) -> (num, den)` — returns coefficient lists, highest power first.
- 6 models: First Order, Second Order, DC Motor (Position), Integrating Plant, Unstable Plant, Third Order.

### `modules/pid_controller.py`
- `get_pid_tf(Kp, Ki, Kd, N) -> (num, den)` — continuous PID with first-order derivative filter `C(s) = Kp + Ki/s + Kd·N·s/(s+N)`. Handles P/PI/PD/PID as special cases.
- `discretize_tf(num, den, Ts, method='tustin') -> (num_d, den_d)` — wraps `scipy.signal.cont2discrete`.

### `modules/analysis.py`
- `build_ol_cl(plant_num, plant_den, ctrl_num, ctrl_den)` — returns `(ol_num, ol_den, cl_num, cl_den)`.
- `step_response`, `ramp_response`, `control_signal_step` — time-domain simulations.
- `bode_data(num, den, n, discrete, Ts)` — auto-ranges frequency axis from pole/zero locations.
- `stability_margins(ol_num, ol_den, discrete, Ts)` — returns `{gm_db, pm_deg, wgc, wpc}` via zero-crossing on unwrapped Bode phase.
- `root_locus_data(ol_num, ol_den, n_gains)` — K-sweep with nearest-neighbour branch assignment for smooth traces.
- `cl_pole_analysis(cl_den)` — classifies poles as real/complex-pair; computes ζ, ωn, ωd, settling time, overshoot.
- `performance_metrics(t, y, ref)` — overshoot, rise time (10→90%), settling time (2%), SS error.

### `modules/filters.py`
- `design_analog(ftype, family, order, wc, wc2, ripple_db)` — wraps scipy Butterworth/Chebyshev/Bessel.
- `design_digital(ftype, family, order, wn, wn2, ripple_db)` — normalised frequency [0,1] where 1=Nyquist.
- `filter_bode(b, a, analog)` — returns `(omega, mag_dB, phase_deg)`.
- `filter_step(b, a, analog, t_end)` — step response using `scipy.signal.step` (analog) or `lfilter` (digital).

### `modules/lqr_lqg.py`
- `SS_PLANTS` dict — 4 state-space plants: Mass-Spring-Damper, DC Motor, Inverted Pendulum on Cart, Double Integrator.
- `get_ss(plant_name, params) -> (A, B, C, D)` — builds numpy matrices from physical parameters.
- `controllability_rank(A, B)`, `observability_rank(A, C)` — rank checks.
- `lqr_design(A, B, Q, R) -> (K, P, cl_eigs)` — solves continuous-time ARE via `scipy.linalg.solve_continuous_are`.
- `kalman_design(A, C, Qn, Rn) -> (L, Pe)` — dual of LQR; observer gain via observer ARE.
- `compute_nbar(A, B, C_row, K) -> float` — DC pre-compensator so steady-state output = reference.
- `simulate_lqr(A, B, C_out, K, Nbar, ref, t_end, ref_state)` — Euler integration at 200 Hz, no noise.
- `simulate_lqg(A, B, C_out, C_obs, K, L, Nbar, ref, t_end, q_std, r_std, ref_state)` — full LQG with process + measurement noise.

---

## Known Gotchas (do not repeat this analysis)

### 1. NumPy 2.x — `float()` on arrays
**Python 3.14 + NumPy 2.4 is in use.** In NumPy 2.0+, `float(arr)` raises `TypeError` if `arr` is 1-dimensional, even with a single element.

```python
# WRONG (breaks in NumPy 2.x)
u_k = float(-K @ x)   # K is (1,n), result is shape (1,) → TypeError

# CORRECT
u_k = (-K @ x).item()
```

Already fixed in `lqr_lqg.py` at lines 231, 237 (`simulate_lqr`) and line 272 (`simulate_lqg`). If you add new simulation code, always use `.item()` or `.flat[0]` to extract scalars from array results.

### 2. `pip` is not in PATH
On this machine only `python` is in PATH, not `pip.exe`. Scripts must use:
```
python -m pip install -r requirements.txt   # correct
pip install -r requirements.txt             # fails silently in .bat
```
`run_simulator.bat` in the standalone `control_simulator/` folder already uses `python -m pip`.

### 3. Streamlit multi-page — `set_page_config` placement
Each page file must call `st.set_page_config()` **before any other `st.*` call**. Both control simulator pages already do this. Don't add `st.write()` or `st.sidebar.*` above it.

### 4. Standalone vs merged
The original standalone simulator lives at `C:\Users\accor\control_simulator\`. It still works on its own via `run_simulator.bat` or `streamlit run app.py`. The `portfolio-analyzer-merge\` clone is the version pushed to GitHub/Streamlit Cloud.

---

## Dependencies

| Package | Version req | Used by |
|---------|------------|---------|
| streamlit | ≥1.35.0 | All pages |
| pandas | ≥2.0.0 | Both tools |
| requests | ≥2.31.0 | Portfolio Analyzer (API calls) |
| yfinance | ≥0.2.40 | Portfolio Analyzer (stock data) |
| numpy | ≥1.24.0 | Control Simulator (all modules) |
| scipy | ≥1.10.0 | Control Simulator (signal, linalg) |
| plotly | ≥5.15.0 | Control Simulator (all charts) |

Portfolio Analyzer uses Plotly implicitly through its own chart code. If you ever add chart types to the portfolio page, plotly is already available.

---

## How to Run Locally

```bash
# From the repo root (portfolio-analyzer-merge/)
streamlit run portfolio_streamlit.py

# Or the standalone simulator only (control_simulator/)
cd C:\Users\accor\control_simulator
streamlit run app.py          # or double-click run_simulator.bat
```

Browser opens at `http://localhost:8501`. The sidebar shows all three pages.

---

## Adding a New Page

1. Create `pages/N_Your_Page_Name.py` (pick N to set sidebar order).
2. Start the file with `st.set_page_config(...)`.
3. Import from `modules/` as needed — path is always relative to repo root.
4. Add any new pip packages to `requirements.txt`.
5. Commit + push to `main` → Streamlit Cloud auto-redeploys.

No changes needed to `portfolio_streamlit.py` or any existing file.

---

## Adding a New Plant Model

**Transfer-function plant** (PID simulator):
- Add an entry to `PLANT_MODELS` in `modules/plants.py`.
- Add the `if plant_name == "Your Plant":` branch in `get_plant_tf()`.
- Nothing else needs changing — the sidebar loop and tab code are data-driven.

**State-space plant** (LQR/LQG simulator):
- Add an entry to `SS_PLANTS` in `modules/lqr_lqg.py`.
- Add the `if plant_name == "Your Plant":` branch in `get_ss()`.
- Provide `default_Q` (list, one per state), `default_R` (float), `ref_state` (index).

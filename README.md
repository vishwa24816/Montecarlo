# Monte Carlo GBM Stock Simulation

An educational Jupyter notebook that simulates thousands of possible future price paths for a stock using Geometric Brownian Motion (GBM), then summarizes the distribution of outcomes: typical results, bad cases, chance of loss, and drawdown risk.

> **Disclaimer.** This project is an educational model of randomness in asset prices. It is not investment advice, and it is not a price forecast. GBM assumes that past return statistics persist, which real markets do not guarantee.

---

## Contents

- [What it does](#what-it-does)
- [Repository layout](#repository-layout)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Notebook walkthrough](#notebook-walkthrough)
- [Inputs reference](#inputs-reference)
- [Outputs](#outputs)
- [How the model works](#how-the-model-works)
- [Limitations](#limitations)
- [Testing and sanity checks](#testing-and-sanity-checks)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## What it does

Given a ticker, the notebook:

1. Downloads adjusted daily closing prices for a lookback window (default 10 years) using `yfinance`.
2. Estimates the annualized arithmetic mean return (`mu`) and volatility (`sigma`) from historical log returns.
3. Applies the volatility-drag correction, so the median path grows at `mu - sigma²/2`.
4. Simulates many price paths over a chosen horizon (default 30 years) with a fixed random seed.
5. Reports terminal-value statistics: median, mean, percentiles, standard deviation, expected shortfall, probability of loss, and implied CAGR.
6. Reports maximum drawdown statistics for each path.
7. Plots a fan chart, terminal and drawdown histograms, and the probability of being above the starting capital over time.
8. Runs diagnostics comparing historical returns with the normal assumption (QQ plot, Jarque-Bera test).
9. Optionally compares several tickers side by side and runs a sensitivity grid over `mu` and `sigma`.
10. Writes a plain-language summary generated from the computed numbers.

---

## Repository layout

```
.
├── monte_carlo_gbm_stock.ipynb   # the notebook (all sections, in order)
├── README.md                     # this file
├── requirements.txt              # pinned-by-range dependencies
└── LICENSE
```

The notebook is self-contained. There are no helper modules, so everything runs from the notebook alone.

---

## Requirements

- Python 3.10 or newer
- Jupyter Notebook, JupyterLab, or VS Code with the Jupyter extension
- Internet access for the first run, to download prices from Yahoo Finance (the notebook falls back to clearly labeled synthetic data if the download fails)

Python packages:

| Package | Purpose |
|---|---|
| `numpy` | Vectorized simulation |
| `pandas` | Tabular summaries |
| `matplotlib` | All figures |
| `scipy` | Jarque-Bera test, QQ plot, skewness and kurtosis |
| `yfinance` | Historical adjusted closing prices |
| `ipython` (optional) | Rich Markdown display; the notebook falls back to plain text without it |

`requirements.txt`:

```
numpy>=1.24
pandas>=2.0
matplotlib>=3.7
scipy>=1.10
yfinance>=0.2.40
ipython>=8.0
```

---

## Quick start

1. Clone the repository and move into it:

   ```bash
   git clone <your-repo-url> monte-carlo-gbm
   cd monte-carlo-gbm
   ```

2. Create an environment and install dependencies:

   ```bash
   python -m venv .venv
   source .venv/bin/activate          # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. Launch Jupyter and open the notebook:

   ```bash
   jupyter lab monte_carlo_gbm_stock.ipynb
   ```

4. Edit only the input cell in Section 1 if you want different settings, then run all cells from top to bottom (**Kernel → Restart & Run All**).

The notebook is designed to run top to bottom in a fresh kernel with no manual edits beyond the input cell.

---

## Notebook walkthrough

| Section | Cell content | Purpose |
|---|---|---|
| 1 | Inputs | All editable parameters in one place |
| 2 | Imports and validation | Checks inputs before any expensive work |
| 3 | Data and calibration | Downloads prices, estimates `mu` and `sigma`, applies the drift correction |
| 4 | Random shocks | Draws the shock matrix (normal or Student-t), with chunking for large runs |
| 5 | Path accumulation | Builds price paths from cumulative log returns |
| 6 | Run simulation | Executes the simulation and reports runtime |
| 7 | Terminal statistics | Summary table of ending values and drawdowns |
| 8 | Visualizations | Fan chart, terminal histogram, drawdown histogram, probability above start |
| 9 | Model diagnostics | QQ plot, Jarque-Bera test, excess kurtosis |
| 10 | Multi-ticker comparison | Side-by-side table and median-path chart (when `ticker` is a list) |
| 11 | Sensitivity analysis | Heatmaps of median outcome and chance of loss over `mu` and `sigma` |
| 12 | Plain-language summary | Generated text from the computed numbers |
| 13 | Disclaimer | Educational-use statement |
| 14 | Sanity checks | Assertions on shape, positivity, and starting values |

---

## Inputs reference

All inputs are set in the Section 1 cell.

| Input | Default | Description |
|---|---|---|
| `ticker` | `"AAPL"` | Ticker symbol, or a list such as `["AAPL", "MSFT", "SPY"]` for comparison mode |
| `start_capital` | `10_000` | Starting investment in USD |
| `years` | `30` | Simulation horizon in years (must be at least 1) |
| `steps_per_year` | `252` | Time steps per year. `252` for trading days, `1` for annual steps |
| `n_paths` | `10_000` | Number of simulated paths (must be at least 1,000) |
| `history_years` | `10` | Lookback window for estimating `mu` and `sigma` |
| `mu_override` | `None` | Annualized arithmetic mean return, e.g. `0.08`. Replaces the historical estimate |
| `sigma_override` | `None` | Annualized volatility, e.g. `0.20`. Must be non-negative. Replaces the historical estimate |
| `shock_model` | `"normal"` | `"normal"` or `"student_t"` |
| `df` | `5` | Degrees of freedom for the Student-t model (must be greater than 2) |
| `risk_free_rate` | `0.04` | Annual rate, used only for Sharpe-style comparison |
| `seed` | `42` | Random seed for reproducibility |
| `loss_threshold` | `start_capital` | Terminal value below this counts as a loss |
| `percentiles` | `[5, 10, 25, 50, 75, 90, 95]` | Percentiles reported in the statistics table |

Notes on specific inputs:

- **`mu_override` and `sigma_override`** are the most important inputs for honest results. Ten years of history gives a noisy drift estimate. The standard error of an annual mean is roughly `sigma / sqrt(history_years)`, which is several percentage points for a typical stock.
- **`steps_per_year = 1`** gives annual steps. It runs much faster and is a useful cross-check against the daily model.
- **`loss_threshold`** defaults to `start_capital`, so it follows changes to the capital. Set it directly to override.

---

## Outputs

Running the notebook produces:

- **Calibration printout**: the data source (real or synthetic), number of observations, `mu`, `sigma`, and the drift per step.
- **Terminal statistics table**: median, mean, mean-to-median ratio, standard deviation, coefficient of variation, chosen percentiles, chance of loss, expected shortfall (mean of the worst 5%), drawdown median and 5th/95th percentiles, and implied CAGR of the median path.
- **Four figures**:
  - Fan chart of paths over time (log scale), with 5–95 and 25–75 percentile bands and a sample of individual paths.
  - Histogram of terminal values on a log x-axis, with markers for median, mean, 10th percentile, and loss threshold.
  - Histogram of maximum drawdowns.
  - Fraction of paths above the starting capital over time.
- **Diagnostics**: skewness, excess kurtosis, Jarque-Bera statistic and p-value, and a QQ plot of historical returns.
- **Comparison table and chart** (multi-ticker mode).
- **Sensitivity heatmaps** (median terminal value and chance of loss).
- **Summary paragraph** generated from the numbers.

Example output at the default settings, using the synthetic fallback data (real AAPL results will differ):

```
Median terminal value        $3,244
10th percentile                $814
P(terminal < $10,000)        85.0%
Max drawdown, median         84.2%
```

Because the model is random, the exact numbers depend on the seed, the calibration window, and the inputs. Changing `seed` produces different paths with similar statistics; changing `mu_override` or `sigma_override` changes the statistics themselves.

---

## How the model works

### Geometric Brownian Motion

The price follows

```
dS_t = mu * S_t * dt + sigma * S_t * dW_t
```

Discretized over steps of length `dt = 1 / steps_per_year`, the log return per step is normal:

```
r_t ~ Normal( (mu - sigma²/2) * dt , sigma² * dt )
```

The notebook draws these shocks directly, sums them over time with `cumsum`, and exponentiates:

```
P_t = P_0 * exp( sum of r_1 ... r_t )
```

### Calibration

From daily log returns `r_t = ln(P_t / P_{t-1})`:

- `mu_arith = (mean(r) + var(r)/2) * steps_per_year`. This converts the log-return mean to an arithmetic mean.
- `sigma = std(r, ddof=1) * sqrt(steps_per_year)`.
- `growth = mu_arith - sigma²/2`. This is the volatility-drag correction. The median path grows at this rate, not at the arithmetic mean.

### Drawdown

For each path, the maximum drawdown is the largest fall from any running peak:

```
max over t of ( 1 - P_t / max(P_0 ... P_t) )
```

### Memory handling

The full path matrix has `n_paths × (years × steps_per_year + 1)` elements. When that exceeds about 500 million elements, the notebook switches to chunked mode. It simulates blocks of paths, accumulates only the statistics it needs, and reuses the same seeded chunks for later sections. In chunked mode, the fan chart and the median-path comparison chart are skipped, because they need percentiles across the full matrix at each time point.

### Sanity checks

The final section asserts that the path matrix has the expected shape, every value is positive and finite, the first column equals `start_capital`, and the drawdowns lie between 0 and 1.

---

## Limitations

This model is useful for understanding randomness and the effect of volatility and drift. It has real limitations, and results should be read with them in mind.

- **Constant parameters.** GBM assumes `mu` and `sigma` do not change. Real volatility varies over time and rises sharply during crises.
- **Thin tails.** Normal shocks understate extreme losses. The Student-t option adds fat tails but does not fix the problem fully.
- **No volatility clustering.** Real returns cluster: large moves tend to follow large moves. GBM treats each step as independent.
- **Drift is hard to estimate.** Ten years of history gives a noisy `mu`. A few percentage points of error in `mu` changes the long-horizon outcome a great deal. Use the sensitivity section, and consider `mu_override`.
- **Survivorship and selection.** The ticker you choose has survived to today. Past winners are not a guide to future returns.
- **No dividends beyond adjusted prices, taxes, fees, or inflation.** Adjusted closes include dividend reinvestment, but the model does not account for taxes, trading costs, or the purchasing power of money over time.
- **Single-asset model.** Portfolios, correlations, and rebalancing are not modeled.
- **Synthetic fallback.** If the download fails, the notebook uses a generated series. Results from that run do not describe any real stock.

---

## Testing and sanity checks

The notebook includes assertions in its final section. They run automatically and stop the notebook if a check fails:

- Path matrix shape is `(sample_size, years × steps_per_year + 1)`.
- All path values are positive and finite.
- The first column equals `start_capital`.
- Drawdowns lie in `[0, 1]`.

To run the notebook headlessly as a check:

```bash
pip install nbclient nbformat ipykernel
jupyter nbconvert --to notebook --execute monte_carlo_gbm_stock.ipynb --output /tmp/check.ipynb
```

A successful run exits with status 0.

---

## Troubleshooting

**"Using SYNTHETIC demo data" appears in the output.**
The Yahoo Finance download failed. Check your internet connection, confirm the ticker is valid, and retry. Results from synthetic data are not real.

**"Only N observations (fewer than 252)" warning.**
The ticker has less history than expected, often a recent listing. Reduce `history_years` or accept that the estimates are unreliable.

**`ValueError` on startup.**
An input failed validation. The message names the parameter. Common causes: `n_paths` below 1,000, `years` below 1, a negative `sigma_override`, or `df` of 2 or less.

**Run is slow or the kernel runs out of memory.**
Reduce `n_paths` or `years`, or set `steps_per_year = 1`. The chunked mode engages automatically for large runs, but very large inputs will still take time.

**Fan chart missing.**
This happens in chunked mode, which is expected for large runs. The fan chart needs the full path matrix.

**`ModuleNotFoundError: yfinance`.**
Run `pip install yfinance`, or reinstall from `requirements.txt`.

**Yahoo Finance rate limits.**
Wait a few minutes and rerun. The notebook downloads one ticker at a time in comparison mode, which keeps requests low.

---

## License

Add your chosen license here (for example MIT). See `LICENSE`.

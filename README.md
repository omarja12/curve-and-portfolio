# curve-and-portfolio

Yield curve construction and mean-variance portfolio analysis in Python.

Two halves that meet in the middle: bootstrap a discount-factor curve from
coupon-bearing bonds, then use the short end of that curve as the risk-free rate for a
Markowitz optimization over an equity panel.

The bootstrapping, the forward-rate derivation and the Nelson-Siegel fit are implemented
directly rather than called out to a curve library.

---

## What it does

### `YieldCurve`

Takes an `(n, 3)` array of `(maturity_years, dirty_price, coupon_rate)` and bootstraps
discount factors three independent ways:

| Method | How |
|--------|-----|
| `'Matrix operations'` | Builds the cash-flow matrix and solves the linear system directly |
| `'Global Solver'` | Minimizes squared error between model and market prices |
| `'Iterative Procedure'` | Forward substitution, the way a curve is built by hand |

They agree to `3.3e-16` — within one bit of float64 machine epsilon. From the discount
factors it derives spot rates, yield to maturity per bond, and 1-year forward rates.

### `NelsonSiegel`

The parametric curve used by central banks and dealers:

```
f(t) = β₀ + β₁·e^(−t/τ) + β₂·(t/τ)·e^(−t/τ)
```

Fitted by least squares to the bootstrapped discount factors, smoothing the jagged curve
that ten bonds alone produce.

### `PortfolioAnalyzer`

Daily and annualized returns, volatility, Sharpe ratio, correlation matrix, the Markowitz
efficient frontier, and the maximum-Sharpe tangency portfolio. Prices come from yfinance
or from any DataFrame you supply.

---

## Quick start

```bash
git clone https://github.com/omarja12/curve-and-portfolio.git
cd curve-and-portfolio

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -e ".[dev,yfinance]"
pytest tests/ -v

# full analysis, headless — writes five plots to docs/images/
python notebooks/run_analysis.py
```

```python
import numpy as np
from yield_curve import YieldCurve, NelsonSiegel

bonds = np.array([
    [1, 96.60, 0.0150],
    [2, 93.71, 0.0175],
    [3, 91.56, 0.0200],
    # ... out to 10 years
], dtype=float)

yc = YieldCurve(bonds)

df = yc.discount_factors("Matrix operations")
spot = yc.spot_rates()
fwd = yc.forward_rates()

ns = NelsonSiegel(bonds[:, 0].astype(float), df)
ns.fit()
print(ns.params)          # β₀, β₁, β₂, τ
```

---

## Layout

```
curve-and-portfolio/
├── src/yield_curve/
│   ├── bootstrap.py     # YieldCurve, NelsonSiegel
│   └── portfolio.py     # PortfolioAnalyzer, yfinance helper
├── tests/
│   └── test_bootstrap.py
├── notebooks/
│   ├── Yield_Curve_And_Portfolio_Analysis.ipynb
│   └── run_analysis.py
└── docs/images/         # figures written by run_analysis.py
```

## Requirements

`numpy` ≥ 1.20 · `scipy` ≥ 1.7 · `pandas` ≥ 1.3 · `matplotlib` ≥ 3.4 · `seaborn` ≥ 0.11 ·
`numpy-financial` ≥ 0.2. Optional: `yfinance` for live prices.

Tested on Python 3.9–3.12.

## A note on the portfolio data

The portfolio section fetches prices from yfinance. If that fetch fails, it falls back to
a simulated panel — that path uses neutral asset names (`ASSET_A`…`ASSET_E`) and marks
every chart `(synthetic sample data)`, so simulated output is never presented as real
market data.

## License

All rights reserved — see [LICENSE](LICENSE). Published for reading and review; any other
use requires written permission.

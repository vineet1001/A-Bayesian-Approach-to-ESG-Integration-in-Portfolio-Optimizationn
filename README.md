# A Bayesian Approach to ESG Integration in Portfolio Optimization

Black-Litterman portfolio construction that turns ESG risk scores into investment **views** instead of exclusion screens, implemented in MATLAB on 15 S&P 500 stocks.

**Course:** Bayesian Methods in Finance (ECO528), IIT Kanpur, taught by Prof. Mohammad Arshad Rahman
**Team:** Sonali Kumari, Rajat Kumar, Vineet Nagrale
**Term paper:** [`TermPaper.pdf`](TermPaper.pdf)

---

## Results at a glance

| Strategy | Ann. return | Ann. volatility | Sharpe | Max drawdown | Final value (from $100,000) |
|---|---:|---:|---:|---:|---:|
| Market (cap-weighted) | 21.63% | 10.77% | 2.01 | 8.05% | $121,533.93 |
| Markowitz MVO (long-only) | 47.15% | 15.29% | 3.08 | 8.25% | $146,923.58 |
| **Black-Litterman ESG (long-only)** | **26.28%** | **11.40%** | **2.31** | **8.15%** | **$126,163.48** |

- **Black-Litterman ESG improves the Sharpe ratio by 15% over the market benchmark** (2.31 vs 2.01), with almost the same drawdown (8.15% vs 8.05%).
- **Markowitz puts 93.06% of the portfolio into one stock (YUM).** Its higher raw Sharpe comes entirely from that one bet doing well in the test window, so it isn't a portfolio you could actually run.
- **Black-Litterman spreads the money across 9 of the 15 stocks.** The six stocks whose expected returns were cut most by the ESG views (WYNN, WDC, WELL, DVA, WY and XYL) end up with zero weight. Nothing is excluded by hand.

Sharpe ratios use a risk-free rate of 0.

---

## The problem

Standard mean-variance optimisation is very sensitive to expected-return estimates. Fed noisy historical means, it piles into whichever asset happened to do best. The usual ways of adding ESG also cause trouble: negative screening drops stocks entirely and cuts into the efficient frontier.

This project asks whether ESG risk can enter the portfolio **continuously and in a principled way**, through Bayesian updating, while keeping the portfolio diversified.

## Approach

### 1. Prior: market-implied equilibrium returns

Expected returns start from what the market already implies (reverse optimisation of the cap-weighted portfolio), not from noisy historical means:

<p align="center"><b>Π = δ · Σ · w<sub>mkt</sub></b>, &nbsp; with δ = 2.5</p>

Σ is the annualised covariance of daily simple returns (×252), and w<sub>mkt</sub> are market-cap weights from `market_caps.csv`.

### 2. ESG views: four relative views

Each ESG dimension (Total, Environmental, Social, Governance) becomes one **relative view**, using sign-reversed z-scores. A stock with below-average ESG risk gets a positive loading (it is held long in that view); one with above-average risk gets a negative loading:

<p align="center"><b>z<sub>i</sub><sup>(k)</sup> = ( μ<sup>(k)</sup> − ESG<sub>i</sub><sup>(k)</sup> ) / σ<sup>(k)</sup></b>, &nbsp; for k ∈ { Total, E, S, G }</p>

Stacking the four views gives a 4 × 15 pick matrix. Each row is scaled to unit length so that no single ESG pillar dominates just because its raw scores are bigger:

<p align="center"><b>P = [ z<sub>Total</sub> ; z<sub>E</sub> ; z<sub>S</sub> ; z<sub>G</sub> ]</b>, each row divided by its length (ℓ<sub>2</sub> norm), &nbsp; P ∈ ℝ<sup>4×15</sup></p>

- **View returns:** <b>Q = P · μ<sub>hist</sub></b>. The ESG tilt comes from the structure of P, so no arbitrary penalty number is needed.
- **View uncertainty:** <b>Ω = P (τΣ) Pᵀ</b>, kept as a full (non-diagonal) matrix because the four views are built from the same assets and are therefore correlated.
- **Prior confidence:** <b>τ = 1/T = 1/752 ≈ 0.0013</b>. This keeps the posterior anchored close to market equilibrium.

### 3. Posterior: Bayesian update

<p align="center"><b>V = [ (τΣ)<sup>−1</sup> + Pᵀ Ω<sup>−1</sup> P ]<sup>−1</sup></b></p>

<p align="center"><b>μ<sub>BL</sub> = V [ (τΣ)<sup>−1</sup> Π + Pᵀ Ω<sup>−1</sup> Q ]</b></p>

<p align="center"><b>Σ<sub>pred</sub> = Σ + V</b></p>

### 4. Long-only optimisation

Both Black-Litterman and the Markowitz benchmark are solved as constrained quadratic programmes with `quadprog`:

<p align="center"><b>minimise &nbsp; (A/2) · wᵀ Σ<sub>pred</sub> w − μ<sub>BL</sub>ᵀ w</b><br>subject to &nbsp; Σ<sub>i</sub> w<sub>i</sub> = 1, &nbsp; w<sub>i</sub> ≥ 0, &nbsp; with A = 2.5</p>

Markowitz uses the same formulation with Σ and μ<sub>hist</sub>.

### 5. Out-of-sample backtest

$100,000 is allocated by each strategy's weights (buy-and-hold) and tracked through the test period. The script reports annualised return, volatility, Sharpe ratio, maximum drawdown and final value.

## Portfolio weights

| Ticker | Market cap-weighted | MVO (long-only) | BL ESG (long-only) |
|---|---:|---:|---:|
| YUM | 0.1304 | **0.9306** | 0.2874 |
| ZBH | 0.2244 | 0.0238 | 0.1893 |
| ZTS | 0.1881 | 0 | 0.1334 |
| DRI | 0.0125 | 0 | 0.1177 |
| WMB | 0.0459 | 0 | 0.1058 |
| DPZ | 0.0347 | 0.0457 | 0.0749 |
| XEL | 0.0263 | 0 | 0.0743 |
| EMN | 0.0187 | 0 | 0.0172 |
| DVA, WTW, WELL, WY, XYL, WYNN, WDC | 0.0146 to 0.0768 each | 0 | 0 |

Full prior and posterior expected returns for every stock are in Table 2 of the [term paper](TermPaper.pdf). The posterior pulls each estimate away from noisy history and back toward equilibrium: YUM's historical mean of 26.93% drops to a posterior of 9.84%, and WYNN gets the largest ESG-driven cut (10.43% → 3.59%).

---

## Repository contents

| File | What it is |
|---|---|
| `final.mlx` | MATLAB Live Script with the full pipeline: data, prior, ESG views, posterior, QP, backtest and metrics |
| `TermPaper.pdf` | Full write-up: literature review, method, results and discussion |
| `historical_prices.csv` | Daily adjusted closes for the 15 stocks, Jan 2021 to Dec 2023 (training data) |
| `test_data (1).csv` | Daily adjusted closes for the out-of-sample test period |
| `selected_esg_data.csv` | ESG risk scores (Total, E, S, G), sector, industry and controversy data |
| `market_caps.csv` | Market capitalisation per ticker, used for the equilibrium weights |

**Universe:** DPZ, DRI, DVA, EMN, WDC, WELL, WMB, WTW, WY, WYNN, XEL, XYL, YUM, ZBH, ZTS, covering consumer discretionary, health care, materials, IT, real estate, energy, financials, utilities and industrials.

## How to run

1. **Requirements:** MATLAB with the **Optimization Toolbox** (for `quadprog`) and the **Financial Toolbox** (for `maxdrawdown`).
2. **Rename the test file.** The script reads `test_data.csv`, so rename `test_data (1).csv` to `test_data.csv`.
3. **Open and run `final.mlx`** with the CSV files in the same folder. It prints the posterior returns, the weight table and the performance metrics, and draws:
   - a bar chart comparing the three sets of portfolio weights
   - the cumulative value of each portfolio over the test period

## Limitations and next steps

- One small universe (15 stocks) and a single test window. The results show the method works, not that it beats the market in general.
- τ is fixed at 1/T. It could be tuned with empirical Bayes or cross-validation.
- ESG scores are a single snapshot. Time-varying ESG data would let the views update at each rebalance.
- A larger universe with sector-relative views would test whether the improvement holds more broadly.

## References

- Black, F. and Litterman, R. (1992). Global Portfolio Optimization. *Financial Analysts Journal*.
- Markowitz, H. (1952). Portfolio Selection. *Journal of Finance*.
- Michaud, R. (1989). The Markowitz Optimization Enigma: Is "Optimized" Optimal? *Financial Analysts Journal*.
- Pedersen, L. H., Fitzgibbons, S. and Pomorski, L. (2021). Responsible Investing: The ESG-Efficient Frontier. *Journal of Financial Economics*.
- Walters, J. (2014). The Black-Litterman Model in Detail.

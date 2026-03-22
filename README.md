# Dynamic Delta Hedging Simulation
### A Black-Scholes Based Options Risk Management Simulator in Python

This project implements a **dynamic delta hedging simulation** for European call options
using the Black-Scholes framework. It models the real-world experience of an options
dealer who sells a call option and continuously rebalances a stock position to remain
delta neutral — demonstrating how volatility mispricing directly translates into
profit or loss.

---

## What This Project Does

When a dealer sells a call option, they face unlimited upside risk if the stock rises.
To neutralize this, they buy shares of the underlying in proportion to the option's
**delta** (sensitivity to price). As the stock moves, delta changes — so the hedge
must be rebalanced repeatedly. This project simulates that entire lifecycle, from
selling the option to expiry, across thousands of price paths.

The central insight being tested:

> **The Black-Scholes option price is exactly the cost of manufacturing
> the option through dynamic hedging — but only if realised volatility
> matches the implied volatility used to price it.**

---

## Project Structure

delta_hedging_simulation.ipynb ← Main Jupyter notebook (10 cells)
output/
├── pnl_distribution.png
├── vol_mismatch_pnl.png
└── sample_hedge_path.png


---

## Cell-by-Cell Breakdown

### Cell 2 — Black-Scholes Call Pricing Function
Implements the `bs_call_price(S, K, T, r, sigma)` function using the
Black-Scholes (1973) closed-form formula
d1 = [ln(S/K) + (r + σ²/2)·T] / (σ·√T)
d2 = d1 - σ·√T
C = S·N(d1) - K·e^(−rT)·N(d2)


`N(·)` is the cumulative standard normal distribution. The option is priced
using `sigma_assumed` (implied vol), not actual vol — exactly as a real
market maker would price it.

---

### Cell 3 — Delta Calculation Function
Implements `bs_delta(S, K, T, r, sigma)` which returns:
Δ = N(d1)

This is the partial derivative of the call price with respect to the stock price —
it tells you how many shares to hold per option contract to be instantaneously
delta neutral. A delta of 0.6 means holding 0.6 shares offsets the risk of 1 call.

---

### Cell 4 — Stock Price Path Simulation (Geometric Brownian Motion)
Generates a simulated stock price path using discrete-time GBM:

S(t+dt) = S(t) · exp[(μ - σ²/2)·dt + σ·√dt·Z]


where `Z ~ N(0,1)` is a standard normal random draw.

Key detail: the path is generated using `sigma_actual`, not `sigma_assumed`.
This correctly separates *what the market actually does* from *what the
model assumed it would do*. The Itô correction term `−σ²/2` prevents
the expected stock price from drifting upward over time.

---

### Cell 5 — Hedge Portfolio Initialisation
Sets up the self-financing portfolio at time t=0:

1. **Sell** the call → collect premium `C = bs_call_price(S0, ...)`
2. **Calculate** initial hedge ratio: `Δ₀ = bs_delta(S0, ...)`
3. **Buy** `Δ₀ × n_shares` shares of stock
4. **Cash account** = premium received − cost of shares purchased

This is the Black-Scholes *self-financing replication* strategy. The hedge
portfolio starts with zero net investment — the premium exactly funds the
initial share purchase (plus/minus a cash balance).

---

### Cell 6 — The Dynamic Rebalancing Loop (Core Engine)
The most critical cell. Iterates through each time step from t=1 to t=T:
For each time step i:

    Observe new stock price S[i] from the GBM path

    Grow cash by risk-free rate: cash = cash × e^(r·dt)

    Recalculate delta using sigma_assumed and remaining time (T - t)

    Compute shares to trade: Δ_new - Δ_old

    Buy/sell those shares and debit/credit the cash account

    Record portfolio value, option value, and delta at each step


This rebalancing uses `sigma_assumed` to compute delta — because that is
the model the dealer committed to when they priced the option. **Gamma**
(the rate of change of delta) continuously erodes delta neutrality between
rebalances, which is why perfect hedging is impossible in discrete time.

---

### Cell 7 — Expiry Payoff and Final P&L Calculation
At maturity T, the simulation closes out:

- **Option payoff** the dealer must pay: `max(S_T − K, 0)`
- **Hedge portfolio value**: `shares_held × S_T + cash_account`
- **Hedge P&L**: `portfolio_value − option_payoff`

Under perfect conditions (continuous rebalancing + `sigma_actual = sigma_assumed`),
this P&L equals exactly **zero** — the Black-Scholes theoretical result.
In practice, discretisation error and vol mismatch produce residual P&L.

---

### Cell 8 — Monte Carlo Simulation Over Multiple Paths
Runs the full simulation pipeline (Cells 4–7) N times (`n_paths`),
collecting `hedge_pnl` for each independent stock path.

A single simulation path is statistically meaningless. Running thousands
of paths produces a **distribution of P&L outcomes**, from which we extract:

- `mean(pnl)` → systematic bias from vol mismatch
- `std(pnl)` → residual risk from discrete rebalancing and Gamma exposure
- The shape of the distribution → whether the option was fairly priced

---

### Cell 9 — Volatility Mismatch Sensitivity Analysis
Runs the Monte Carlo simulation for a range of `sigma_actual` values
while holding `sigma_assumed` fixed, recording `mean(pnl)` at each point.

This tests the Gamma P&L formula:
Mean P&L ≈ ½ · Γ · S² · (σ_actual² − σ_implied²) · dt


Results follow the theoretical prediction:
- `sigma_actual > sigma_implied` → **dealer loses** (realised vol was higher
  than priced, Gamma losses exceed premium collected)
- `sigma_actual < sigma_implied` → **dealer profits** (premium was generous
  relative to actual hedging cost)
- `sigma_actual = sigma_implied` → **mean P&L ≈ 0** (fair pricing)

---

### Cell 10 — Visualisation and Output
Generates three charts saved to `output/`:

1. **P&L Distribution Histogram** — distribution of `hedge_pnl` across all
   paths for the base case. A symmetric bell centred on 0 confirms fair pricing.

2. **Mean P&L vs. Vol Mismatch** — plots `mean(pnl)` against
   `(sigma_actual − sigma_assumed)`, showing the parabolic Gamma P&L relationship.

3. **Sample Hedge Path** — plots one simulated stock price path alongside
   the evolving delta, showing how aggressively the hedge rebalances as the
   option moves in/out of the money near expiry.

---

## Key Findings

| Condition | Expected Mean P&L | Interpretation |
|---|---|---|
| `σ_actual = σ_implied` | ≈ 0 | Option fairly priced |
| `σ_actual > σ_implied` | Negative | Dealer under-collected premium |
| `σ_actual < σ_implied` | Positive | Dealer over-collected premium |

Residual standard deviation of P&L (even at fair vol) quantifies the
**discrete rebalancing risk** — the irreducible cost of not hedging continuously.

---




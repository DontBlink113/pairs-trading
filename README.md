# Pairs Trading with a Kalman-Filtered Dynamic Hedge Ratio

This is a cointegration-based statistical-arbitrage pipeline on S&P 500 equities, comparing 
a static OLS hedge ratio against a hand-crafted Kalman filter, recursively estimating the hedge ratio 
at each time. The Kalman algorithm actually did not help create larger or less risky returns; the 
OLS hedge ratio actually performed better. This README attempts to explain why.

## Summary of results

Out-of-sample, across the pairs where both models traded:

| Model            | Trade-wtd Sharpe | Mean ann. RoC | Worst drawdown | Trades |
|------------------|:----------------:|:-------------:|:--------------:|:------:|
| Static OLS       | ~1.17            | ~9%           | -10%           | 19     |
| Kalman (dynamic) | ~1.03            | ~9%           | -8%            | 17     |

The difference is driven almost entirely by a single pair; on the rest the two models are
statistically indistinguishable. Because of the marginal difference between the two on 
most of the 5 tested pairs, and the small data size, it is clear that there is no evidence
that the Kalman filter helps in this situation. It's not exactly clear if the OLS hedge ratio
is, without reservation, a better model.

## Pipeline

1. **Data** — daily adjusted closes for S&P 500 constituents (yfinance), log-prices, split
   into in-sample / out-of-sample before any fitting.
2. **Pair selection** — ~8,700 within-sector candidate pairs (same-sector prior, to avoid
   coincidental cointegration). Engle-Granger test (OLS + ADF on residuals) on each, with a
   Bonferroni correction: 2,661 pairs pass at naive p<0.05, but only 13 survive correction.
3. **Static baseline** — fixed OLS hedge ratio (β, α) from the in-sample window; spread
   z-scored against in-sample mean and std.
4. **Kalman filter** — state [β, α] as a random walk, observation
   `log_y = [log_x, 1] · [β, α]ᵀ + ε`. The standardized innovation `z = e / √S` is the
   trading signal
5. **Signal → position** — enter at |z| > 1.5, exit at |z| < 0.5; hedge the X leg with the
   current β. Next-bar execution, no same-bar fills.
6. **Backtest** — daily PnL, daily re-hedge, no cost penalty
7. **Evaluation** — Sharpe, RoC, drawdown, trade count, hit rate; in-sample vs.
   out-of-sample, static vs. Kalman.

## Estimating Q and R

R (observation noise) is fixed from the in-sample OLS residual variance — it's an estimate,
not a knob. P₀ is the OLS parameter covariance. That leaves Q (process noise) as the only
free parameter, chosen two ways:

- **Log-likelihood** maximizes one-step forecast accuracy.
- **Innovation calibration** picks Q so `std(z) ≈ 1` — the filter's stated uncertainty
  matches its realized errors.

These disagree by orders of magnitude. The likelihood-optimal Q is large and lets β chase
recent data — a better forecaster, but β then absorbs the divergences the strategy is
supposed to trade. The calibrated Q is near-zero, leaving β nearly static.

That near-zero Q is why the filter doesn't help: β still jitters slightly around the OLS
value, and on pairs whose hedge ratio is already stable, that jitter is just noise. A
time-varying β only pays off when the true β drifts, which these pairs didn't do.

## Limitations

Part of the result, not fine print.

- **Survivorship bias** — universe is today's S&P 500, so failed/delisted names are missing.
  Biases every metric upward.
- **Small sample** — ~3–5 trades per pair. The 100% hit rate is a small-sample artifact, not
  a real edge (mean-reversion loses 30–45% of the time normally). 
- **Selection bias** — pairs are best-of-5 out of ~8,700, so their Sharpes are inflated
  order statistics.
- **Equities are a weak fit** — cointegration here is mostly statistical and fragile.
  Commodities, rates, and FX (not implemented) have tighter economic links.
- **Costs are omitted** — no trading fees, or market impact

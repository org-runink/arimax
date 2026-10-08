# arimax

ARIMAX — ARIMA with exogenous regressors — in Go, using **only the standard
library**.

```go
m, err := arimax.Fit(y, x, 1, arimax.Order{P: 1, D: 1, Q: 1})
point, lo, hi, err := m.Forecast(12, xFuture, 0.05)
```

```
Y_t = c + Σ φ_i·Y_{t−i} + Σ θ_j·ε_{t−j} + Σ β_k·X_{t,k} + ε_t
```

## Why it exists

Plain ARIMA assumes a series is driven only by its own past. That is rarely true
of anything worth forecasting: demand responds to price and spend, load responds
to weather, traffic responds to a schedule. ARIMAX puts those drivers in the
model as exogenous regressors, so their effect is estimated rather than absorbed
into noise.

## What makes this implementation different

**Forecasting requires future X, and the API says so.** An ARIMAX forecast `h`
steps ahead needs the exogenous values for those `h` steps. `Forecast` takes
them and returns an error when they are missing — it will not silently assume
zero or hold the last value flat, which are the two ways this quietly produces a
confident wrong answer. Where X is genuinely known ahead — planned spend, a
published timetable, a committed price — this is ARIMAX at its best. Where X is
itself a forecast, its error flows into yours, and the documentation says where
that leaves the intervals.

**Prediction intervals that were actually checked.** Nominal 95% intervals
covered **97.4%** of realised values over 1,800 held-out forecast points. An
interval nobody has measured is decoration; this one has a number and a
reproducible test. See [BENCHMARKS.md](BENCHMARKS.md).

**Zero dependencies.** `go.mod` requires nothing. QR least squares,
Nelder–Mead, the Durbin–Levinson recursion and the inverse normal CDF are each a
few dozen lines here rather than a framework import — which keeps the package
usable in a CLI, an agent sidecar, a WASM build or on a device.

**Deterministic.** Same input, same output. No goroutines, no `math/rand`, no
clock. The tests use a fixed LCG rather than `math/rand`, whose stream is not
guaranteed stable across Go releases.

## Numerical choices, stated rather than hidden

- **Regression with ARIMA errors**, not a joint likelihood: OLS for β, then an
  ARMA fit to the regression errors. Equivalent in what it expresses, far better
  behaved numerically, and each stage is separately testable.
- **Householder QR**, not the normal equations. Exogenous regressors in this
  domain are routinely near-collinear and `(X'X)⁻¹` squares the condition
  number.
- **Conditional sum of squares**, not exact ML. CSS conditions on the first `p`
  observations instead of running a Kalman filter. It is the standard starting
  point and needs no matrix algebra. It is *not* exact ML and this package does
  not claim to be.
- **A conservative stationarity guard**: `Σ|φ| < 1` can reject a stationary
  model, costing a little search space, but it cannot accept an explosive one
  and produce a forecast that runs away.

## API

| | |
|---|---|
| `Fit(y, x, k, Order{P,D,Q})` | fit; `x` row-major, `k` columns, or `nil` |
| `(*Model).Forecast(h, xFuture, alpha)` | point forecast + (1−alpha) interval |
| `Difference` / `SeasonalDifference` | the "I", ordinary and seasonal |
| `ACF` / `PACF` | order identification (MA from ACF, AR from PACF) |
| `LjungBox` | residual diagnostic; returns Q and dof, not a p-value |
| `MAE` / `RMSE` / `MAPE` | accuracy; `MAPE` reports how many zeros it skipped |

`LjungBox` deliberately returns the statistic and its degrees of freedom rather
than a p-value: that needs a chi-squared CDF, and shipping an approximate one
invites people to read significance off a number that is only roughly right.

`MAPE` returns a `skipped` count because MAPE is undefined at zero, and the
usual silent handling — dropping those points, or letting the result become
`+Inf` — has misled people reading a dashboard. Series with genuine zeros are
common.

## Choosing p, d, q

1. Difference until stationary; `d` is usually 0, 1 or rarely 2. Over-differencing
   destroys signal.
2. `PACF` cuts off after lag `p` for an AR process.
3. `ACF` cuts off after lag `q` for an MA process.
4. Fit, then check `LjungBox` on the residuals. Significant structure left means
   the model is misspecified, not that the data is noisy.

## References

Hyndman & Athanasopoulos, *Forecasting: Principles and Practice* — the practical
treatment of regression with ARIMA errors, free online. Box & Jenkins, *Time
Series Analysis: Forecasting and Control* for the original formulation.

## Licence

BSD-3-Clause.

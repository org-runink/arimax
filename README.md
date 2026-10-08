> **Moved.** This module now lives at **[github.com/org-runink/stdx](https://github.com/org-runink/stdx)**.
> The code is unchanged; only the import path differs. This repository is archived.

# arimax

[![Go Reference](https://pkg.go.dev/badge/github.com/org-runink/arimax.svg)](https://pkg.go.dev/github.com/org-runink/arimax)
[![Go Report Card](https://goreportcard.com/badge/github.com/org-runink/arimax)](https://goreportcard.com/report/github.com/org-runink/arimax)
![coverage](https://img.shields.io/badge/coverage-87.0%25-brightgreen)
![dependencies](https://img.shields.io/badge/dependencies-0-blue)
![license](https://img.shields.io/badge/license-BSD--3--Clause-blue)

**Forecast a time series using the things that drive it.** ARIMA assumes a series
is explained by its own past. ARIMAX adds the external drivers — price, spend,
weather, a published schedule — so their effect is estimated instead of being
absorbed into noise.

Pure Go, **zero dependencies**, standard library only.

```bash
go get github.com/org-runink/arimax
```

```go
// y is the series. x holds one row per observation, k columns of drivers.
m, err := arimax.Fit(y, x, 1, arimax.Order{P: 1, D: 1, Q: 1})

// Forecast 12 steps. xFuture is REQUIRED — see below.
point, lo, hi, err := m.Forecast(12, xFuture, 0.05)
```

```
Y_t = c + Σ φ_i·Y_{t−i} + Σ θ_j·ε_{t−j} + Σ β_k·X_{t,k} + ε_t
```

---

## Why this package exists

Three things it does that a textbook implementation usually doesn't.

### 1. It refuses to forecast without the future drivers

An ARIMAX forecast `h` steps ahead needs the exogenous values **for those h
steps**. Most implementations quietly assume zero, or hold the last value flat,
and hand back a confident wrong answer. This one returns an error.

```go
_, _, _, err := m.Forecast(12, nil, 0.05)
// => arimax: forecasting with exogenous regressors requires future X for every step
```

Where X is genuinely known ahead — planned spend, a published timetable, a
committed price — this is ARIMAX at its best. Where X is itself a forecast, its
error flows into yours, and the docs say exactly what that does to the intervals
rather than letting you assume it doesn't.

### 2. Its prediction intervals were measured, not just derived

An interval nobody has checked is decoration. Over 1,800 held-out forecast
points, **nominal 95% intervals covered 97.4%** of realised values — slightly
conservative, which is the safe direction to err. There is a test that proves it
and will fail if it regresses.

### 3. It states its numerical choices instead of hiding them

- **Regression with ARIMA errors**, not a joint likelihood: OLS for β, then an
  ARMA fit to the errors. Equivalent in what it expresses, better behaved
  numerically, and each stage is separately testable.
- **Householder QR**, not the normal equations — real exogenous regressors are
  near-collinear and `(X'X)⁻¹` squares the condition number.
- **Conditional sum of squares**, not exact ML. Standard, needs no Kalman
  filter, and the package says it is not ML rather than implying otherwise.
- **A conservative stationarity guard** that can reject a stationary model but
  cannot accept an explosive one and produce a forecast that runs away.

---

## Measured

```
goos: linux   goarch: amd64   cpu: AMD Ryzen 7 8840U
go test -bench . -benchmem -benchtime=200x
```

**Accuracy** — 200 synthetic series with AR(1) errors and one driver, n=500:

| | true | bias | RMSE |
|---|---:|---:|---:|
| exogenous β | 2.50 | **−0.0005** | 0.0087 |
| AR(1) φ | 0.60 | −0.0079 | 0.0420 |

β comes back essentially unbiased *despite* strongly correlated errors — which is
the whole reason for the staged fit.

| | |
|---|---|
| 95% interval empirical coverage | **97.4%** (1,800 points, h=1..6) |
| vs naive carry-forward | **82.0% lower RMSE** (100 held-out windows) |

**Speed:**

| | ns/op | B/op | allocs/op |
|---|---:|---:|---:|
| `Fit` ARIMAX(1,0,1)+1 driver, n=500 | 659,162 | 489,577 | 235 |
| `Fit` same, n=10,000 | 12,309,872 | 10,324,230 | 249 |
| `Forecast` 24 steps | 11,830 | 72,065 | 10 |
| `ACF` 40 lags, n=5,000 | 305,932 | 761 | **1** |

`Fit` is linear in n (20× the data → 18.7× the time) and allocations are flat.
Fit once, forecast often — forecasting is ~12 µs.

Dev-laptop figures, not capacity numbers. Full detail and **what these numbers do
not show** in [BENCHMARKS.md](BENCHMARKS.md) — including that no head-to-head
against other libraries was run, and that accuracy is measured on synthetic data
because that is the only way to know the truth you are recovering.

---

## Choosing p, d, q

1. Difference until stationary. `d` is usually 0 or 1, rarely 2 — over-differencing destroys signal.
2. `PACF` cuts off after lag `p` → that's your AR order.
3. `ACF` cuts off after lag `q` → that's your MA order.
4. Fit, then check `LjungBox` on residuals. Structure left means misspecified, not noisy.

```go
acf  := arimax.ACF(arimax.Difference(y, 1), 20)
pacf := arimax.PACF(arimax.Difference(y, 1), 20)
q, df := arimax.LjungBox(m.Residuals, 20, m.Order.P+m.Order.Q)
```

`LjungBox` returns the statistic and degrees of freedom, **not** a p-value — that
would need a chi-squared CDF, and an approximate one invites reading significance
off a number that is only roughly right.

---

## Maintainer's map

Every file, what it owns, and roughly how big it is. Nothing here is clever; it
is deliberately boring so it stays maintainable.

| File | Owns |
|---|---|
| `doc.go` | package documentation — the model, the staged fit, the numerical caveats |
| `arimax.go` | `Order`, `Model`, `Fit`, `Forecast`, interval construction, inverse normal CDF |
| `arma.go` | the ARMA stage: CSS residuals, `fitARMA`, stationarity guard, psi-weights |
| `linalg.go` | Householder QR least squares (`olsQR`) |
| `optimize.go` | Nelder–Mead — derivative-free, deterministic simplex |
| `acf.go` | `ACF`, `PACF` (Durbin–Levinson), `LjungBox` |
| `diff.go` | `Difference`, `SeasonalDifference`, and `integrate` (the forecasting inverse) |
| `metrics.go` | `MAE`, `RMSE`, `MAPE` (which reports how many zeros it skipped) |
| `errors.go` | the sentinel errors |

**Where the subtlety is**, so a future change doesn't reintroduce it:

- `integrate` is the forecasting inverse of `Difference`. For `d ≥ 2` it is *not*
  enough to add the tail values back — the inner level needs the last
  **difference** at that level, so the tail is differenced down to recover one
  base per level. This was a real bug, caught by the round-trip test.
- `armaForecast` treats future shocks as zero-expectation, which is correct for
  the conditional mean but means the intervals (not the point forecast) carry all
  the uncertainty.
- `psiWeights` is what makes intervals widen with the right *shape*. If intervals
  ever look wrong, start there.

### Working on it

```bash
go test ./...                                   # unit + accuracy tests
go test -bench . -benchmem -benchtime=200x      # the benchmark table above
go test -cover ./...                            # currently 87.0%
gofmt -l . && go vet ./...                      # must both be silent
```

The accuracy tests use a fixed LCG rather than `math/rand`, whose stream is not
guaranteed stable across Go releases — a test whose data silently changes is
worse than no test. If you add randomness, do the same.

---

## Used by

Open-sourced for the Go community. If your organisation uses `arimax`, we would
like to feature you here.

| Organisation | Use |
|---|---|
| _(yours could be here)_ | |

**To be added:** open a pull request adding a row, or [open an issue](https://github.com/org-runink/arimax/issues/new)
titled `Add <organisation> to Used by` with a one-line description of what you
forecast with it. A link to your site is welcome, and nothing beyond a name and a
sentence is required. We will not ask for logos, case studies or a quote.

---

## Contributing

Issues and pull requests are welcome. Two expectations, both so the package stays
what it is:

1. **No dependencies.** The zero-dependency property is why this is usable in a
   CLI, a sidecar, a WASM build or on a device. A PR that adds a `require` line
   needs a very good argument.
2. **Claims need a test.** If you state an accuracy or performance property,
   there should be a test that fails when it stops being true.

## References

Hyndman & Athanasopoulos, *Forecasting: Principles and Practice* — the practical
treatment of regression with ARIMA errors, free online.
Box & Jenkins, *Time Series Analysis: Forecasting and Control* — the original.

## Licence

BSD-3-Clause. See [LICENSE](LICENSE).

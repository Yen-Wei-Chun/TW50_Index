# TW Index Tracking Simulator

A research project comparing **Full Replication** vs. **Sampled Replication** for tracking a self-defined "Top 30 Taiwan market-cap" basket — quantifying tracking error, validating the selection methodology out-of-sample, and estimating rebalance/reconstitution turnover cost.

## Overview

Index funds and ETFs rarely hold every constituent of the index they track. This project asks two core questions that any buy-side index execution team has to answer:

1. **If you approximate an index with a subset of its constituents, how much tracking error results — and is that error a real, reproducible signal or just noise?**
2. **How much does a single reconstitution event (rebalance) cost, once realistic transaction costs are applied?**

The universe is a self-defined "Top 30 Taiwan market-cap" basket (constructed from public market data), used as a stand-in benchmark rather than an official index.

## Key Findings

> Figures below are from a single run (2026-08-14). Universe selection and price data are pulled live via API, so re-running this notebook will produce numerically different (though methodologically consistent) results — see [How to Run](#how-to-run).

- **Full Replication** tracks the benchmark almost exactly by construction (annualized tracking error effectively **0 bps**), confirmed via an independently-built, share-based buy-and-hold portfolio — a sanity check on the return-calculation pipeline itself, not just a restatement of the benchmark construction.
- **Sampled Replication** (15 of 30 names, selected by correlation to the benchmark) shows an annualized tracking error of **339.72 bps in-sample** and **331.08 bps out-of-sample**. A Newey-West (HAC-robust) significance test on the mean daily return differential finds **neither window is statistically distinguishable from zero** (p = 0.19 in-sample, p = 0.29 out-of-sample, n ≈ 120 each). This is a deliberately cautious conclusion — see [Methodology — Out-of-Sample Validation](#methodology--out-of-sample-validation) for why it matters.
- **Reconstitution** (a single quarterly review, using a separate market-cap-ranked sample to isolate turnover from selection-methodology effects) produces a one-way turnover of **24.24%** and a return drag of **10.181 bps**, or roughly **41 bps/year** extrapolated across four quarterly reviews.
- Taken together, the order-of-magnitude gap between tracking error (hundreds of bps) and turnover cost (tens of bps) is directionally consistent with what's typically observed in practice: **selection risk tends to dominate rebalancing cost** for a concentrated sample, though the two are measured on differently-selected portfolios and shouldn't be read as a precise decomposition for a single fund.

## Methodology — Out-of-Sample Validation

An earlier version of this analysis selected the sampled portfolio by computing correlation-to-benchmark over the **entire** observation window, then evaluated tracking error on that **same** window — a look-ahead bias. That version reported a tracking error of 322.55 bps with a mean daily differential that stayed persistently and visibly positive throughout the whole period, consistent with the selection criterion having been fit and tested on the same data.

This was corrected by:

1. **Splitting the observation window** into a training period (first half) and a held-out test period (second half).
2. **Selecting tickers using only the training window's** correlation to the benchmark — the selection never sees test-window data.
3. **Evaluating tracking error separately** on the training window (in-sample) and the test window (out-of-sample), rather than blending both into a single number.
4. **Testing statistical significance** of the mean daily return differential in each window using both a naive t-test and a Newey-West (HAC) robust version that accounts for potential autocorrelation in daily returns.

The result: neither window shows a statistically significant directional bias. This is a more conservative finding than simply comparing the sign or magnitude of the mean differential between windows would suggest — a mean this close to zero can differ in sign between two non-overlapping windows purely by chance, so significance testing (not just the raw numbers) is what determines whether the original bias has actually been resolved or merely become undetectable at this sample size.

## Methodology — Shares-Outstanding Data Quality

Universe selection depends on `shares_outstanding` from TWSE's `t187ap03_L` API, which does not publish an explicit as-of date and can lag real corporate actions. Cross-checking the computed Top 30 against TWSE's official market-cap ranking surfaced two distinct failure modes:

- **Minor, ongoing lag** (4 tickers: 3037, 2408, 2382, 2327) — the API's share count trails recent capital activity by roughly 1–11%. Small enough that none of these tickers' Top-30 membership is affected, but large enough to bias their weight in the cap-weighted benchmark. Left uncorrected and documented as a known limitation.
- **A severe, single case (6949)** — the API's share count had already updated to reflect a pending 20-for-1 par-value split (NT$10 → NT$0.5) whose effective date hadn't yet been reflected in the traded price. `post-split shares × pre-split price` inflated computed market cap by ~20x, which would have wrongly pulled 6949 into the Top 30. Confirmed against MOPS and corrected via a manual override.

To catch cases like the second one systematically rather than by manual spot-checking alone, the notebook includes:

1. **A scoped consistency check** — cross-validates `shares_outstanding` against `paid-in capital (net of preferred stock) / par value` from the same API response, restricted to a rough Top-60 candidate pool (to avoid noise from small caps, F-shares, and TDRs where this formula doesn't apply).
2. **A manually-verified, source-documented override table** — corrections are only applied after confirming against MOPS directly; the check flags candidates, it does not auto-correct them. One flagged ticker (2887) was investigated and confirmed to be a false positive of the check's formula (a recent merger-related capital structure it doesn't model well), not an actual data issue — documented rather than overridden.

This mechanism is a point-in-time patch, not a permanent fix: overrides need to be revisited once the underlying corporate action (e.g., 6949's split) is fully reflected in price.

## Known Limitations

**Data layer**
- Shares-outstanding data may lag the most recent equity issuance/buyback activity relative to official filings — see [Methodology — Shares-Outstanding Data Quality](#methodology--shares-outstanding-data-quality) for the detection/override mechanism this project uses to catch the more severe cases.
- No free-float adjustment — weights use total market cap rather than investable free-float market cap, which diverges from official index methodologies (e.g., FTSE).
- Adjusted Close prices are retroactively adjusted, not real historical quotes — this doesn't distort relative return/turnover calculations but would matter if compared against external market-cap data.

**Methodology**
- The train/test split is a **single 50/50 split**, not a rolling walk-forward validation. A single split has limited statistical power and can be sensitive to which market regime falls into each window.
- Selection uses correlation-to-benchmark as a one-dimensional proxy for tracking-error minimization. A rigorous version would solve a full covariance-based quadratic program (minimizing `(w_sample − w_benchmark)ᵀ Σ (w_sample − w_benchmark)` under a fixed-cardinality, long-only constraint) — not implemented here to stay within scope.
- Reconstitution (Step 6) only re-ranks within the existing Top-30 universe rather than rescanning the full market, and collapses the ranking-date/effective-date gap that real reconstitutions have.
- The tracking-error and turnover-cost figures come from two differently-selected portfolios (correlation-based vs. market-cap-ranked) and are not a rigorous risk decomposition for a single fund.

## Repository Structure

```
.
├── README.md
├── index_tracking_simulator.ipynb   # Main analysis notebook
└── requirements.txt
```

## Tech Stack

- **Python**: pandas, numpy
- **Data**: yfinance (price history), Taiwan Stock Exchange (TWSE) OpenAPI (`STOCK_DAY_ALL`, `t187ap03_L` for shares outstanding); MOPS used for manual verification of flagged tickers only, not as a programmatic data source
- **Statistics**: scipy (naive t-test), statsmodels (Newey-West/HAC-robust regression)
- **Visualization**: matplotlib

## How to Run

```bash
pip install -r requirements.txt
jupyter notebook index_tracking_simulator.ipynb
```

Run all cells top to bottom. Universe selection and price data are pulled live via API calls, so results will reflect current market data rather than the frozen figures reported above — re-running will produce numerically different (though methodologically consistent) output.

## Scope

This is an independent research project, scoped to fit a short, focused time budget rather than a production-grade index construction system. It intentionally excludes FX hedging, multi-asset indices, corporate-action processing, securities lending/cash management, and continuous multi-period rebalancing — see [Known Limitations](#known-limitations) for what's out of scope and why.

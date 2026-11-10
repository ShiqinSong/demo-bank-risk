# Quant Risk Portfolio Project Plan: Demo Bank Enterprise Risk Framework

2026-11-10

## Overview

One mortgage-centric project that covers the five risk areas most relevant to quant risk roles: market, credit, counterparty, interest rate/liquidity, and model risk. Estimated at 14 weeks; every phase produces something that can go on a resume.

**Setup**: A fictional bank, Demo Bank, holds three types of assets:

- A residential mortgage loan portfolio (Freddie Mac loan-level data)
- An agency MBS investment portfolio
- Treasuries and interest rate swaps used to hedge interest rate risk

The liability side is deposits, used for interest rate risk in the banking book (IRRBB) and liquidity risk analysis.

**Core design**: All modules share a single scenario engine (Hull-White rate model + CCAR macro scenarios). The same set of rate paths drives MBS pricing, prepayment, default, swap exposure, and deposit run-off, mirroring how bank risk systems actually work.

**Out of scope**: Fraud and AML lean toward data science or financial crime roles, so they are left out of the main line.

## Architecture

One scenario engine generates rate and macro paths; five risk modules consume the same paths; a stress testing layer aggregates results; a model risk layer covers every model.

| Layer | Module | Inputs | Outputs |
| --- | --- | --- | --- |
| Scenario engine | Hull-White rate model + macro variable mapping | Treasury curve, CCAR scenarios | Thousands of rate and macro paths |
| Risk module | Credit risk | Loan data, macro paths | PD/LGD/EAD, CECL expected losses |
| Risk module | Prepayment (bridge between credit and market) | Loan data, rate paths | CPR paths |
| Risk module | Market risk | Cash flows, rate paths, historical prices | MBS prices, OAS, duration/convexity, VaR/ES |
| Risk module | Counterparty risk | Swap contracts, rate paths | EE/PFE, CVA |
| Risk module | Interest rate & liquidity risk | Balance sheet, deposit data | EVE/NII sensitivity, simplified LCR, survival horizon |
| Aggregation | Stress testing | Each module's results under the same scenario | Bank-wide losses, P&L, capital and liquidity impact |
| Governance | Model risk | Every model | Development docs, monitoring reports, validation findings and remediation |

## Data Sources

All free, public data. The only gaps are interest rate option volatilities and MBS market prices; these are estimated from historical data instead and documented as model limitations.

| Purpose | Data | Notes |
| --- | --- | --- |
| Loan portfolio | Freddie Mac Single-Family Loan-Level Dataset | Free registration; use the sample files for 2–3 origination years, including monthly performance |
| Yield curve | U.S. Treasury daily par yield curve; FRED DGS series | Hull-White calibration, discounting |
| Macro variables | FRED: unemployment rate UNRATE, 30-year mortgage rate MORTGAGE30US, Case-Shiller home price index | Explanatory variables for PD and prepayment models |
| Stress scenarios | The Federal Reserve's most recent supervisory stress test scenarios (baseline / severely adverse) | CSVs on the Fed website, quarterly macro paths |
| Market risk history | FRED rate history + yfinance (MBS ETFs such as MBB, Treasury ETFs) | Historical simulation VaR and backtesting |
| Deposits | FRED total deposits at commercial banks; FDIC public data | Deposit beta and run-off assumptions |

## Module Details

Each module should be correct, documented, and explainable; state-of-the-art models are not the goal.

### 0. Scenario Engine

- One-factor Hull-White model: θ(t) backed out from the current Treasury curve; a and σ estimated from historical rate data
- Monte Carlo generation of 1,000–5,000 monthly paths, with antithetic variates for variance reduction
- Map the short rate to the 10-year Treasury and 30-year mortgage rate (regression or spread assumption)
- Plug in CCAR macro scenarios: unemployment, home prices, rate paths
- Interview points: why not Vasicek; risk-neutral measure (pricing) vs. real-world measure (risk measurement); negative rates

### 1. Credit Risk

- PD: logistic regression baseline + Cox proportional hazards model; variables include FICO, LTV, DTI, loan age, current LTV (updated with home prices), unemployment rate
- LGD: modeled on the actual loss fields in the Freddie Mac data, linked to current LTV
- EAD: balance from the amortization schedule net of prepayments
- CECL: lifetime expected loss = discounted sum of PD × LGD × EAD over each period, computed under baseline and stress scenarios
- Validation: AUC/KS, calibration curves, out-of-time testing by origination year
- Interview points: point-in-time vs. through-the-cycle PD; CECL vs. IFRS 9; why survival models

### 2. Prepayment

- Competing risks model: default and prepayment are mutually exclusive terminations, estimated jointly with cause-specific hazards or multinomial logistic regression
- Key variables: refinance incentive (note rate − market mortgage rate), loan age, burnout, seasonality, current LTV
- Output CPR under each rate path; backtest predicted vs. actual CPR
- Interview points: why prepayment creates negative convexity in MBS; the 2020–2021 refinance wave and the post-2022 lock-in effect

### 3. Market Risk

- MBS pricing: generate cash flows on each path (principal, interest, prepayments, default recoveries), discount, and average
- OAS: solve for the spread given an assumed market price; effective duration and effective convexity via ±25bp parallel curve shifts
- Portfolio VaR/ES: compare historical simulation, parametric, and Monte Carlo methods; 99% VaR and 97.5% ES
- Backtesting: Kupiec and Christoffersen tests, Basel traffic light zones
- Hedging: hedge duration with Treasuries and swaps; show P&L distributions before and after hedging
- Interview points: negative convexity; why FRTB replaced VaR with ES; OAS vs. Z-spread

### 4. Counterparty Risk

- Revalue the hedging interest rate swaps on the same Hull-White paths to get the exposure distribution at each future date
- Compute EE, EPE, and 95% PFE profiles
- CVA = (1 − recovery rate) × Σ EE × marginal default probability × discount factor, with counterparty default probabilities from an assumed CDS spread
- Compare exposures with and without collateral (CSA) and netting
- Interview points: wrong-way risk; the different uses of EPE and PFE; what SA-CCR is

### 5. Interest Rate & Liquidity Risk

- IRRBB: build a simplified balance sheet; compute EVE and NII changes under ±100/±200bp parallel shifts and the six Basel scenarios
- Deposit modeling: deposit rate beta and behavioral maturity assumptions for core deposits
- Liquidity: simplified LCR (high-quality liquid assets / 30-day net outflows); deposit run-off plus MBS fire-sale haircuts under stress to compute a survival horizon
- Silicon Valley Bank case study: rapid rate rises → unrealized MBS losses → deposit run → forced sales
- Interview points: trade-offs between the EVE and NII perspectives; impact of HTM vs. AFS accounting classification

### 6. Stress Test Aggregation

- Under baseline and severely adverse scenarios, compute 9 quarters of credit losses, MBS and hedge P&L, CVA changes, and NII changes
- Aggregate into a hypothetical capital ratio (CET1) path and liquidity metrics
- Interview points: how to aggregate across risk types and handle correlation; reverse stress testing

### 7. Model Risk

- One development document per model, structured per SR 11-7: purpose, data, methodology, assumptions and limitations, benchmarking, sensitivity analysis
- A shared set of validation functions: stability (PSI), discrimination, calibration, backtesting, parameter sensitivity
- 3–5 validation findings per model, rated by severity, with remediation records
- A model inventory listing each model, its use, risk tier, and last validation date
- Interview points: how would you validate this model; what effective challenge means

## Phased Plan

5 phases over 14 weeks, assuming about 10 hours per week. You can start applying once Phase 2 is done.

### Phase 1 (Weeks 1–3): Data + Scenario Engine

- [ ] Register and download Freddie Mac sample data (2–3 origination years)
- [ ] Clean loan data; build a loan-month panel and termination events (default / prepaid / active)
- [ ] Pull macro and rate data via the FRED API in a reproducible script
- [ ] Implement Hull-White: curve fitting, parameter estimation, path generation; verify zero-coupon bond prices match the market curve
- [ ] Load CCAR scenario CSVs

### Phase 2 (Weeks 4–6): Credit + Prepayment

- [ ] Exploratory analysis: default rate and CPR curves by origination year
- [ ] PD: logistic regression + Cox, out-of-time validation
- [ ] Competing risks prepayment model, CPR backtest
- [ ] LGD and EAD; compute CECL under both scenarios
- [ ] Write the credit model development document (set the template to reuse later)

### Phase 3 (Weeks 7–9): Market Risk

- [ ] MBS cash flow engine and pricing
- [ ] OAS, effective duration, effective convexity; plot the price-yield curve to show negative convexity
- [ ] Portfolio VaR/ES with three methods + backtesting
- [ ] Duration hedge and before/after P&L comparison

### Phase 4 (Weeks 10–11): Counterparty + Interest Rate/Liquidity

- [ ] Swap valuation and exposure simulation; EE/PFE/CVA with and without CSA
- [ ] Simplified balance sheet; EVE/NII sensitivity
- [ ] Deposit run-off assumptions, simplified LCR and survival horizon; SVB case study

### Phase 5 (Weeks 12–14): Aggregation + Governance

- [ ] Stress test aggregation: 9-quarter loss, P&L, capital, and liquidity paths
- [ ] One-page bank-wide risk report (modeled on a risk committee report)
- [ ] Complete development docs, validation findings, remediation records, and the model inventory for all models
- [ ] README, unit tests, one-command run script

## Repository Structure

Keep code, documentation, and reports separate; a reviewer opening the repo should see the README and `docs/` first.

```
demo-bank-risk/
├── README.md               # Business problem → method → results → limitations → governance
├── config/                 # Scenarios, portfolios, model parameters (YAML)
├── data/                   # Download scripts; raw data not committed
├── src/demobank/
│   ├── scenarios/          # Hull-White, macro mapping, CCAR loading
│   ├── credit/             # PD, LGD, EAD, CECL
│   ├── prepayment/         # Competing risks model
│   ├── market/             # MBS pricing, OAS, VaR/ES, backtesting
│   ├── counterparty/       # Swap valuation, EE/PFE, CVA
│   ├── alm/                # EVE/NII, deposits, LCR
│   ├── stress/             # Stress test aggregation
│   └── validation/         # Shared validation functions
├── notebooks/              # One showcase notebook per module
├── docs/
│   ├── model_inventory.md
│   ├── development/        # One development document per model
│   ├── validation/         # Validation findings and remediation records
│   └── reports/            # Monitoring reports, bank-wide risk report
└── tests/
```

## Resume Bullets and Interview Story

Replace the bracketed numbers with your actual results once done; do not make them up.

**Enterprise Risk Framework for a Mortgage-Centric Bank** (Python)

- Built a Hull-White Monte Carlo scenario engine calibrated to the Treasury curve and Fed stress scenarios, shared across credit, market, counterparty and ALM modules
- Modeled default and prepayment as competing risks on [X]M Freddie Mac loan-months; computed lifetime CECL losses under baseline and severely adverse scenarios
- Priced agency MBS with OAS, effective duration and convexity; built VaR/ES with Kupiec and Christoffersen backtesting
- Simulated swap exposures (EE/PFE) and CVA with and without collateral; measured EVE/NII sensitivity and LCR under deposit run-off
- Documented each model per SR 11-7 and performed independent validation with [X] findings and remediation plans

**One-line interview story**: "I built a risk framework for a fictional mortgage-focused bank. All risk modules share one set of rate scenarios, so you can see how a single rate hike simultaneously affects credit losses, MBS prices, swap exposure, and liquidity. Every model is documented and validated per SR 11-7."

When preparing for interviews, practice each module's interview points until you can talk through them for 2 minutes without notes.

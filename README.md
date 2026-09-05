# Akuna Market Making Bot

A quantitative market-making bot developed for the **Akuna Capital Virtual Trading Challenge**.

The challenge involved building a market maker for binary options written on three simulated underlyings. The bot had to estimate fair values, respond to quote requests and fill-or-kill orders, manage inventory and collateral, and remain solvent while competing against other market-making strategies.

## Project Overview

The final approach combines three main components:

1. **Probabilistic option pricing**
   - Estimates the dynamics of the simulated underlyings from historical observations.
   - Prices binary options from the implied terminal payoff probabilities.
   - Handles options with multiple weighted underlying legs.

2. **Adaptive execution and quoting**
   - Quotes bid and ask prices around model-derived fair value.
   - Learns from observed fills and missed trades to estimate how competitive a quote needs to be.
   - Allows bid- and ask-side behaviour to adapt independently rather than assuming symmetric order flow.

3. **Risk-aware position sizing**
   - Uses a Kelly-style expected log-wealth objective to determine trade size.
   - Incorporates existing option inventory when assessing incremental risk.
   - Tracks collateral conservatively so that outstanding quotes and accepted orders cannot make the strategy insolvent.

## Final Strategy

The submitted strategy evolved through a large number of experiments around pricing, sizing, inventory management and execution.

The final bot was based on the **Pipeline-Asymmetric Auction Kelly MM** architecture. Its main idea was to separate the system into a pipeline:

```text
Market history
     |
     v
Parameter estimation
     |
     v
Binary option fair value
     |
     v
Robust valuation bounds
     |
     v
Execution / fill model
     |
     v
Bid and ask selection
     |
     v
Kelly position sizing
     |
     v
Collateral & solvency checks
```

This separation made it easier to test whether performance changes came from the pricing model, the execution policy or the risk layer.

## Key Ideas

### Dynamic parameter estimation

The bot estimates market parameters from the supplied historical time series rather than relying on fixed assumptions. These estimates are updated as the simulated market evolves.

### Binary option pricing

Because the contracts pay either `0` or `1`, their theoretical value is the probability of the terminal payoff event under the model.

For a binary contract,

```text
Fair Value = P(option expires in the money)
```

The implementation uses analytic and numerical probability calculations rather than expensive live Monte Carlo simulation, keeping pricing deterministic and fast enough for repeated quote requests.

### Asymmetric execution learning

One of the main improvements was treating the two sides of the market independently.

Observed fills and misses provide information about the effective competitive quote threshold. The strategy maintains separate bid- and ask-side estimates so that it can react when buying and selling pressure differ.

This avoids forcing a single symmetric spread on both sides of the market.

### Kelly-style sizing

Trade size is chosen using expected log wealth rather than a fixed contract quantity.

Conceptually, the bot chooses quantity `q` to maximise

```text
E[log(W_after(q))] - log(W_before)
```

subject to collateral and solvency constraints.

This naturally reduces position size when available capital is low or the trade is risky, while allowing larger positions when the estimated edge is stronger.

### Portfolio-aware risk

The risk calculation considers existing inventory rather than evaluating each option independently. This allows the bot to recognise cases where a new trade partially offsets an existing position and therefore consumes less incremental risk.

### Conservative collateral accounting

A separate collateral ledger tracks capital committed to outstanding quotes and accepted FOK orders. This was important because a strategy could otherwise appear profitable while accidentally committing more capital than it could afford if several requests were filled simultaneously.

## Development Process

The project was developed experimentally. I built a baseline strategy and then tested isolated changes to understand which assumptions actually mattered.

Experiments included:

- different parameter-estimation windows
- robust versus central fair-value estimates
- inventory-aware Kelly sizing
- execution-probability learning
- asymmetric bid/ask models
- adaptive capital allocation
- option-position netting
- exact strike-ladder portfolio treatment
- finite-sample uncertainty adjustments
- more aggressive quote-search policies

A useful lesson from the challenge was that **greater model complexity did not automatically improve trading performance**. Several theoretically appealing modifications produced no measurable improvement in the hidden test cases. Keeping changes isolated made it possible to distinguish genuine improvements from overfitting to visible scenarios.

## What I Learned

This challenge was particularly useful for connecting quantitative modelling with the practical constraints of an execution system.

The main lessons were:

- A good fair-value model is only one part of a market maker; execution probability matters just as much.
- Risk should be assessed at the portfolio level rather than contract-by-contract where possible.
- Kelly sizing provides a principled bridge between estimated edge and capital allocation.
- Hidden-state information can often be inferred indirectly from order flow and fill behaviour.
- Conservative bookkeeping and solvency constraints are as important as the trading logic itself.
- In a noisy environment, simple robust strategies can outperform more complicated models that introduce additional estimation error.

## Repository Structure

The repository is intended to contain the final submission and supporting documentation.

```text
AkunaMarketMakingBot/
├── README.md
└── pipeline_asymmetric_v18.py
```

## Technologies

- Python 3
- `dataclasses`
- `collections`
- `statistics`
- numerical probability methods
- stochastic modelling
- Kelly criterion / expected-log optimisation

## Disclaimer

This repository is a personal project based on my participation in a trading challenge. It is provided for educational and portfolio purposes and is not intended as financial advice or as a production trading system.

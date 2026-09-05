# Akuna Market Making Bot

A quantitative market-making bot developed for the **Akuna Capital Virtual Trading Challenge**.

The task was to build a market maker for binary options written on three simulated underlyings. The bot had to infer market dynamics from historical data, value contracts, respond to RFQs and fill-or-kill orders, manage inventory and collateral, and remain solvent while competing against other market-making strategies.

The final strategy is the **Pipeline-Asymmetric Auction Kelly MM**. The exact source file is preserved in `pipeline_asymmetric_v18.zip`.

## Strategy at a Glance

```text
Historical market data
        |
        v
Estimate stochastic dynamics
        |
        v
Construct terminal distributions
        |
        v
Calculate P(option expires ITM)
        |
        v
Fair value + uncertainty bounds
        |
        v
Estimate probability of winning the auction
        |
        v
Search executable bid / ask prices
        |
        v
Portfolio Kelly sizing
        |
        v
Collateral and solvency checks
```

The design deliberately separates **prediction**, **execution**, and **risk**. This made it easier to test whether an improvement came from better pricing, better quote placement, or better capital allocation.

---

## 1. Contract Mathematics

Each option is a binary event contract:

```math
X = \mathbf{1}\!\left[\sum_i w_i S_i(T) \ge K\right]
```

where:

- $S_i(T)$ is the terminal value of underlying $i$,
- $w_i$ is its option weight,
- $K$ is the strike,
- $X \in \{0,1\}$ is the settlement payoff.

Ignoring discounting, the fair value is therefore

```math
V = \mathbb{E}[X]
```

and because $X$ is binary,

```math
V = \Pr\!\left(\sum_i w_i S_i(T) \ge K\right).
```

So pricing becomes a probability problem: estimate the probability that the terminal weighted basket finishes above the strike.

---

## 2. Modelling the Underlyings

The simulated market contains one discrete interest-rate process and two company-value processes, AJR and THR.

### 2.1 Mean-Reverting Rate Process

From the observed rate history, the bot estimates:

- baseline upward-move probability $u_0$,
- baseline downward-move probability $d_0$,
- rate step $\Delta r$,
- long-run target $r^*$,
- mean-reversion strength $\kappa$.

At current rate $r_t$, define the mean-reversion tilt

```math
\tau_t = \kappa(r^* - r_t).
```

The transition probabilities are then

```math
p_{\uparrow}(r_t) = \mathrm{clip}(u_0 + \tau_t),
```

```math
p_{\downarrow}(r_t) = \mathrm{clip}(d_0 - \tau_t),
```

and

```math
p_0(r_t) = 1 - p_{\uparrow}(r_t) - p_{\downarrow}(r_t).
```

The parameters are fitted by maximising the log-likelihood of the observed up/down/stay transitions using a constrained coordinate search.

The terminal rate distribution is then propagated exactly through dynamic programming. From any current state $r$:

```math
P_{t+1}(r+\Delta r) = P_{t+1}(r+\Delta r) + P_t(r)p_{\uparrow}(r),
```

```math
P_{t+1}(r-\Delta r) = P_{t+1}(r-\Delta r) + P_t(r)p_{\downarrow}(r),
```

```math
P_{t+1}(r) = P_{t+1}(r) + P_t(r)p_0(r).
```

This gives an exact discrete terminal-rate distribution without Monte Carlo noise.

### 2.2 Company Log-Return Model

For each company, one-period log returns are modelled as

```math
\log\!\left(\frac{S_{t+1}}{S_t}\right)
= \alpha + \beta\,\Delta r_t + \varepsilon_t.
```

The drift $\alpha$ and rate sensitivity $\beta$ are estimated using ordinary least squares.

Over $T$ periods, conditional on the terminal rate, the model is approximately

```math
\log S_T
\approx \log S_0 + T\alpha + \beta(r_T-r_0) + \text{noise}.
```

Therefore each company is conditionally lognormal.

### 2.3 Correlated Company Shocks

The AJR and THR regression residuals are not assumed independent. Their covariance matrix is estimated as

```math
\Sigma =
\begin{pmatrix}
\sigma_A^2 & \sigma_{AT} \\
\sigma_{AT} & \sigma_T^2
\end{pmatrix}.
```

The code represents this through a shared sector factor plus idiosyncratic shocks:

```math
\varepsilon_A = b_A Z_s + \sigma_{A,\mathrm{id}}Z_A,
```

```math
\varepsilon_T = b_T Z_s + \sigma_{T,\mathrm{id}}Z_T.
```

This reproduces the estimated variances and covariance while keeping the pricing calculation tractable.

A weak variance prior is included for very small samples so that a short warm-up history does not falsely imply deterministic company values.

---

## 3. Binary Option Pricing

For every possible terminal rate $r$, the model calculates

```math
P(R_T=r).
```

The option value is then obtained by conditioning on the terminal rate:

```math
V
= \sum_r P(R_T=r)
\Pr\!\left(\sum_i w_iS_i(T) \ge K \mid R_T=r\right).
```

### 3.1 Single-Company Contracts

If

```math
\log S_T \sim N(\mu,\sigma^2),
```

then for a positive weight $w$,

```math
\Pr(wS_T \ge K)
= 1 - \Phi\!\left(\frac{\log(K/w)-\mu}{\sigma}\right).
```

The implementation evaluates this using the complementary error function.

### 3.2 Two-Company Contracts

A weighted sum of correlated lognormal variables generally has no simple closed-form distribution.

Instead of running a large Monte Carlo simulation for every quote, the bot conditions on one Gaussian factor. If $Z_A$ and $Z_T$ are correlated standard normals, then

```math
Z_T \mid Z_A=z
\sim N(\rho z, 1-\rho^2).
```

This reduces the two-dimensional problem to a one-dimensional numerical expectation. The code evaluates it using deterministic normal quantile points:

- `64` points for live pricing,
- `128` points for higher-accuracy theoretical pricing.

This is faster and less noisy than live Monte Carlo.

---

## 4. Parameter-Uncertainty Bounds

A single fitted parameter set can be overconfident, especially with limited data. The bot therefore perturbs important fitted quantities, including:

- rate transition probabilities,
- mean-reversion strength,
- company drift,
- rate sensitivity,
- residual variance,
- residual correlation.

For each plus/minus perturbation pair, the option is repriced. The local pricing uncertainty is approximated by

```math
\widehat{\sigma}_V^2
= \sum_j
\left(\frac{V_j^+ - V_j^-}{2}\right)^2.
```

The conservative pricing interval is then approximately

```math
V^- = \max\!\left(0, V - z\widehat{\sigma}_V - c\right),
```

```math
V^+ = \min\!\left(1, V + z\widehat{\sigma}_V + c\right),
```

with $z=1.645$ and a small numerical allowance $c=0.003$.

The bot uses the conservative side of this interval when deciding whether a quote has enough edge.

---

## 5. Execution as an Auction Problem

Fair value alone is not enough. A quote also needs to trade.

The strategy models each RFQ as an auction against an unknown effective rival quote. Let $d$ denote the quote's distance from fair value in ticks, and let $C$ be the latent rival cutoff.

A quote is competitive when approximately

```math
C \ge d.
```

The fill probability is therefore the posterior survival probability

```math
P(\text{fill at distance }d)
= P(C \ge d \mid \text{observations}).
```

### 5.1 Bayesian Updating from Fills and Misses

The initial prior over $C$ is geometric, representing a rapidly decreasing chance that increasingly wide quotes remain competitive.

After a fill or miss, incompatible cutoff states are removed and the remaining mass is renormalised:

```math
P(C=c \mid O)
\propto P(O \mid C=c)P(C=c).
```

The bid and ask sides maintain **separate posterior distributions**. This is the core reason for the *Pipeline-Asymmetric* name: buy-side and sell-side competition need not be identical.

### 5.2 Hidden RFQ Direction

If neither side of a two-sided quote fills, the bot does not directly observe whether the counterparty intended to buy or sell.

It therefore uses a soft responsibility weight. Conceptually,

```math
\gamma
=
\frac{P(\text{sell})P(\text{bid miss}\mid\text{sell})}
{P(\text{sell})P(\text{bid miss}) + P(\text{buy})P(\text{ask miss})}.
```

The bid-side posterior receives weight $\gamma$ and the ask-side posterior receives weight $1-\gamma$.

This is similar to a latent-variable or soft-assignment update.

---

## 6. Quote Optimisation

The bot searches the executable penny grid independently on the bid and ask sides.

For each candidate price $x$, it evaluates:

1. whether the quote has positive robust model edge,
2. the probability of winning allocation,
3. the Kelly-optimal quantity,
4. the expected log-wealth gain if filled,
5. the collateral-safe executable quantity.

The basic objective is

```math
\mathrm{score}(x)
= P(\mathrm{fill}\mid x)\times \Delta U(x,q^*).
```

This explicitly trades off **profit per fill** against **probability of getting filled**.

The bot also records allocated trade sizes by counterparty. If a counterparty historically trades less than the full displayed quantity, expected utility is evaluated using the likely allocated quantity rather than assuming the whole quote trades.

---

## 7. Kelly Criterion and Position Sizing

The core capital-allocation rule is expected log wealth.

Suppose terminal state $s$ has:

- probability $p_s$,
- current state-dependent wealth $W_s$,
- contract payoff $X_s\in\{0,1\}$,
- execution price $x$,
- trade direction $a\in\{+1,-1\}$,
- quantity $q$.

Then terminal wealth becomes

```math
W_s'(q) = W_s + qa(X_s-x).
```

The strategy chooses $q$ to maximise

```math
\Delta U(q)
= \sum_s p_s
\log\!\left(\frac{W_s'(q)}{W_s}\right).
```

Equivalently,

```math
\Delta U(q)
= \sum_s p_s
\log\!\left(1+\frac{qa(X_s-x)}{W_s}\right).
```

Log utility rewards positive edge while strongly penalising trades that put too much capital at risk.

### 7.1 Admissible Quantity Domain

Expected log utility is only defined if terminal wealth remains positive in every state with non-zero probability:

```math
W_s + qa(X_s-x) > 0.
```

The implementation calculates the maximum integer quantity satisfying these pathwise inequalities before attempting optimisation.

### 7.2 Fast Optimisation

For a simple two-state binary event, the continuous first-order condition has a single root. The code calculates that root and checks the neighbouring integer quantities.

For a more general portfolio state representation,

```math
\frac{d\Delta U}{dq}
=
\sum_s
p_s
\frac{a(X_s-x)}{W_s+qa(X_s-x)}.
```

Because expected log utility is concave, this derivative is monotone. The program therefore locates the sign change with binary search on the integer quantity lattice rather than checking every possible size.

---

## 8. Exact Treatment of Nested Strike Ladders

Options with proportional leg vectors and the same expiry are nested events.

For strikes

```math
k_1 < k_2 < \cdots < k_m,
```

the contracts can be written as

```math
X_i = \mathbf{1}\!\left[Y \ge k_i\right].
```

If

```math
p_i = P(Y \ge k_i),
```

then the full joint distribution follows directly from the marginals:

```math
P(Y<k_1)=1-p_1,
```

```math
P(k_i\le Y<k_{i+1})=p_i-p_{i+1},
```

```math
P(Y\ge k_m)=p_m.
```

This lets the bot construct exact terminal portfolio-wealth states for an entire strike ladder without incorrectly assuming that those contracts are independent.

It also allows a lower-strike long and a higher-strike short to receive the correct diversification credit.

---

## 9. Solvency and Collateral

For a long binary bought at price $x$, maximum loss per unit is

```math
L_{\mathrm{long}} = x.
```

For a short binary sold at price $x$, maximum loss per unit is

```math
L_{\mathrm{short}} = 1-x.
```

For quantity $q$, the raw collateral requirement is therefore

```math
D = qL.
```

The bot keeps a separate reservation ledger for outstanding quotes and accepted FOK orders so that multiple simultaneous fills cannot accidentally commit more capital than is available.

### 9.1 Guaranteed Hedge Credit

Suppose the portfolio holds $L$ long contracts and $S$ short contracts on the same binary event. At expiry, total payoff is

```math
LX + S(1-X),
```

where $X\in\{0,1\}$.

The minimum possible payoff is

```math
\min(L,S).
```

This is a pathwise guaranteed credit, not an expected-value approximation. The bot uses it when assessing solvency and collateral requirements.

---

## 10. Fill-or-Kill Orders

For a FOK order, the strategy checks three separate conditions:

1. **Positive edge** — the order price must beat model fair value in the correct direction.
2. **Kelly desirability** — the requested size must not exceed the portfolio-optimal quantity and must produce positive expected log gain.
3. **Exact affordability** — collateral after guaranteed hedge credit must remain within the available solvency slack.

This prevents the bot from accepting a statistically attractive trade whose requested size creates unacceptable pathwise risk.

---

## 11. Development Process

I developed the strategy experimentally from a simpler baseline and tried to isolate changes so I could identify *why* performance moved.

Experiments included:

- stationary versus adaptive parameter estimation,
- recency weighting,
- robust versus central fair values,
- different uncertainty allowances,
- fixed versus Kelly sizing,
- inventory-aware sizing,
- option-position netting,
- exact strike-ladder portfolio treatment,
- execution-probability learning,
- asymmetric bid/ask learning,
- order-size learning,
- adaptive capital allocation,
- more aggressive quote-search policies,
- finite-sample uncertainty corrections.

A major lesson was that **greater model complexity did not automatically improve trading performance**. Several theoretically appealing changes materially altered the model while barely moving the hidden-test score.

That made the challenge a useful exercise in avoiding overfitting and distinguishing mathematical sophistication from economically useful signal.

---

## 12. What I Learned

The project connected several ideas that are often studied separately:

- time-series estimation,
- maximum-likelihood estimation,
- ordinary least squares,
- covariance modelling,
- conditional probability,
- lognormal option modelling,
- deterministic numerical integration,
- finite-difference uncertainty analysis,
- Bayesian updating,
- latent-variable responsibility weighting,
- Kelly optimisation,
- concave optimisation and root finding,
- exact joint-distribution reconstruction for nested events,
- pathwise collateral constraints.

The broader trading lesson was that fair value is only one component of a market maker. The final decision combines

```math
\text{prediction}
+ \text{execution probability}
+ \text{portfolio risk}
+ \text{capital constraints}.
```

---

## Repository Structure

```text
AkunaMarketMakingBot/
├── README.md
└── pipeline_asymmetric_v18.zip
    └── pipeline_asymmetric_v18.py
```

The ZIP contains the exact final Python source preserved from the challenge work.

## Technologies

- Python 3
- `dataclasses`
- `collections`
- `statistics.NormalDist`
- stochastic-process modelling
- ordinary least squares
- numerical integration
- Bayesian updating
- Kelly criterion / expected-log optimisation

## Disclaimer

This repository is a personal project based on my participation in a trading challenge. It is provided for educational and portfolio purposes only. It is not financial advice and is not intended to be a production trading system.
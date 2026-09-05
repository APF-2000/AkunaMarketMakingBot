# Akuna Market Making Bot

A quantitative market-making bot developed for the **Akuna Capital Virtual Trading Challenge**.

The task was to build a market maker for binary options written on three simulated underlyings. The bot had to infer the dynamics of the market from historical data, value new contracts, quote competitively against other market makers, size positions under uncertainty, respond to fill-or-kill orders, and remain solvent throughout the simulation.

The final strategy is the **Pipeline-Asymmetric Auction Kelly MM** implemented in `pipeline_asymmetric_v18.py`.

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
P(option expires ITM)
        |
        v
Fair value + parameter-uncertainty bounds
        |
        v
Estimate probability of winning an auction
        |
        v
Search executable bid / ask prices
        |
        v
Portfolio Kelly sizing
        |
        v
Exact collateral / solvency checks
```

The design deliberately separates **prediction**, **execution**, and **risk**. That made it possible to test changes to each layer independently instead of treating the market maker as one opaque model.

---

# 1. Contract Mathematics

Each option is a binary event contract of the form

$$
X = \mathbf{1}\left\{\sum_i w_i S_i(T) \ge K\right\},
$$

where:

- $S_i(T)$ is the terminal value of underlying $i$,
- $w_i$ is the option-leg weight,
- $K$ is the strike,
- $X\in\{0,1\}$ is the settlement payoff.

Ignoring discounting, the model value of the option is therefore simply

$$
V = \mathbb{E}[X]
  = \Pr\left(\sum_i w_i S_i(T)\ge K\right).
$$

This was useful because pricing a binary option becomes a **probability problem** rather than a general expected-payoff simulation problem.

---

# 2. Modelling the Underlyings

The simulated market contains a discrete interest-rate process and two company-value processes, AJR and THR.

## 2.1 Mean-reverting rate process

The rate moves on a discrete grid. From the observed history I estimate:

- baseline probability of an upward move $u_0$,
- baseline probability of a downward move $d_0$,
- grid step $\Delta r$,
- long-run target level $r^*$,
- mean-reversion strength $\kappa$.

At current rate $r_t$, the transition probabilities are tilted toward the target:

$$
\tau_t = \kappa(r^*-r_t),
$$

$$
p_{\uparrow}(r_t)=\operatorname{clip}(u_0+\tau_t),
$$

$$
p_{\downarrow}(r_t)=\operatorname{clip}(d_0-\tau_t),
$$

with

$$
p_{0}(r_t)=1-p_{\uparrow}(r_t)-p_{\downarrow}(r_t).
$$

The parameters are fitted by maximising the log-likelihood of the observed up/down/stay transitions. I used a constrained coordinate search rather than introducing an external optimiser.

Given these transition probabilities, the exact distribution of the rate after $T$ steps is propagated recursively:

$$
P_{t+1}(r+\Delta r) \mathrel{+}= P_t(r)p_{\uparrow}(r),
$$

$$
P_{t+1}(r-\Delta r) \mathrel{+}= P_t(r)p_{\downarrow}(r),
$$

$$
P_{t+1}(r) \mathrel{+}= P_t(r)p_0(r).
$$

This avoids Monte Carlo noise for the discrete rate component.

## 2.2 Company log-return model

For each company I model one-period log returns as

$$
\log\frac{S_{t+1}}{S_t}
= \alpha + \beta\,\Delta r_t + \varepsilon_t.
$$

The drift $\alpha$ and rate sensitivity $\beta$ are estimated with ordinary least squares using the observed rate changes and company log returns.

This gives a conditional multi-period model of the form

$$
\log S_T
\approx \log S_0 + T\alpha + \beta(r_T-r_0) + \text{stochastic shock}.
$$

So conditional on the terminal rate, each company is approximately lognormal.

## 2.3 Correlated company shocks

The residuals from the two regressions are not assumed independent. Their empirical residual covariance matrix is estimated as

$$
\Sigma =
\begin{pmatrix}
\sigma_A^2 & \sigma_{AT}\\
\sigma_{AT} & \sigma_T^2
\end{pmatrix}.
$$

The code factorises this into a shared sector component and idiosyncratic components:

$$
\varepsilon_A = b_A Z_s + \sigma_{A,\text{id}}Z_A,
$$

$$
\varepsilon_T = b_T Z_s + \sigma_{T,\text{id}}Z_T,
$$

where $Z_s,Z_A,Z_T$ are standard normal shocks.

This reproduces the estimated marginal variances and covariance while keeping the pricing calculation tractable.

A weak variance prior is included when the warm-up sample is very small so that a one- or two-observation history does not incorrectly imply deterministic company values.

---

# 3. Pricing the Binary Options

For each possible terminal rate $r$, the model first calculates

$$
P(R_T=r).
$$

The total option price is then obtained by conditioning on that rate:

$$
V
= \sum_r P(R_T=r)
  \Pr\left(\sum_i w_iS_i(T)\ge K\mid R_T=r\right).
$$

## 3.1 One-company contracts

For a single company whose conditional log value satisfies

$$
\log S_T\sim N(\mu,\sigma^2),
$$

a positive-weight event

$$
wS_T\ge K
$$

has probability

$$
\Pr(wS_T\ge K)
= 1-\Phi\left(\frac{\log(K/w)-\mu}{\sigma}\right).
$$

The code evaluates this with the complementary error function.

## 3.2 Two-company contracts

For contracts involving both company underlyings, the exact event generally has no convenient closed form because it contains a weighted sum of correlated lognormal variables.

Instead of running a large Monte Carlo simulation for every quote, the implementation conditions on one Gaussian factor. If

$$
(Z_A,Z_T)
$$

are correlated normals, then

$$
Z_T\mid Z_A=z
\sim N(\rho z,1-\rho^2).
$$

This reduces the two-dimensional problem to a deterministic one-dimensional normal integration. The code evaluates this using fixed normal quantile points (`64` for live pricing and `128` for higher-accuracy theoretical pricing).

The result is fast, deterministic and substantially less noisy than live Monte Carlo.

---

# 4. Parameter-Uncertainty Bounds

A single fitted parameter set can be overconfident, particularly with limited history. The bot therefore constructs local perturbations of important fitted parameters, including:

- rate up/down probabilities,
- mean-reversion strength,
- company drift,
- rate beta,
- residual variance,
- residual correlation.

For each plus/minus perturbation pair, the option is repriced. A finite-difference uncertainty estimate is accumulated as

$$
\widehat{\sigma}_V^2
= \sum_j\left(\frac{V_j^+-V_j^-}{2}\right)^2.
$$

The quoted valuation interval is approximately

$$
[V^-,V^+]
=
\left[
V-z\widehat{\sigma}_V-c,
V+z\widehat{\sigma}_V+c
\right]\cap[0,1],
$$

with $z=1.645$ and a small numerical allowance $c=0.003$ for penny rounding and numerical integration error.

I use the conservative side of this interval when deciding whether a bid or ask has sufficient edge.

---

# 5. Execution as an Auction Problem

Correct fair value is not sufficient for a profitable market maker. A quote also needs to trade.

The strategy treats each RFQ as an auction against an unknown effective rival quote. Define the **distance from fair value** in ticks as $d$ and a latent rival cutoff $C$.

A quote receives allocation when approximately

$$
C\ge d.
$$

The initial prior over $C$ is geometric, corresponding to a rapidly decreasing chance that increasingly wide quotes remain competitive.

The fill probability used by the bot is the posterior survival function

$$
P(\text{fill at distance }d)=P(C\ge d\mid\text{observations}).
$$

## 5.1 Bayesian conditioning from fills and misses

A fill at distance $d$ rules out all cutoff states below $d$. A known miss rules out all states at or above $d$. The posterior is therefore formed by zeroing incompatible states and renormalising:

$$
P(C=c\mid O)
\propto P(O\mid C=c)P(C=c).
$$

The bid and ask sides maintain **separate posterior distributions**. This was the main reason for the "Asymmetric" name: buying and selling pressure need not behave identically.

## 5.2 Hidden-side RFQs

When a two-sided quote receives no fill, the bot does not directly know whether the counterparty wanted to buy or sell. It therefore calculates a responsibility weight from the counterparty's estimated side probability and the miss probability on each side.

Conceptually,

$$
\gamma
=
\frac{P(\text{sell})P(\text{bid miss}\mid\text{sell})}
{P(\text{sell})P(\text{bid miss})+P(\text{buy})P(\text{ask miss})}.
$$

The bid-side posterior receives weight $\gamma$ and the ask-side posterior receives $1-\gamma$.

This is similar to a latent-variable or soft-assignment update.

---

# 6. Quote Optimisation

The bot searches the executable penny grid independently on each side.

For a candidate price $x$, it calculates:

1. whether the price has positive model edge,
2. the probability that the quote wins allocation,
3. the Kelly-optimal quantity,
4. the expected log-wealth gain if filled,
5. the collateral-safe executable quantity.

The basic quote score is

$$
\text{score}(x)
= P(\text{fill}\mid x)\times \Delta U(x,q^*),
$$

where $\Delta U$ is the incremental expected log utility.

So the bot is not simply choosing the widest profitable spread or the highest fill probability. It trades off **edge against execution probability**.

Observed allocated order sizes are also stored. When historical orders from a counterparty tend to be smaller than the full Kelly quantity, the expected gain is evaluated using the likely actually allocated size rather than assuming every winning quote trades the entire displayed quantity.

---

# 7. Kelly Criterion and Portfolio Sizing

The central risk-allocation idea is the Kelly criterion.

If terminal state $s$ has probability $p_s$, current state-dependent wealth $W_s$, contract payoff $X_s\in\{0,1\}$, execution price $x$, trade direction $a\in\{+1,-1\}$ and quantity $q$, then terminal wealth becomes

$$
W'_s(q)=W_s+qa(X_s-x).
$$

The strategy chooses $q$ to maximise incremental expected log wealth:

$$
\Delta U(q)
=
\sum_s p_s
\log\left(\frac{W'_s(q)}{W_s}\right)
=
\sum_s p_s
\log\left(1+\frac{qa(X_s-x)}{W_s}\right).
$$

The log utility has two useful properties here:

- it rewards profitable edge,
- it strongly penalises strategies that risk exhausting capital.

## 7.1 Exact admissible domain

Kelly optimisation is only meaningful when terminal wealth remains positive in every state with non-zero probability:

$$
W_s+qa(X_s-x)>0\qquad\forall s:p_s>0.
$$

The implementation explicitly calculates the maximum integer quantity satisfying these inequalities before optimising.

## 7.2 Closed-form binary case

For a simple two-state binary payoff, the derivative of expected log wealth has a single root. The code calculates this continuous root, checks the adjacent integer quantities, and selects the better one.

For the more general portfolio state representation, expected log utility remains concave. Its derivative is

$$
\frac{d\Delta U}{dq}
=
\sum_s
p_s\frac{a(X_s-x)}{W_s+qa(X_s-x)}.
$$

Because this derivative is monotone, the program locates its sign change with a binary search on the integer quantity lattice rather than scanning every possible size.

---

# 8. Exact Treatment of Nested Strike Ladders

A particularly useful portfolio observation is that options with proportional leg vectors and the same expiry are **nested events**.

For strikes

$$
k_1<k_2<\cdots<k_m,
$$

the contracts are

$$
X_i=\mathbf{1}\{Y\ge k_i\}.
$$

If

$$
p_i=P(Y\ge k_i),
$$

then their complete joint distribution follows directly from the marginals:

$$
P(Y<k_1)=1-p_1,
$$

$$
P(k_i\le Y<k_{i+1})=p_i-p_{i+1},
$$

$$
P(Y\ge k_m)=p_m.
$$

This means I can construct exact terminal portfolio-wealth states for an entire strike ladder without assuming independence between contracts.

That matters because a long lower-strike binary and a short higher-strike binary can partially hedge each other even though they have different option IDs.

---

# 9. Solvency and Collateral Mathematics

The challenge required the bot to remain solvent under the engine's cash accounting, so I maintained an explicit conservative collateral ledger.

For a long binary bought at price $x$, maximum loss per unit is

$$
L_{\text{long}}=x.
$$

For a short binary sold at $x$, maximum loss is

$$
L_{\text{short}}=1-x.
$$

Thus a quantity $q$ has raw maximum-loss debit

$$
D=qL.
$$

Outstanding quotes and accepted FOKs reserve this amount before the trade is known to have disappeared.

## 9.1 Guaranteed offset credit

For the same binary event, suppose the portfolio contains $L$ longs and $S$ shorts. At expiry its payoff is

$$
LX+S(1-X),\qquad X\in\{0,1\}.
$$

The minimum possible payoff is therefore

$$
\min(L,S).
$$

This gives an exact pathwise lower bound on settlement cash. The bot uses this guaranteed credit when assessing how much additional quantity can safely be accepted.

This lets opposite positions net where the hedge is mathematically guaranteed, without relying on model correlation or expected value.

---

# 10. FOK Orders

For a fill-or-kill order, the strategy checks three separate conditions:

1. **Positive edge:** the offered trade price must beat the model fair value in the correct direction.
2. **Kelly desirability:** the requested size cannot exceed the portfolio-optimal quantity and must have positive expected log gain.
3. **Exact affordability:** collateral after guaranteed hedge credit must remain within available solvency slack.

This separation prevents a statistically attractive trade from being accepted simply because its expected value is positive when its requested size would create unacceptable pathwise risk.

---

# 11. Development Process

I developed the strategy experimentally from a much simpler baseline and tried to isolate changes so I could tell *why* performance moved.

Experiments included:

- stationary versus adaptive parameter estimation,
- recency weighting,
- robust versus central fair values,
- different confidence allowances,
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

One of the most useful findings was that **increasing complexity did not reliably increase score**. Some theoretically appealing modifications changed the model substantially while leaving hidden-test performance essentially unchanged. That was a useful lesson in avoiding overfitting and in distinguishing modelling sophistication from economically useful signal.

---

# 12. What I Learned

The project connected several ideas that are often taught separately:

- **time-series estimation** for learning market dynamics,
- **maximum likelihood** for the discrete rate process,
- **OLS regression** for conditional company returns,
- **covariance factorisation** for correlated shocks,
- **conditional probability and numerical integration** for option pricing,
- **finite-difference sensitivity analysis** for model uncertainty,
- **Bayesian updating** for execution learning,
- **latent-variable responsibility weighting** when order direction is hidden,
- **Kelly optimisation** for position sizing,
- **concavity and derivative root-finding** for fast integer optimisation,
- **joint-distribution reconstruction** for nested binary options,
- **pathwise inequalities** for collateral and solvency.

The broader trading lesson was that fair value is only one part of market making. The final decision needs to combine

$$
\text{prediction} + \text{execution probability} + \text{portfolio risk} + \text{capital constraints}.
$$

---

# Repository Structure

```text
AkunaMarketMakingBot/
├── README.md
└── pipeline_asymmetric_v18.py
```

`pipeline_asymmetric_v18.py` is the final strategy preserved from the challenge work.

## Technologies

- Python 3
- `dataclasses`
- `collections`
- `statistics.NormalDist`
- probability and stochastic-process modelling
- numerical integration
- ordinary least squares
- Bayesian updating
- Kelly criterion / expected-log optimisation

## Disclaimer

This repository is a personal project based on my participation in a trading challenge. It is provided for educational and portfolio purposes only. It is not financial advice and is not intended to be a production trading system.

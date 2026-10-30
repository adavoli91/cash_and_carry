# Cash&carry on inverse futures

## Introduction
This repository contains a study of the cash&carry (C&C) strategy on the inverse futures of Bybit, for BTC and ETH. The strategy consists in buying the cryptocurrency on the spot market and, at the same time, opening a short position of equal notional on an inverse futures, using the purchased cryptocurrency as collateral: the position is (approximately) neutral with respect to the price, and the profit comes from the premium of the futures over the spot.

Two variants are backtested:

- **dated C&C**: the short is opened on the quarterly futures; among the contracts with at least 30 days left to expiry, the one with the maximum annualized premium is chosen, and the position is held until expiry, when a new contract is opened;
- **perpetual C&C**: the short is opened on the inverse perpetual futures, and the profit comes from the *funding*; the funding obtained is reinvested, by increasing the short position as soon as it is large enough.

The notebook also combines the two variants in an equally weighted portfolio. In this document, we describe the mathematics of the perpetual C&C and we show its results.

## Repository structure
```
cash_and_carry/
├── data/                  # Bybit data (parquet), downloaded by the notebook and cached; not tracked by git
├── figures/               # figures produced by the notebook (PNG)
├── notebooks/
│   └── backtest.ipynb     # data download, backtest of the dated C&C, of the perpetual C&C and of the portfolio
├── .gitignore
└── README.md
```

The data are downloaded from the public API of Bybit (no API key is needed) only if the corresponding file is not present in `data/`; to update them, the files must be deleted.

## Theory

### Cash&carry on perpetual without reinvestment
In this section, we describe the mathematics behind a cash&carry operation done on the inverse perpetual futures, without reinvestment of the obtained *funding*.

At the initial time $`t_0`$, let us imagine having a capital $`C_\$`$ in USD; the quantity of cryptocurrency we can buy is:

```math
C_\text{฿} = \frac{C_\$}{p_0}\, (1 - \phi_s)\,,
```

where $`p_0`$ is the quote of the cryptocurrency in USD and $`\phi_s`$ is the fee rate for the spot.

The cash&carry strategy on perpetual futures consists in opening a short position using the purchased cryptocurrency as collateral, and collecting the *funding* which is periodically paid (in normal conditions, it is the short counterparty that collects it).

Denoting by $`k`$ the multiplier of the inverse futures[^1], by $`\ell`$ the leverage used and by $`p_f`$ the quote of the cryptocurrency at the time $`t_f`$ of opening of the position, the number of futures contracts on which we go short is:

```math
n_c = \left\lfloor\frac{\ell\, C_\text{฿}\, p_f}{k}\right\rfloor\,.
```

The opening of a position on the futures also involves the payment of a fee, given by:

```math
\mathrm{fee}_f = \frac{k\, n_c\, \phi_f}{p_f}\,,
```

where $`\phi_f`$ is the fee rate for the futures; therefore, the net value $`W_0`$ of the portfolio is:

```math
W_0 = C_\text{฿} - \mathrm{fee}_f\,.
```

Since the number $`n_c`$ of contracts is computed starting from $`C_\text{฿}`$, while the net value of the portfolio is $`W_0 < C_\text{฿}`$, the cash&carry is in fact slightly short with respect to the cryptocurrency, even with $`\ell = 1`$ (at the same time, there would also be a slight long effect deriving from the fact that $`n_c`$ is rounded down, thus leaving part of the cryptocurrency position partially unhedged; however, this effect is often negligible).

An open position on an inverse perpetual futures involves the payment of a *funding* which depends on the *funding rate* $`r(t)`$ and on the notional of the position, and is defined as:

```math
f(t) = \frac{k\, n_c\, r(t)}{p(t)}\,,
```

and is expressed in cryptocurrency. From this it follows that the USD value of the funding received at time $`t`$, at that given instant, is $`f_\$ = k\, n_c\, r(t)`$, i.e., it depends on time only through the funding rate. Overall, the balance of the portfolio, over time, is:

```math
W(t) = W_0 + \sum_{t_f < t'\leq t}f(t')\,.
```

Moreover, at a given instant $`t`$, the short position on the futures involves a P&L (expressed in cryptocurrency) given by:

```math
PL_f(t) = k\, n_c\, \left(\frac{1}{p(t)} - \frac{1}{p_f}\right)\,.
```

Overall, therefore, the equity in cryptocurrency as a function of time is:

```math
\tilde E_\text{฿}(t) = W(t) + PL_f(t) = W_0 + \sum_{t_f < t'\leq t}f(t') + k\, n_c\, \left(\frac{1}{p(t)} - \frac{1}{p_f}\right)\,.
```

To obtain the corresponding value in USD, it is sufficient to multiply the total quantity of cryptocurrency by the current quote, obtaining:

```math
\tilde E_\$(t) = \tilde E_\text{฿}(t)\,p(t) = W_0\, p(t) + k\, n_c\, p(t)\, \left[\sum_{t_f < t'\leq t}\frac{r(t')}{p(t')} + \frac{1}{p(t)} - \frac{1}{p_f}\right]\,.
```

To obtain the net equity, we must consider that, when closing the position, a double fee has to be paid: for the closing of the short on the futures, and for the sale of the resulting spot. The net equity is therefore:

```math
E_\$(t) = (1 - \phi_s)\, \left[\tilde E_\$(t) - k\, n_c\, \phi_f\right]\,.
```

It is important to note that, in this formulation, the quantity of cryptocurrency obtained from the funding is exposed to the price risk, since it is not hedged by the futures; in the following section, we see how things change with reinvestment.

### Cash&carry on perpetual with reinvestment
In case we want to keep hedging the cryptocurrency position, it is necessary to open further positions on the futures; this can be done at the moment when we come to hold a quantity of cryptocurrency sufficient to open a contract, i.e., when:

```math
\Delta n_c(t) \equiv \left\lfloor\frac{\ell\, p(t)\, F(t)}{k}\right\rfloor \geq 1\,,
```

where $`F(t)`$ is the quantity of cryptocurrency deriving from the funding not yet hedged by the futures:

```math
F(t) = \sum_{t_f < t'\leq t} f(t') - \sum_{t'\in T_c^{(t)}}\frac{k\, \Delta n_c(t')}{\ell\, p(t')}\,,
```

with $`T_c^{(t)}`$ the set of the rehedge instants preceding $`t`$. The second term represents the cryptocurrency hedged by the rehedges already done: since $`\Delta n_c`$ is rounded down, at each rehedge an unhedged residual of funding remains, which is carried forward and contributes to the following rehedge. When the condition on $`\Delta n_c(t)`$ is satisfied, it is possible to increase the short position on the futures by $`\Delta n_c(t)`$[^2].

The new average entry price of the position can be computed taking into account that the P&L of the futures (in cryptocurrency, since this is the reference currency for the futures) at any instant $`\bar t`$ must coincide with the sum of the P&Ls deriving from the single positions; defining $`p_c`$ the rehedge price and $`p_m`$ the average entry price, we obtain:

```math
k\, (n_c + \Delta n_c)\,\left(\frac{1}{\bar p} - \frac{1}{p_m}\right) = k\, n_c\,\left(\frac{1}{\bar p} - \frac{1}{p_f}\right) + k\, \Delta n_c\,\left(\frac{1}{\bar p} - \frac{1}{p_c}\right)\,,
```

from which:

```math
p_m = \frac{n_c + \Delta n_c}{\dfrac{n_c}{p_f} + \dfrac{\Delta n_c}{p_c}}\,.
```

Therefore, the average entry price is the (weighted) harmonic mean between the rehedge price and $`p_f`$. For subsequent rehedges, the starting configuration will no longer be $`(n_c,\, p_f)`$ but $`(n_c(t),\, p_m)`$.

Apart from these modifications, the computations of the previous section remain valid, with a couple of clarifications:

- the funding must be computed starting from the number of futures contracts, which varies over time:

  ```math
  f(t) = \frac{k\, n_c(t)\, r(t)}{p(t)}\,;
  ```

- the rehedge leads to a decrease of the balance of the portfolio:

  ```math
  W(t) = W_0 + \sum_{t_f < t'\leq t}f(t') - k\, \phi_f\,\sum_{t'\in T_c^{(t)}}\frac{\Delta n_c(t')}{p(t')}\,;
  ```

- the P&L deriving from the futures is:

  ```math
  PL_f(t) = k\, n_c(t)\,\left(\frac{1}{p(t)} - \frac{1}{p_m(t)}\right)\,;
  ```

- the equity in USD $`\tilde E_\$(t)`$ becomes:

  ```math
  \begin{aligned}
  \tilde E_\$(t) &= \left[W(t) + PL_f(t)\right]\, p(t)\\[.2cm]
  &= \left[W_0 - k\, \phi_f\,\sum_{t'\in T_c^{(t)}}\frac{\Delta n_c(t')}{p(t')}\right]\, p(t) + p(t)\,\sum_{t_f < t'\leq t}f(t') \\[.2cm]
  &\quad + k\, n_c(t)\, p(t)\, \left[\frac{1}{p(t)} - \frac{1}{p_m(t)}\right]\,;
  \end{aligned}
  ```

- finally, the net equity is obtained as:

  ```math
  E_\$(t) = (1 - \phi_s)\, \left[\tilde E_\$(t) - k\, n_c(t)\, \phi_f\right]\,.
  ```

## Results of the perpetual cash&carry

### Data and setup
The backtest uses the following data, downloaded from Bybit:

- the hourly index price of `BTCUSD` and `ETHUSD`, used as spot price $`p(t)`$;
- the history of the funding rate $`r(t)`$ of the inverse perpetuals `BTCUSD` and `ETHUSD`, which is paid every 8 hours.

The spot is resampled on 8-hour bars, aligned with the funding times. The operation starts on 2021-07-01 and ends on 2026-10-14 (last available data), i.e., it covers about 5.3 years and 5793 funding payments for each cryptocurrency. The parameters of the backtest are:

| Parameter | Symbol | Value |
|---|---|---|
| Initial capital | $`C_\$`$ | 1000 USD |
| Multiplier of the inverse futures | $`k`$ | 1 USD |
| Leverage | $`\ell`$ | 1 |
| Fee rate for the spot | $`\phi_s`$ | 0.1% |
| Fee rate for the futures | $`\phi_f`$ | 0.055% |
| Minimum rehedge value | | 1% of $`C_\$`$ |

The last parameter is the threshold discussed in footnote 2: a rehedge is done only when $`k\,\Delta n_c(t) \geq 10`$ USD. The equity shown below is always the net equity $`E_\$(t)`$ defined in the previous section, i.e., the value which would be obtained by closing the whole position at time $`t`$. The performance is measured by:

- the compound annual growth rate (CAGR);
- the drawdown (DD), computed on the 8-hour equity and then taking the daily minimum; the average DD is the mean over the days in which the DD is non-zero;
- the annualized volatility of the daily returns of the equity;
- the correlation between the daily returns of the equity and those of the spot.

### Equity and drawdown
The equity and the drawdown of the perpetual C&C are shown in Figure 1 for BTC and in Figure 2 for ETH, while the main metrics are summarized in Table 1.

![Perpetual C&C on BTC](figures/perpetual_return_BTC.png)

**Figure 1.** Net return (top) and drawdown (bottom) of the perpetual C&C on BTC.

![Perpetual C&C on ETH](figures/perpetual_return_ETH.png)

**Figure 2.** Net return (top) and drawdown (bottom) of the perpetual C&C on ETH.

**Table 1.** Performance of the perpetual C&C, compared with the spot over the same period.

| Metric | BTC | ETH |
|---|---|---|
| Total return | 37.37% | 37.90% |
| CAGR | 6.18% | 6.26% |
| Max DD | −2.12% | −3.02% |
| Average DD | −0.29% | −0.58% |
| CAGR / (average DD) | 21.00 | 10.85 |
| Annualized volatility | 0.65% | 0.86% |
| Correlation with the spot | 0.07 | 0.03 |
| Number of rehedges | 37 | 38 |
| Total return of the spot | 137.1% | 9.6% |
| Max DD of the spot | −77.0% | −80.3% |

The two cryptocurrencies give very similar results, with a CAGR of about 6.2% per year, although the spot had a completely different behaviour in the two cases (+137% for BTC and +10% for ETH, with drawdowns of about −80% for both). The correlation with the spot is negligible, which confirms that the position is neutral with respect to the price: the equity grows almost monotonically, with a volatility below 1% per year.

### Dependence on the funding regime
Since the price risk is hedged, the return depends almost entirely on the funding rate. Table 2 shows, for each year, the return of the strategy and the mean funding rate, annualized ($`3 \times 365`$ payments per year).

**Table 2.** Yearly return of the perpetual C&C and annualized mean funding rate. 2021 starts on 2021-07-01, and 2026 ends on 2026-10-14.

| Year | Return BTC | Funding BTC | Return ETH | Funding ETH |
|---|---|---|---|---|
| 2021 (partial) | 4.15% | 8.52% | 6.94% | 13.41% |
| 2022 | −0.95% | −0.35% | −0.03% | −0.12% |
| 2023 | 7.92% | 7.87% | 6.84% | 7.40% |
| 2024 | 13.80% | 12.72% | 13.81% | 12.76% |
| 2025 | 6.05% | 5.90% | 4.76% | 4.98% |
| 2026 (partial) | 2.58% | 3.07% | 1.56% | 1.99% |

The yearly return follows closely the annualized funding rate. The median funding rate is 10.95% per year for both cryptocurrencies, which corresponds to 0.01% every 8 hours, i.e., the base value of the funding on Bybit; however, the funding is negative in about 21% of the payments, which lowers the mean to about 6.2–6.3% per year, in line with the CAGR.

The only losing year is 2022, when the funding was on average slightly negative. The maximum drawdown occurs at the end of 2022, during the market stress following the collapse of FTX: for BTC, the equity went down by 2.12% between 2022-11-07 and 2022-12-19, while the sum of the funding rates over the same period was about −1.97%, which corresponds to about −20 USD out of a drop of about 22 USD; for ETH, the drawdown of 3.02% (2022-08-19 to 2022-12-13) corresponds to a sum of the funding rates of about −3.17%. The drawdowns are therefore due to the periods of negative funding, and not to the price movements. As a consequence, the return over the last year (red curve in Figure 3 for BTC and in Figure 4 for ETH) goes slightly below zero only at the beginning of 2023 (minimum of −1.06% for BTC and −0.45% for ETH), and reaches its maximum of about 14% at the end of 2024; at the end of the backtest, it is about 3.4% for BTC and 2.4% for ETH.

![Return over the last year on BTC](figures/rolling_return_BTC.png)

**Figure 3.** Return over the last year on BTC. The perpetual C&C is the red curve; the dated C&C (blue) and the portfolio (green) are shown for comparison.

![Return over the last year on ETH](figures/rolling_return_ETH.png)

**Figure 4.** Return over the last year on ETH. The perpetual C&C is the red curve; the dated C&C (blue) and the portfolio (green) are shown for comparison.

### Conclusions
On the period 2021-07 to 2026-10, the perpetual C&C with reinvestment of the funding obtained a CAGR of about 6.2% per year on both BTC and ETH, with a maximum drawdown of 2–3% and a negligible correlation with the spot. The return is essentially the funding rate, so it is not constant over time: it was close to zero in 2022, about 13–14% in 2024, and has decreased to about 2–3% in the last year.

The backtest has some limitations:

- the P&L of the futures is computed using the index price, so the basis between the perpetual and the index is neglected;
- the slippage and the margin requirements are not modelled;
- a single set of parameters is tested (in particular, $`\ell = 1`$ and a rehedge threshold of 1% of the capital).

[^1]: The multiplier $`k`$ is defined in such a way that an inverse futures contract is worth $`k`$ \$, regardless of the quote.

[^2]: Sometimes, however, it can be preferable to wait until the position to be possibly opened corresponds, in USD, at least to a certain fraction of the initial capital, to limit the impact of fees.

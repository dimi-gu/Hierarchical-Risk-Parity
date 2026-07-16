# Hierarchical-Risk-Parity

A Python/Jupyter implementation of **Hierarchical Risk Parity (HRP)** portfolio with rolling out-of-sample backtesting, transaction-cost modelling, portfolio diagnostics, and comparison against traditional allocation strategies.

The notebook implements the HRP algorithm portfolio introduced by Marcos López de Prado and compares it with following portfolios:

- **1/N** equal weighting
- **Minimum Global Variance (MGV)**
- **Maximum Sharpe**

It also produces visual diagnostics for asset clustering, portfolio composition, performance, and trading costs.

---

## Main Features

- Rolling in-sample optimisation and out-of-sample backtesting
- Hierarchical clustering based on asset correlations
- HRP quasi-diagonalisation and recursive bisection
- Long-only MGV and Maximum Sharpe optimisation
- Equal-weight benchmark
- Configurable rebalancing frequency
- Optional inclusion of government bonds
- Separate transaction-cost assumptions for:
  - positions entering or leaving the portfolio
  - changes to positions already held
- Gross and net-of-cost wealth comparison
- Portfolio performance and risk metrics
- Asset-group allocation analysis
- Dendrogram, correlation heatmaps, and minimum spanning tree
- Portfolio-weight evolution through time

---

## Methodology

### 1. Rolling backtest

The input data are assumed to be weekly.

For every rebalancing date, the notebook:

1. selects a rolling historical window;
2. estimates the required return, covariance, and correlation inputs;
3. calculates portfolio weights;
4. applies those weights to the following out-of-sample observations;
5. moves the estimation window forward by the selected rebalancing interval.

The optimisation-window length and rebalancing frequency are user-configurable.

### 2. Hierarchical Risk Parity

The HRP allocation follows three main steps.

#### Tree clustering

The correlation matrix is converted into a distance matrix:

$$\ d_{i,j} = \sqrt{\frac{1-\rho_{i,j}}{2}} \$$

Assets are then clustered using single-linkage hierarchical clustering.

#### Quasi-diagonalisation

The assets are reordered so that strongly related assets are placed close together in the covariance matrix.

#### Recursive bisection

The ordered asset set is recursively divided into two clusters. Capital is allocated between each pair according to inverse cluster variance:

$$ \
\alpha = 1-\frac{\sigma_0^2}{\sigma_0^2+\sigma_1^2}
\ $$

where $\\sigma_0^2\$ and $\\sigma_1^2\$ are the inverse-variance portfolio variances of the two clusters.

### 3. Benchmark portfolios

The notebook calculates three alternative strategies:

- **1/N:** the same weight is assigned to every asset;
- **MGV:** long-only portfolio with minimum estimated variance;
- **Maximum Sharpe:** long-only portfolio maximising the estimated Sharpe ratio, using a risk-free rate of zero.

MGV and Maximum Sharpe are implemented with `PyPortfolioOpt`.

### 4. Transaction costs

The cost model distinguishes between two types of turnover:

- `c1`: cost applied when a position is initiated or fully removed;
- `c2`: cost applied when an existing position is resized.

The initial portfolio allocation is treated as a complete purchase and is charged at `c1`.

At each rebalancing date, costs are approximated from absolute changes in portfolio weights multiplied by current portfolio value. Net value is calculated by compounding portfolio returns and subtracting the corresponding rebalancing costs.

---

## Input Data

Place an Excel file named **`HRP.xlsx`** in the same directory as the notebook.

The Excel file contain two sheets.

### `Returns`

Weekly asset returns:

| Dates | Asset 1 | Asset 2 | ... |
|---|---:|---:|---:|
| 2015-01-02 | 0.012 | -0.004 | ... |
| 2015-01-09 | -0.006 | 0.009 | ... |

### `Prices`

Weekly asset prices:

| Dates | Asset 1 | Asset 2 | ... |
|---|---:|---:|---:|
| 2015-01-02 | 101.25 | 98.40 | ... |
| 2015-01-09 | 100.64 | 99.29 | ... |

File description:

- both sheets contain a `Dates` column;
- observations are chronologically ordered;
- the data frequency is weekly;

The notebook recognises the following bond columns:

- `Eurozone Bonds`
- `US Treasury`

These can be included or excluded through the `include_bonds` parameter.

The visualisations also classify assets into groups such as precious metals, commodities, US equities, Asian equities, European equities, real estate, and bonds. New asset names must be added to the `ASSET_GROUPS` mapping before the grouped charts can be generated.

---

## Configuration

The main parameters are defined near the beginning of the notebook:

```python
Years_optimization = 3
rebalance = 8
Wlt = 100000
c1 = 0.02
c2 = 0.005
include_bonds = 0
```

| Parameter | Description |
|---|---|
| `Years_optimization` | Length of the rolling estimation window in years |
| `rebalance` | Number of weeks between portfolio rebalances |
| `Wlt` | Initial wealth used in transaction-cost calculations |
| `c1` | Cost rate for entering or completely exiting a position |
| `c2` | Cost rate for resizing an existing position |
| `include_bonds` | `1` includes bond assets; `0` excludes them |

Because the split point is identified from weekly row numbers, use optimisation periods compatible with the weekly grid. Quarter-year increments such as `0.25`, `0.50`, `1.00`, and `3.25` are suitable for the current implementation.

---

## Usage

1. Clone or download the repository.
2. Place `HRP.xlsx` beside the notebook.
3. Open the notebook in Jupyter.
4. Review the asset groups and user parameters.
5. Run all cells in order.

---

## Outputs

The notebook produces the following analyses and figures:

1. HRP allocation by asset group at inception
2. Detailed initial allocation table
3. Hierarchical-clustering dendrogram
4. Original and quasi-diagonalised correlation matrices
5. Minimum spanning tree
6. HRP cumulative out-of-sample performance
7. HRP gross wealth versus net wealth
8. Cumulative HRP transaction costs
9. Performance comparison across HRP, 1/N, MGV, and Maximum Sharpe
10. Initial portfolio weights by strategy
11. Performance-metrics table
12. Cumulative transaction costs by strategy
13. Gross and net wealth for HRP, MGV, and Maximum Sharpe
14. Asset-group weights through time for HRP, MGV, and Maximum Sharpe

The reported metrics include:

- total return;
- geometric annualised return;
- annualised volatility;
- Sharpe ratio;
- worst weekly return;
- initial in-sample annualised volatility.

Annualisation assumes 52 weekly observations per year.

---

## Important Implementation Notes

### SciPy clustering input

The current notebook passes the full square distance matrix directly to:

```python
sch.linkage(dist, method="single")
```

This triggers SciPy's `ClusterWarning`, because `linkage` normally expects either an observation matrix or a condensed distance vector.

For a conventional precomputed-distance implementation, convert the matrix before clustering:

```python
from scipy.spatial.distance import squareform

condensed_dist = squareform(dist.to_numpy(), checks=False)
link = sch.linkage(condensed_dist, method="single")
```

This change may alter the clustering tree, HRP weights, and backtest results.

The linkage method alone is the key parameter of the HRP portfolio performance as it implicitly determines the re-ordered covariance matrix, the cluster's composition and the inverse-variance allocation. The different performance is due to the HRP's recursive bisection that identifies clusters by position and not by the structure of the dendrogram. User can verify this behaviour by selecting, for example, the "ward" linkage (ceil 11 of the notebook).

```python
sch.linkage(dist, method="ward")
```

### Other assumptions

- Portfolios are long-only.
- The Maximum Sharpe calculation assumes a zero risk-free rate.
- Expected returns and covariance matrices are estimated from historical samples.
- Transaction costs are simplified proportional costs.
- Taxes, bid-ask spread dynamics, market impact, liquidity constraints, and execution delays are not modelled.
- Transaction costs are not currently presented for the 1/N strategy.
- Results are sensitive to the optimisation window, rebalancing frequency, linkage method, asset universe, and cost assumptions.

---

## Reference

Marcos López de Prado, *Building Diversified Portfolios that Outperform Out-of-Sample*:

https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2708678

---

## Disclaimer

This project was the subject of my master thesis in "Finance and Risk Management" (Università degli studi di Firenze) and was intended for research and educational purposes only. It does not constitute investment advice, and historical backtest results do not guarantee future performance.

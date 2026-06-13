# Reverse Engineering the Crypto Market

**Author:** Betha Morrison
**Level:** Intermediate–Advanced
**Scope:** A complete research pipeline — data collection, feature engineering, regime detection, signal generation, backtesting, validation, and live deployment — with working code and documented limitations.

---

## Table of Contents

1. [Scope and Definitions](#1-scope-and-definitions)
2. [Statistical Properties of Crypto Markets](#2-statistical-properties-of-crypto-markets)
3. [Pipeline Overview](#3-pipeline-overview)
4. [Data Collection](#4-data-collection)
5. [Feature Engineering](#5-feature-engineering)
6. [Regime Detection with Hidden Markov Models](#6-regime-detection-with-hidden-markov-models)
7. [Signal Generation](#7-signal-generation)
8. [Backtesting](#8-backtesting)
9. [Evaluation Metrics](#9-evaluation-metrics)
10. [Walk-Forward Validation and Robustness Checks](#10-walk-forward-validation-and-robustness-checks)
11. [Deployment](#11-deployment)
12. [Limitations](#12-limitations)
13. [References and Tools](#13-references-and-tools)

---

## 1. Scope and Definitions

This article does not attempt to predict prices. Reliable point prediction of crypto prices is not achievable with the methods described here. The objective is narrower and measurable:

> Identify repeating statistical patterns in market data and build a system that exploits them with a positive expected value after costs.

The analogy is structural cryptanalysis: rather than recovering a key directly, the analyst studies distributional structure to locate exploitable weaknesses. Applied to markets, this means modeling return distributions, regime persistence, and cross-feature relationships rather than forecasting individual candles.

Crypto markets offer practical advantages for this kind of research:

* Tick-level and OHLCV historical data are freely available from major exchanges.
* On-chain data is publicly verifiable.
* The market is relatively young, with more documented inefficiencies than mature equity markets.
* Exchange APIs are straightforward to integrate for live execution.

These advantages do not remove risk. The remaining sections quantify the constraints and the realistic size of any edge.

---

## 2. Statistical Properties of Crypto Markets

The modeling choices in later sections depend on the statistical properties established here.

### 2.1 Return Distribution

Crypto returns are non-Gaussian. Three properties are consistently observed:

* **Fat tails** (excess kurtosis > 0): extreme moves occur more frequently than a normal distribution predicts.
* **Negative skewness** on daily timeframes: drawdowns tend to be sharper than rallies.
* **Volatility clustering**: high-volatility periods follow high-volatility periods (ARCH effects).

```python
import numpy as np
import pandas as pd
from scipy import stats
import ccxt

exchange = ccxt.binance()
ohlcv = exchange.fetch_ohlcv('BTC/USDT', timeframe='1d', limit=500)
df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
df['returns'] = df['close'].pct_change()

returns = df['returns'].dropna()

# scipy.stats.kurtosis returns EXCESS kurtosis (normal = 0)
print(f"Excess kurtosis: {stats.kurtosis(returns):.2f}")
print(f"Skewness:        {stats.skew(returns):.2f}")

# Jarque-Bera: p < 0.05 rejects normality
jb_stat, jb_p = stats.jarque_bera(returns)
print(f"Jarque-Bera p-value: {jb_p:.6f}")
```

Strategies that assume normality, such as standard Bollinger Bands sized on a normal distribution, systematically underestimate tail risk on this data.

### 2.2 Stationarity and Autocorrelation

```python
from statsmodels.tsa.stattools import adfuller, acf

# ADF on price: typically non-stationary (p > 0.05)
print(f"ADF p-value (price):   {adfuller(df['close'].dropna())[1]:.4f}")

# ADF on returns: typically stationary (p < 0.05)
print(f"ADF p-value (returns): {adfuller(returns)[1]:.4f}")

# Return autocorrelation is typically near zero,
# so direct prediction from lagged returns alone is weak
acf_vals = acf(returns, nlags=10)
print("ACF returns (lag 1-5):", acf_vals[1:6])
```

Near-zero return autocorrelation is the reason this pipeline relies on regime structure and cross-features rather than direct return prediction.

### 2.3 Hurst Exponent

The Hurst exponent (H) characterizes long-range dependence:

```
H > 0.5  -> persistent (trending)
H = 0.5  -> random walk
H < 0.5  -> anti-persistent (mean-reverting)
```

The estimator below uses the structure-function (generalized variance) method, not classical rescaled-range (R/S) analysis. It is faster and adequate for a directional read, but it is sensitive to `max_lag` and is a rough estimate rather than a precise measurement.

```python
def hurst_exponent(ts, max_lag=20):
    """Estimate the Hurst exponent via the structure-function method.

    Note: this is not classical R/S analysis. The slope of
    log(std of lagged differences) vs log(lag) estimates H.
    """
    ts = np.asarray(ts, dtype=float)
    lags = range(2, max_lag)
    tau = [np.std(ts[lag:] - ts[:-lag]) for lag in lags]
    slope = np.polyfit(np.log(lags), np.log(tau), 1)[0]
    return slope

H = hurst_exponent(df['close'].dropna().values)
print(f"Hurst exponent (BTC daily close): {H:.4f}")
```

Daily BTC close commonly estimates around H ≈ 0.55–0.65 (mildly persistent), but the value shifts with the sample window and the prevailing regime, so it is treated as a descriptive statistic rather than a fixed property.

---

## 3. Pipeline Overview

```
Raw Data (OHLCV + On-Chain)
        |
        v
Feature Engineering
(returns, volatility, on-chain metrics)
        |
        v
Regime Detection (HMM)
        |
        +--> Regime = Bull  -> Trend Following
        +--> Regime = Bear  -> Short / Hedge
        +--> Regime = Chop  -> Mean Reversion
                 |
                 v
        Signal Generation (entry/exit rules)
                 |
                 v
        Position Sizing (Kelly, fixed fractional)
                 |
                 v
        Backtesting (vectorized / vectorbt)
                 |
                 v
        Walk-Forward Validation
                 |
                 v
        Performance Evaluation
        (Sharpe, Sortino, Calmar, MDD)
```

A design constraint applied throughout: the feature set used for regime detection is fixed in one place and reused at fit time, in walk-forward validation, and in live inference. A mismatch between the features used to fit the scaler/model and the features supplied at inference is a common and silent source of failure, so it is centralized:

```python
# Single source of truth for regime features.
# The scaler and HMM are fit on exactly these columns,
# and inference must supply exactly these columns in this order.
REGIME_FEATURES = ['log_returns', 'volatility_7d', 'vol_ratio']
```

---

## 4. Data Collection

### 4.1 Price Data via CCXT

```python
import ccxt
import pandas as pd
import time

def fetch_full_ohlcv(symbol='BTC/USDT', timeframe='1h', since_days=365):
    """Fetch paginated OHLCV history from Binance."""
    exchange = ccxt.binance({'enableRateLimit': True})
    since = exchange.parse8601(
        (pd.Timestamp.utcnow() - pd.Timedelta(days=since_days)).strftime('%Y-%m-%dT%H:%M:%SZ')
    )

    all_ohlcv = []
    while True:
        ohlcv = exchange.fetch_ohlcv(symbol, timeframe, since=since, limit=1000)
        if not ohlcv:
            break
        all_ohlcv.extend(ohlcv)
        since = ohlcv[-1][0] + 1
        time.sleep(exchange.rateLimit / 1000)
        if ohlcv[-1][0] >= exchange.milliseconds():
            break

    df = pd.DataFrame(all_ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    df['datetime'] = pd.to_datetime(df['timestamp'], unit='ms')
    df.set_index('datetime', inplace=True)
    df = df[~df.index.duplicated(keep='first')]  # pagination can yield overlaps
    return df

df = fetch_full_ohlcv('BTC/USDT', '1d', since_days=1000)
```

### 4.2 On-Chain Data via Glassnode

On-chain metrics are optional in this pipeline. Regime detection uses only price-derived features (Section 3), while on-chain data feeds a separate divergence signal (Section 7.3). Treat the free-tier availability and exact metric paths as subject to change, and verify them against current Glassnode documentation.

```python
import requests

GLASSNODE_API_KEY = "your_api_key"

def get_glassnode_metric(metric, asset='BTC', resolution='24h'):
    """Fetch a single on-chain metric as a pandas Series.

    Example metric paths (verify current availability and tier):
      indicators/sopr  -> Spent Output Profit Ratio
      market/mvrv      -> Market Value to Realized Value
      market/nupl      -> Net Unrealized Profit/Loss
      distribution/exchange_net_position_change -> Exchange flow
    """
    url = f"https://api.glassnode.com/v1/metrics/{metric}"
    params = {'a': asset, 'i': resolution, 'api_key': GLASSNODE_API_KEY}
    response = requests.get(url, params=params, timeout=30)
    response.raise_for_status()

    data = pd.DataFrame(response.json())
    data['datetime'] = pd.to_datetime(data['t'], unit='s')
    data.set_index('datetime', inplace=True)
    return data['v'].rename(metric)

sopr = get_glassnode_metric('indicators/sopr')
mvrv = get_glassnode_metric('market/mvrv')
exchange_flow = get_glassnode_metric('distribution/exchange_net_position_change')
```

### 4.3 Merge and Alignment

```python
df_merged = df[['high', 'low', 'close', 'volume']].copy()
df_merged['sopr'] = sopr
df_merged['mvrv'] = mvrv
df_merged['exchange_flow'] = exchange_flow

# On-chain series can have gaps relative to the price index.
# Forward-fill only carries the last known value forward (no future leakage),
# then drop any remaining leading NaNs.
df_merged = df_merged.ffill().dropna()
```

Forward-fill uses only past observations, so it does not introduce look-ahead bias. The `high` and `low` columns are retained because the ATR feature requires them.

---

## 5. Feature Engineering

### 5.1 Price-Based Features

```python
def compute_price_features(df):
    df = df.copy()

    # Returns
    df['returns_1d'] = df['close'].pct_change()
    df['returns_7d'] = df['close'].pct_change(7)
    df['log_returns'] = np.log(df['close'] / df['close'].shift(1))

    # Realized volatility (annualized; 365 trading days for crypto)
    df['volatility_7d'] = df['log_returns'].rolling(7).std() * np.sqrt(365)
    df['volatility_30d'] = df['log_returns'].rolling(30).std() * np.sqrt(365)
    df['vol_ratio'] = df['volatility_7d'] / df['volatility_30d']

    # Momentum
    df['roc_14'] = df['close'].pct_change(14)
    df['roc_30'] = df['close'].pct_change(30)

    # Trend reference
    df['sma_50'] = df['close'].rolling(50).mean()
    df['sma_200'] = df['close'].rolling(200).mean()
    df['price_vs_sma200'] = (df['close'] - df['sma_200']) / df['sma_200']

    # Normalized ATR (computed in a local frame to avoid leaving helper columns)
    tr = pd.concat([
        df['high'] - df['low'],
        (df['high'] - df['close'].shift(1)).abs(),
        (df['low'] - df['close'].shift(1)).abs(),
    ], axis=1).max(axis=1)
    df['atr_14'] = tr.rolling(14).mean()
    df['natr'] = df['atr_14'] / df['close']

    return df
```

### 5.2 On-Chain Features

```python
def compute_onchain_features(df):
    df = df.copy()

    # MVRV Z-score, normalized on a rolling one-year window (past data only)
    mvrv_mean = df['mvrv'].rolling(365).mean()
    mvrv_std = df['mvrv'].rolling(365).std()
    df['mvrv_zscore'] = (df['mvrv'] - mvrv_mean) / mvrv_std

    # Smoothed SOPR to reduce day-to-day noise
    df['sopr_7d_ma'] = df['sopr'].rolling(7).mean()

    # Net exchange flow momentum
    df['flow_momentum'] = df['exchange_flow'].rolling(7).sum()

    return df

df_feat = compute_price_features(df_merged)
df_feat = compute_onchain_features(df_feat)
df_feat.dropna(inplace=True)
```

All rolling windows reference only past observations. Normalization is computed on rolling windows rather than the full sample, which prevents the data leakage discussed in Section 8.3.

---

## 6. Regime Detection with Hidden Markov Models

Crypto markets cycle between trending, ranging, and declining phases. A Gaussian HMM assigns each observation a regime probabilistically, without manually chosen thresholds.

Model components:

```
State sequence:    S_1 -> S_2 -> ... -> S_t
Observations:      O_1,   O_2,  ...,  O_t

Transition matrix A:  P(S_t | S_{t-1})
Emission matrix B:    P(O_t | S_t)   [Gaussian here]
```

The implementation below fixes two correctness issues that are easy to introduce:

1. The model is fit on a single cleaned frame, and labels are aligned to that frame's index. Computing the regime label statistics against a differently filtered series (for example `df['log_returns'].dropna()`) causes a length and alignment mismatch whenever feature columns and the return column have different NaN patterns.
2. The fitted `scaler`, `model`, and `state_map` are returned together so inference reuses the exact transformation applied at training time.

```python
from hmmlearn import hmm
from sklearn.preprocessing import StandardScaler
import warnings
warnings.filterwarnings('ignore')

def fit_hmm_regimes(df, n_regimes=3, features=REGIME_FEATURES, verbose=True):
    """Fit a Gaussian HMM and return regimes aligned to the input index.

    Returns:
        regimes : pd.Series of labeled states (0=Bear, 1=Chop, 2=Bull)
                  indexed to the rows actually used (after dropna on features)
        model, scaler, state_map : reusable for out-of-sample inference
    """
    clean = df.dropna(subset=features)
    X = clean[features].values

    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)

    model = hmm.GaussianHMM(
        n_components=n_regimes,
        covariance_type='full',
        n_iter=1000,
        random_state=42,
    )
    model.fit(X_scaled)
    raw_states = model.predict(X_scaled)

    # Rank raw states by mean log-return, using the SAME aligned frame
    aligned_returns = clean['log_returns'].values
    stats_by_state = {}
    for s in range(n_regimes):
        mask = raw_states == s
        stats_by_state[s] = {
            'mean_return': aligned_returns[mask].mean(),
            'mean_vol': clean['volatility_7d'].values[mask].mean(),
            'count': int(mask.sum()),
        }

    ordered = sorted(stats_by_state, key=lambda s: stats_by_state[s]['mean_return'])
    state_map = {raw: rank for rank, raw in enumerate(ordered)}
    labeled = np.array([state_map[s] for s in raw_states])

    labels = {0: 'Bear', 1: 'Chop', 2: 'Bull'}
    if verbose:
        print("\n=== Regime Statistics ===")
        for rank, name in labels.items():
            st = stats_by_state[ordered[rank]]
            print(f"{name}: mean_return={st['mean_return']:.4f}, "
                  f"mean_vol={st['mean_vol']:.4f}, count={st['count']} days")
        print("\nTransition Matrix:")
        tm = model.transmat_
        for i in range(n_regimes):
            row = [f"->{labels[j]}={tm[ordered[i]][ordered[j]]:.2f}" for j in range(n_regimes)]
            print(f"  From {labels[i]}: {row}")

    return pd.Series(labeled, index=clean.index, name='regime'), model, scaler, state_map


def predict_regimes(df, model, scaler, state_map, features=REGIME_FEATURES):
    """Apply a fitted HMM to new data, returning regimes aligned to the index."""
    clean = df.dropna(subset=features)
    X_scaled = scaler.transform(clean[features].values)
    raw = model.predict(X_scaled)
    labeled = np.array([state_map.get(s, 1) for s in raw])  # default to Chop if unseen
    return pd.Series(labeled, index=clean.index, name='regime')


regimes, hmm_model, hmm_scaler, hmm_state_map = fit_hmm_regimes(df_feat)
df_feat = df_feat.join(regimes, how='inner')
```

Example output on BTC daily data (2019–2024):

```
=== Regime Statistics ===
Bear: mean_return=-0.0082, mean_vol=0.89, count=298 days
Chop: mean_return=+0.0012, mean_vol=0.54, count=412 days
Bull: mean_return=+0.0094, mean_vol=0.73, count=290 days

Transition Matrix:
  From Bear: ['->Bear=0.91', '->Chop=0.07', '->Bull=0.02']
  From Chop: ['->Bear=0.05', '->Chop=0.88', '->Bull=0.07']
  From Bull: ['->Bear=0.03', '->Chop=0.08', '->Bull=0.89']
```

High diagonal probabilities indicate persistent regimes. Persistence is the property that makes regime-conditioned strategies viable: a state detected today is likely to hold over the next several observations.

---

## 7. Signal Generation

Each sub-strategy is conditioned on the regime so that it is active only where its logic applies.

### 7.1 Trend Following (Bull and Bear regimes)

```python
def trend_signal(df):
    """SMA crossover with a regime filter.
    Long in Bull regime, short in Bear regime, flat otherwise.
    """
    df = df.copy()
    sma_fast = df['close'].rolling(20).mean()
    sma_slow = df['close'].rolling(50).mean()

    signal = pd.Series(0, index=df.index)
    signal[(sma_fast > sma_slow) & (df['regime'] == 2)] = 1
    signal[(sma_fast < sma_slow) & (df['regime'] == 0)] = -1
    return signal
```

### 7.2 Mean Reversion (Chop regime)

```
z_t = (price_t - mean_window) / std_window

Long  -> z < -1.5 and regime == Chop
Short -> z > +1.5 and regime == Chop
Exit  -> |z| < 0.5
```

```python
def mean_reversion_signal(df, window=20, entry_z=1.5, exit_z=0.5):
    """Z-score mean reversion, active only in the Chop regime."""
    df = df.copy()
    rolling_mean = df['close'].rolling(window).mean()
    rolling_std = df['close'].rolling(window).std()
    zscore = (df['close'] - rolling_mean) / rolling_std

    signal = pd.Series(0, index=df.index)
    signal[(zscore < -entry_z) & (df['regime'] == 1)] = 1
    signal[(zscore > entry_z) & (df['regime'] == 1)] = -1

    # Flatten when price returns toward the mean
    held = signal.replace(0, np.nan).ffill()
    signal[(zscore.abs() < exit_z) & (held != 0)] = 0
    return signal
```

### 7.3 On-Chain Divergence

This signal requires the `sopr` and `mvrv` columns. It returns all zeros when they are absent, so the composite remains well defined on price-only data.

```python
def onchain_divergence_signal(df):
    """Divergence between price and on-chain profitability.

    Bullish: price falling, SOPR recovering, MVRV < 1.0 (capitulation zone)
    Bearish: price rising, SOPR falling, MVRV > 2.5 (distribution zone)

    Returns zeros if on-chain columns are not present.
    """
    df = df.copy()
    if not {'sopr', 'mvrv'}.issubset(df.columns):
        return pd.Series(0, index=df.index)

    price_trend = np.sign(df['returns_7d'])
    sopr_trend = np.sign(df['sopr'].pct_change(7))

    signal = pd.Series(0, index=df.index)
    signal[(price_trend < 0) & (sopr_trend > 0) & (df['mvrv'] < 1.0)] = 1
    signal[(price_trend > 0) & (sopr_trend < 0) & (df['mvrv'] > 2.5)] = -1
    return signal
```

### 7.4 Composite Signal

```python
def composite_signal(df, weights=None):
    """Weighted combination of the sub-signals, thresholded to a discrete position."""
    if weights is None:
        weights = {'trend': 0.4, 'mr': 0.3, 'onchain': 0.3}
    df = df.copy()

    df['trend_signal'] = trend_signal(df)
    df['mr_signal'] = mean_reversion_signal(df)
    df['onchain_signal'] = onchain_divergence_signal(df)

    df['composite'] = (
        df['trend_signal'] * weights['trend']
        + df['mr_signal'] * weights['mr']
        + df['onchain_signal'] * weights['onchain']
    )

    df['final_signal'] = 0
    df.loc[df['composite'] > 0.25, 'final_signal'] = 1
    df.loc[df['composite'] < -0.25, 'final_signal'] = -1
    return df
```

---

## 8. Backtesting

### 8.1 Vectorized Backtest

```python
def vectorized_backtest(df, signal_col='final_signal',
                        fee=0.001, slippage=0.0005, initial_capital=10000.0):
    """Vectorized backtest with next-bar execution and per-change costs.

    The signal is shifted by one bar so that a signal formed on the close
    of bar t is executed on bar t+1. This removes same-bar look-ahead.
    """
    df = df.copy()
    df['position'] = df[signal_col].shift(1).fillna(0)
    df['price_returns'] = df['close'].pct_change()

    # Cost applied on every change in position size (a flip from +1 to -1 costs 2x)
    position_changes = df['position'].diff().abs()
    df['trade_cost'] = position_changes * (fee + slippage)
    df['strat_returns'] = df['position'] * df['price_returns'] - df['trade_cost']
    df['equity'] = initial_capital * (1 + df['strat_returns']).cumprod()
    return df

result = composite_signal(df_feat)
result = vectorized_backtest(result)
```

### 8.2 Event-Based Backtest with vectorbt

The vectorized backtest in 8.1 supports both long and short positions (`position` takes values −1, 0, +1). The `vbt.Portfolio.from_signals` configuration below is long/flat only — it treats `final_signal == -1` as an exit, not a short. The two engines therefore measure different strategies; use the vectorbt run as a cross-check on the long side, not as a replication of the vectorized result.

```python
import vectorbt as vbt

entries = result['final_signal'] == 1
exits = result['final_signal'] == -1  # exit long; not a short entry

portfolio = vbt.Portfolio.from_signals(
    result['close'], entries, exits,
    size=0.95, fees=0.001, slippage=0.0005,
    init_cash=10000, freq='1D',
)
print(portfolio.stats())
```

### 8.3 Errors That Invalidate Backtests

```python
# WRONG: same-bar look-ahead. The signal uses information from bar t
# to trade bar t's return.
df['signal'] = compute_signal(df)
df['returns'] = df['signal'] * df['price_returns']

# CORRECT: shift the signal so execution happens on the next bar.
df['position'] = df['signal'].shift(1)
df['returns'] = df['position'] * df['price_returns']
```

```python
# WRONG: normalization fit on the full series leaks future statistics
# into past observations.
scaler = StandardScaler()
df['close_norm'] = scaler.fit_transform(df[['close']])

# CORRECT: rolling normalization uses only past data at each point.
def rolling_zscore(series, window=252):
    return (series - series.rolling(window).mean()) / series.rolling(window).std()

df['close_norm'] = rolling_zscore(df['close'])
```

---

## 9. Evaluation Metrics

Total return alone is not sufficient to judge a strategy. The metrics below evaluate risk-adjusted performance and tail behavior.

**Sharpe ratio**

```
SR = (R_p - R_f) / sigma_p

R_p     = annualized return
R_f     = risk-free rate (~0 for crypto, or a USDT yield)
sigma_p = annualized standard deviation of returns
```

Reference values: SR > 1.0 is the minimum bar, > 1.5 is reasonable, > 2.0 is strong. Live Sharpe is commonly 30–50% below backtest Sharpe.

**Sortino ratio** — penalizes downside volatility only, which is more appropriate for an asset class with frequent large upside moves:

```
Sortino = (R_p - R_f) / sigma_downside
sigma_downside = annualized std of negative returns only
```

**Maximum drawdown (MDD)**

```
MDD = max over t of (peak_before_t - trough_after_t) / peak_before_t
```

**Calmar ratio**

```
Calmar = Annualized Return / |MDD|
```

**Profit factor**

```
PF = gross profit / gross loss
PF > 1.5 viable, PF > 2.0 good
```

```python
def compute_metrics(equity_curve, returns, risk_free=0.0, freq=365):
    """Compute risk-adjusted performance metrics. freq=365 for daily crypto."""
    returns = returns.dropna()
    total_return = equity_curve.iloc[-1] / equity_curve.iloc[0] - 1
    ann_return = (1 + total_return) ** (freq / len(returns)) - 1
    ann_vol = returns.std() * np.sqrt(freq)
    sharpe = (ann_return - risk_free) / ann_vol if ann_vol else 0

    downside = returns[returns < 0]
    downside_vol = downside.std() * np.sqrt(freq)
    sortino = (ann_return - risk_free) / downside_vol if downside_vol else 0

    drawdown = (equity_curve - equity_curve.cummax()) / equity_curve.cummax()
    max_dd = drawdown.min()
    calmar = ann_return / abs(max_dd) if max_dd else 0

    wins = returns[returns > 0]
    losses = returns[returns < 0]
    profit_factor = wins.sum() / abs(losses.sum()) if losses.sum() else np.inf
    active = returns[returns != 0]
    win_rate = len(wins) / len(active) if len(active) else 0

    return pd.Series({
        'Total Return': f"{total_return:.2%}",
        'Ann. Return': f"{ann_return:.2%}",
        'Ann. Volatility': f"{ann_vol:.2%}",
        'Sharpe Ratio': f"{sharpe:.3f}",
        'Sortino Ratio': f"{sortino:.3f}",
        'Max Drawdown': f"{max_dd:.2%}",
        'Calmar Ratio': f"{calmar:.3f}",
        'Profit Factor': f"{profit_factor:.3f}",
        'Win Rate': f"{win_rate:.2%}",
        'Total Trades': f"{len(active)}",
    })

print(compute_metrics(result['equity'], result['strat_returns']))
```

---

## 10. Walk-Forward Validation and Robustness Checks

In-sample backtest results overstate live performance. Walk-forward validation, where the model is fit on one window and evaluated on the following unseen window, is the primary defense against this and the main differentiator between strategies that survive deployment and those that do not.

### 10.1 Walk-Forward Architecture

```
[==TRAIN 1==][TEST 1]
      [==TRAIN 2==][TEST 2]
            [==TRAIN 3==][TEST 3]
                  [==TRAIN 4==][TEST 4]
                        [==TRAIN 5==][TEST 5]  <- final out-of-sample
```

A warmup buffer is prepended to each test window so that rolling indicators are valid from the first test bar. Without it, the rolling means, standard deviations, and z-scores at the start of each test window are computed on too few observations and the early signals are degraded.

```python
def walk_forward_validation(df, train_window=365, test_window=90, warmup=60):
    """Train the regime model on each window, evaluate on the next unseen window."""
    results = []
    n = len(df)
    start = 0

    while start + train_window + test_window <= n:
        train_end = start + train_window
        test_end = train_end + test_window

        df_train = df.iloc[start:train_end]

        # Include `warmup` rows before the test window so rolling features are valid
        warm_start = max(0, train_end - warmup)
        df_test_full = df.iloc[warm_start:test_end].copy()

        # Fit on train only; predict out-of-sample on the test window
        _, model, scaler, state_map = fit_hmm_regimes(df_train, verbose=False)
        regimes_test = predict_regimes(df_test_full, model, scaler, state_map)
        df_test_full = df_test_full.join(regimes_test, how='inner')

        # Generate signals on the buffered frame, then keep only the true test window
        df_test_full = composite_signal(df_test_full)
        df_test_full = vectorized_backtest(df_test_full)
        df_test = df_test_full.loc[df.index[train_end]:df.index[test_end - 1]]

        rets = df_test['strat_returns']
        sharpe = rets.mean() / rets.std() * np.sqrt(365) if rets.std() else 0
        results.append({
            'start': df_test.index[0],
            'end': df_test.index[-1],
            'return': rets.sum(),
            'sharpe': sharpe,
            'n_trades': int((df_test['final_signal'] != 0).sum()),
        })
        start += test_window

    return pd.DataFrame(results)

wf_results = walk_forward_validation(df_feat)
print(wf_results)
print(f"\nMedian out-of-sample Sharpe: {wf_results['sharpe'].median():.3f}")
print(f"Positive periods: {(wf_results['return'] > 0).mean():.2%}")
```

The Sharpe annualization here uses √365 to match the daily crypto convention used in `compute_metrics`. Mixing √252 (equities) and √365 (crypto) across the codebase produces metrics that are not comparable.

### 10.2 Monte Carlo Resampling

Resampling realized returns estimates the distribution of outcomes and the probability of severe loss. Bootstrap resampling assumes returns are independent; because crypto returns exhibit volatility clustering (Section 2.1), this understates the probability of consecutive losses. Treat the ruin probability as a lower bound, or use a block bootstrap to partially preserve serial structure.

```python
def monte_carlo_simulation(returns, n_simulations=1000, confidence=0.95, ruin_level=0.5):
    """Bootstrap equity curves to estimate the outcome distribution.

    Note: i.i.d. resampling ignores volatility clustering and therefore
    underestimates the probability of consecutive losses.
    """
    returns = np.asarray(returns)
    n = len(returns)
    finals = []

    for _ in range(n_simulations):
        sampled = np.random.choice(returns, size=n, replace=True)
        finals.append(10000 * (1 + sampled).prod())

    finals = np.array(finals)
    ruin_probability = (finals < 10000 * ruin_level).mean()

    print(f"Median final equity: ${np.median(finals):,.0f}")
    print(f"5th percentile:      ${np.percentile(finals, 5):,.0f}")
    print(f"95th percentile:     ${np.percentile(finals, 95):,.0f}")
    print(f"Ruin probability (<{ruin_level:.0%} capital): {ruin_probability:.2%}")
    return finals

_ = monte_carlo_simulation(result['strat_returns'].dropna().values)
```

---

## 11. Deployment

### 11.1 Pre-Deployment Checklist

* Out-of-sample Sharpe ≥ 0.7, measured on walk-forward test windows, not in-sample.
* Expectation adjusted for a 30–50% reduction in live Sharpe relative to backtest.
* Paper trading for at least 30 days before committing capital.
* Slippage model widened by 0.1–0.3% for market orders.
* Drawdown budget set to roughly 2× the backtest MDD as a conservative planning figure.
* Regime model retrained on a fixed schedule (for example monthly).

### 11.2 Position Sizing with the Kelly Criterion

```
Kelly fraction = (p * b - q) / b

p = win rate
q = 1 - p
b = average win / average loss
```

```python
def kelly_position_size(returns, fraction=0.5, cap=0.25):
    """Half-Kelly sizing, capped. Full Kelly is too aggressive in practice."""
    wins = returns[returns > 0]
    losses = returns[returns < 0]
    active = returns[returns != 0]
    if len(wins) == 0 or len(losses) == 0 or len(active) == 0:
        return 0.0

    p = len(wins) / len(active)
    q = 1 - p
    b = wins.mean() / abs(losses.mean())
    kelly = (p * b - q) / b
    return max(0.0, min(kelly * fraction, cap))

position_size = kelly_position_size(result['strat_returns'].dropna())
print(f"Half-Kelly position size: {position_size:.2%}")
```

### 11.3 Live Execution Skeleton

This skeleton is consistent with the rest of the pipeline: it fetches both price and on-chain data, computes the same features, and uses `predict_regimes` with the fitted `hmm_model`, `hmm_scaler`, and `hmm_state_map` so the inference feature set matches the training feature set exactly. Computing features on three columns while the scaler was fit on a different set is a dimension mismatch that fails at runtime; centralizing on `REGIME_FEATURES` prevents it.

```python
import ccxt
import schedule
import time

def run_live_strategy():
    exchange = ccxt.binance({
        'apiKey': 'YOUR_KEY',
        'secret': 'YOUR_SECRET',
        'enableRateLimit': True,
    })

    # 1. Price data
    ohlcv = exchange.fetch_ohlcv('BTC/USDT', '1d', limit=400)
    live = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    live['datetime'] = pd.to_datetime(live['timestamp'], unit='ms')
    live.set_index('datetime', inplace=True)

    # 2. On-chain data (required by the composite signal)
    live['sopr'] = get_glassnode_metric('indicators/sopr')
    live['mvrv'] = get_glassnode_metric('market/mvrv')
    live['exchange_flow'] = get_glassnode_metric('distribution/exchange_net_position_change')
    live = live.ffill()

    # 3. Features (same functions as the backtest)
    live = compute_price_features(live)
    live = compute_onchain_features(live)
    live.dropna(inplace=True)

    # 4. Regime inference using the fitted model (same REGIME_FEATURES)
    regimes_live = predict_regimes(live, hmm_model, hmm_scaler, hmm_state_map)
    live = live.join(regimes_live, how='inner')

    # 5. Signal
    live = composite_signal(live)
    signal = live['final_signal'].iloc[-1]
    regime = int(live['regime'].iloc[-1])

    # 6. Execution with balance checks
    balance = exchange.fetch_balance()
    free_btc = balance['BTC']['free']
    free_usdt = balance['USDT']['free']
    last_price = live['close'].iloc[-1]

    if signal == 1 and free_usdt > 100:
        size = free_usdt * position_size / last_price
        exchange.create_market_buy_order('BTC/USDT', size)
        print(f"BUY {size:.6f} BTC")
    elif signal == -1 and free_btc > 0.001:
        exchange.create_market_sell_order('BTC/USDT', free_btc)
        print(f"SELL {free_btc:.6f} BTC")
    else:
        print(f"HOLD (signal={signal}, regime={regime})")

# Run shortly after the daily candle closes (00:05 UTC)
schedule.every().day.at("00:05").do(run_live_strategy)
while True:
    schedule.run_pending()
    time.sleep(60)
```

On a spot account, `final_signal == -1` cannot open a short. The skeleton treats it as a flatten (sell to USDT), matching the long/flat behavior. To trade the short side, route −1 to a futures or margin account and adjust the balance and order logic accordingly.

---

## 12. Limitations

### 12.1 Structural Constraints

| Constraint | Description | Mitigation |
| --- | --- | --- |
| Overfitting | Backtest Sharpe is commonly 30–50% above live Sharpe because the strategy fits historical noise | Walk-forward and mandatory out-of-sample validation |
| Regime shift | An HMM fit on one period may not characterize a later period | Retrain on schedule; monitor performance decay |
| Market impact | Backtests assume fills do not move the price | Keep order size below ~1% of daily volume |
| Structural change | Spot BTC ETFs (since January 2024) altered on-chain and flow dynamics | Incorporate ETF flow data, or down-weight pre-2024 on-chain features |

### 12.2 Survivorship Bias

Exchange data contains only assets still listed. Delisted tokens are absent, so altcoin backtests are optimistic relative to the realized universe. Restricting analysis to BTC and ETH, which have the cleanest and longest histories, is the simplest mitigation. For broader coverage including delisted assets, a survivorship-bias-free data source is required.

### 12.3 Fundamental Limits

The pipeline cannot anticipate exchange failures (for example Mt. Gox, FTX, or the Bybit incident in February 2025), regulatory actions, or non-public information held by project teams and early investors. The realistic objective is a small, consistent post-cost edge — on the order of +0.5–1% monthly alpha — rather than a system that solves the market. Risk controls, position limits, and drawdown budgets exist precisely because these tail events are unpredictable.

---

## 13. References and Tools

**Libraries**

* `ccxt` — unified exchange API client
* `vectorbt` — vectorized backtesting
* `hmmlearn` — hidden Markov models
* `backtrader` — event-based backtesting
* `quantstats` — performance reporting

**Data platforms**

* Glassnode — https://docs.glassnode.com (verify current free-tier metric availability)
* CryptoQuant — https://cryptoquant.com/product/api
* Dune Analytics — https://dune.com (on-chain SQL, free tier)

**Background reading**

* L. La Morgia et al., "Pump and Dump in the Bitcoin Era," arXiv:2005.06610 — https://arxiv.org/abs/2005.06610
* Literature on cryptocurrency market microstructure and on HMM-based regime detection. Verify specific titles, venues, and dates against the source databases before citing, as some preprints are not peer-reviewed.

---

*For research and educational purposes only. Not financial advice. All code is provided as a starting point and requires independent validation before any use with real capital.*

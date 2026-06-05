# Reverse Engineering the Crypto Market: A Quant & Backtesting Approach

**Author:** Betha Morrison  
**Level:** Intermediate–Advanced  
**Focus:** Algorithmic approach, end-to-end pipeline, working code, and honest limitations

---

## Table of Contents

1. [Framing: What "Reverse Engineering" Actually Means Here](#1-framing-what-reverse-engineering-actually-means-here)
2. [Statistical Properties of Crypto Markets](#2-statistical-properties-of-crypto-markets)
3. [End-to-End Pipeline](#3-end-to-end-pipeline)
4. [Data Collection: CCXT + On-Chain API](#4-data-collection-ccxt--on-chain-api)
5. [Feature Engineering](#5-feature-engineering)
6. [Regime Detection with Hidden Markov Models](#6-regime-detection-with-hidden-markov-models)
7. [Signal Generation: Strategies with Actual Formulas](#7-signal-generation-strategies-with-actual-formulas)
8. [Backtesting Correctly](#8-backtesting-correctly)
9. [Evaluation Metrics](#9-evaluation-metrics)
10. [Walk-Forward Validation & Overfitting Prevention](#10-walk-forward-validation--overfitting-prevention)
11. [From Backtest to Live Trading](#11-from-backtest-to-live-trading)
12. [Honest Limitations](#12-honest-limitations)

---

## 1. Framing: What "Reverse Engineering" Actually Means Here

"Reverse engineering" here does not mean **predicting prices perfectly** — that's not possible. What it does mean:

> **Identifying repeating statistical patterns in market data, then building a system that exploits those patterns consistently with a probabilistic edge.**

Think of it like a cryptanalyst who doesn't try to guess the encryption key directly, but studies character distribution to find structural weaknesses.

Crypto has some practical advantages over traditional markets:
- Tick-level historical data is freely available (Binance, Bybit, etc.)
- On-chain data is fully transparent and tamper-proof
- The market is young — there are more inefficiencies to exploit
- Exchange APIs are easy to integrate for live deployment

---

## 2. Statistical Properties of Crypto Markets

Before building anything, understand what you're actually working with.

### 2.1 Return Distribution

Crypto returns are **not normally distributed** (non-Gaussian). They show:
- **Fat tails** (kurtosis > 3): extreme events happen more often than a normal distribution would predict
- **Negative skewness** (on daily timeframes): crashes are sharper than rallies
- **Volatility clustering**: high-volatility periods tend to follow other high-volatility periods (ARCH effects)

```python
import numpy as np
import pandas as pd
from scipy import stats
import ccxt

# Fetch daily BTC data
exchange = ccxt.binance()
ohlcv = exchange.fetch_ohlcv('BTC/USDT', timeframe='1d', limit=500)
df = pd.DataFrame(ohlcv, columns=['timestamp','open','high','low','close','volume'])
df['returns'] = df['close'].pct_change().dropna()

# Test for normality
print(f"Kurtosis: {stats.kurtosis(df['returns'].dropna()):.2f}")  # Normal = 0
print(f"Skewness: {stats.skew(df['returns'].dropna()):.2f}")      # Normal = 0

# Jarque-Bera test (p < 0.05 = reject normality)
jb_stat, jb_p = stats.jarque_bera(df['returns'].dropna())
print(f"Jarque-Bera p-value: {jb_p:.6f}")
```

Strategies that assume normality (standard Bollinger Bands, for example) will systematically underestimate tail risk.

### 2.2 Autocorrelation and Mean Reversion

```python
from statsmodels.tsa.stattools import adfuller, acf

# ADF Test: is the series stationary (i.e., not a random walk)?
adf_result = adfuller(df['close'])
print(f"ADF p-value (price): {adf_result[1]:.4f}")  # Usually > 0.05 -> non-stationary

# Returns are typically stationary
adf_result_ret = adfuller(df['returns'].dropna())
print(f"ADF p-value (returns): {adf_result_ret[1]:.4f}")  # Usually < 0.05 -> stationary

# Autocorrelation of returns
acf_vals = acf(df['returns'].dropna(), nlags=10)
print("ACF returns (lag 1-5):", acf_vals[1:6])
# Typically close to zero -> direct prediction from past returns is hard
```

### 2.3 Hurst Exponent

The Hurst Exponent (H) measures whether a series is trending, mean-reverting, or random:

```
H > 0.5  -> Trending (persistent)
H = 0.5  -> Random walk
H < 0.5  -> Mean-reverting (anti-persistent)
```

```python
def hurst_exponent(ts, max_lag=20):
    """Compute Hurst Exponent via R/S Analysis"""
    lags = range(2, max_lag)
    tau  = [np.std(np.subtract(ts[lag:], ts[:-lag])) for lag in lags]
    poly = np.polyfit(np.log(lags), np.log(tau), 1)
    return poly[0]

H = hurst_exponent(df['close'].values)
print(f"Hurst Exponent BTC (daily close): {H:.4f}")
```

Daily BTC data typically comes in around H ≈ 0.55–0.65 (slightly trending), though this shifts depending on the regime.

---

## 3. End-to-End Pipeline

```
Raw Data (OHLCV + On-Chain)
         │
         ▼
Feature Engineering
(returns, volatility, on-chain metrics)
         │
         ▼
Regime Detection (HMM)
         │
         ├──► Regime = Bull  -> Strategy A (Trend Following)
         ├──► Regime = Bear  -> Strategy B (Short / Hedge)
         └──► Regime = Chop  -> Strategy C (Mean Reversion)
                   │
                   ▼
           Signal Generation
           (entry/exit rules)
                   │
                   ▼
           Position Sizing
           (Kelly, fixed fractional)
                   │
                   ▼
           Backtesting Engine
           (vectorbt / backtrader)
                   │
                   ▼
           Walk-Forward Validation
                   │
                   ▼
           Performance Evaluation
           (Sharpe, Sortino, Calmar, MDD)
```

---

## 4. Data Collection: CCXT + On-Chain API

### 4.1 Price Data via CCXT

```python
import ccxt
import pandas as pd
import time

def fetch_full_ohlcv(symbol='BTC/USDT', timeframe='1h', since_days=365):
    """Fetch full OHLCV history from Binance"""
    exchange = ccxt.binance({'enableRateLimit': True})
    since = exchange.parse8601(
        (pd.Timestamp.now() - pd.Timedelta(days=since_days)).strftime('%Y-%m-%dT%H:%M:%SZ')
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

    df = pd.DataFrame(all_ohlcv, columns=['timestamp','open','high','low','close','volume'])
    df['datetime'] = pd.to_datetime(df['timestamp'], unit='ms')
    df.set_index('datetime', inplace=True)
    return df

df = fetch_full_ohlcv('BTC/USDT', '1d', since_days=1000)
```

### 4.2 On-Chain Data via Glassnode API

```python
import requests

GLASSNODE_API_KEY = "your_api_key"

def get_glassnode_metric(metric, asset='BTC', resolution='24h'):
    """
    Free-tier metrics include:
    - indicators/sopr  -> Spent Output Profit Ratio
    - market/mvrv      -> Market Value to Realized Value
    - market/nupl      -> Net Unrealized Profit/Loss
    - distribution/exchange_net_position_change -> Exchange flow
    """
    url = f"https://api.glassnode.com/v1/metrics/{metric}"
    params = {
        'a': asset,
        'i': resolution,
        'api_key': GLASSNODE_API_KEY
    }
    response = requests.get(url, params=params)
    data = response.json()

    df = pd.DataFrame(data)
    df['datetime'] = pd.to_datetime(df['t'], unit='s')
    df.set_index('datetime', inplace=True)
    df.rename(columns={'v': metric}, inplace=True)
    return df[metric]

# Usage
sopr          = get_glassnode_metric('indicators/sopr')
mvrv          = get_glassnode_metric('market/mvrv')
exchange_flow = get_glassnode_metric('distribution/exchange_net_position_change')
```

### 4.3 Merge and Alignment

```python
# Merge all data sources
df_merged = df[['close','volume']].copy()
df_merged['sopr']          = sopr
df_merged['mvrv']          = mvrv
df_merged['exchange_flow'] = exchange_flow

# On-chain data sometimes has gaps — forward fill
df_merged = df_merged.fillna(method='ffill')
df_merged.dropna(inplace=True)
```

---

## 5. Feature Engineering

### 5.1 Price-Based Features

```python
def compute_price_features(df):
    df = df.copy()

    # Returns
    df['returns_1d']  = df['close'].pct_change()
    df['returns_7d']  = df['close'].pct_change(7)
    df['log_returns'] = np.log(df['close'] / df['close'].shift(1))

    # Realized volatility
    df['volatility_7d']  = df['log_returns'].rolling(7).std()  * np.sqrt(365)
    df['volatility_30d'] = df['log_returns'].rolling(30).std() * np.sqrt(365)
    df['vol_ratio']      = df['volatility_7d'] / df['volatility_30d']  # volatility regime signal

    # Momentum
    df['roc_14'] = df['close'].pct_change(14)  # Rate of Change
    df['roc_30'] = df['close'].pct_change(30)

    # Moving averages
    df['sma_50']          = df['close'].rolling(50).mean()
    df['sma_200']         = df['close'].rolling(200).mean()
    df['price_vs_sma200'] = (df['close'] - df['sma_200']) / df['sma_200']

    # Normalized ATR
    df['hl'] = df['high'] - df['low']
    df['hc'] = abs(df['high'] - df['close'].shift(1))
    df['lc'] = abs(df['low']  - df['close'].shift(1))
    df['tr'] = df[['hl','hc','lc']].max(axis=1)
    df['atr_14'] = df['tr'].rolling(14).mean()
    df['natr']   = df['atr_14'] / df['close']

    return df

# 5.2 On-Chain Features
def compute_onchain_features(df):
    df = df.copy()

    # MVRV Z-Score (rolling normalization)
    mvrv_mean = df['mvrv'].rolling(365).mean()
    mvrv_std  = df['mvrv'].rolling(365).std()
    df['mvrv_zscore'] = (df['mvrv'] - mvrv_mean) / mvrv_std

    # Smoothed SOPR (reduce noise)
    df['sopr_7d_ma'] = df['sopr'].rolling(7).mean()

    # Exchange flow momentum
    df['flow_momentum'] = df['exchange_flow'].rolling(7).sum()

    # Price-SOPR divergence: price rising while SOPR falls = distribution
    df['price_sopr_div'] = df['returns_7d'] - df['sopr'].pct_change(7)

    return df

df_feat = compute_price_features(df_merged)
df_feat = compute_onchain_features(df_feat)
df_feat.dropna(inplace=True)
```

---

## 6. Regime Detection with Hidden Markov Models

The market doesn't stay in one mode — it cycles between trending, choppy, and crashing phases. HMM lets us detect those states probabilistically without hard-coding thresholds.

**Core math:**

```
State sequence:  S_1 -> S_2 -> S_3 -> ... -> S_t
Observation:     O_1,  O_2,  O_3, ...,  O_t

Transition matrix A:  P(S_t | S_{t-1})
Emission matrix B:    P(O_t | S_t)  [Gaussian in this case]
```

```python
from hmmlearn import hmm
from sklearn.preprocessing import StandardScaler
import warnings
warnings.filterwarnings('ignore')

def fit_hmm_regimes(df, n_regimes=3, features=['log_returns', 'volatility_7d', 'vol_ratio']):
    """
    Fit a Gaussian HMM for regime detection.
    n_regimes=3: Bull, Bear, Chop (sideways)
    """
    X = df[features].dropna().values
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)

    model = hmm.GaussianHMM(
        n_components=n_regimes,
        covariance_type='full',
        n_iter=1000,
        random_state=42
    )
    model.fit(X_scaled)

    states = model.predict(X_scaled)

    # Label states by mean return (0=Bear, 1=Chop, 2=Bull)
    regime_stats = {}
    for s in range(n_regimes):
        mask = states == s
        regime_stats[s] = {
            'mean_return': df['log_returns'].dropna().values[mask].mean(),
            'mean_vol':    df['volatility_7d'].dropna().values[mask].mean(),
            'count':       mask.sum()
        }

    sorted_states = sorted(regime_stats.keys(), key=lambda s: regime_stats[s]['mean_return'])
    state_map     = {old: new for new, old in enumerate(sorted_states)}
    states_labeled = np.array([state_map[s] for s in states])

    labels = {0: 'Bear', 1: 'Chop', 2: 'Bull'}

    print("\n=== Regime Statistics ===")
    for i, label in labels.items():
        orig_state = sorted_states[i]
        st = regime_stats[orig_state]
        print(f"{label}: mean_return={st['mean_return']:.4f}, "
              f"mean_vol={st['mean_vol']:.4f}, "
              f"count={st['count']} days")

    print("\nTransition Matrix:")
    tm = model.transmat_
    for i in range(n_regimes):
        row = [tm[sorted_states[i]][sorted_states[j]] for j in range(n_regimes)]
        print(f"  From {labels[i]}: {['->'+labels[j]+f'={v:.2f}' for j,v in enumerate(row)]}")

    return states_labeled, model, scaler, state_map

features = ['log_returns', 'volatility_7d', 'vol_ratio', 'mvrv_zscore', 'sopr_7d_ma']
regimes, hmm_model, scaler, state_map = fit_hmm_regimes(df_feat, n_regimes=3, features=features)
df_feat = df_feat.iloc[-len(regimes):].copy()
df_feat['regime'] = regimes
```

**Expected output (example on BTC 2019–2024):**
```
=== Regime Statistics ===
Bear: mean_return=-0.0082, mean_vol=0.89, count=298 days
Chop: mean_return=+0.0012, mean_vol=0.54, count=412 days
Bull: mean_return=+0.0094, mean_vol=0.73, count=290 days

Transition Matrix:
  From Bear: [->Bear=0.91, ->Chop=0.07, ->Bull=0.02]
  From Chop: [->Bear=0.05, ->Chop=0.88, ->Bull=0.07]
  From Bull: [->Bear=0.03, ->Chop=0.08, ->Bull=0.89]
```

High diagonal values mean regimes are persistent — that's what makes them exploitable.

---

## 7. Signal Generation: Strategies with Actual Formulas

### 7.1 Trend Following (active in Bull/Bear regimes)

```python
def trend_signal(df):
    """
    SMA crossover with regime filter.
    Long  -> only in Bull regime
    Short -> only in Bear regime
    """
    df = df.copy()

    sma_fast = df['close'].rolling(20).mean()
    sma_slow = df['close'].rolling(50).mean()

    df['trend_signal'] = 0
    df.loc[(sma_fast > sma_slow) & (df['regime'] == 2), 'trend_signal'] =  1  # Long
    df.loc[(sma_fast < sma_slow) & (df['regime'] == 0), 'trend_signal'] = -1  # Short

    return df['trend_signal']
```

### 7.2 Mean Reversion (active in Chop regime)

Using a rolling Z-Score:

```
z_score_t = (price_t - μ_window) / σ_window

Entry long  -> z < -1.5 AND regime == Chop
Entry short -> z > +1.5 AND regime == Chop
Exit        -> |z| < 0.5
```

```python
def mean_reversion_signal(df, window=20, entry_z=1.5, exit_z=0.5):
    """Z-score mean reversion, active only in Chop regime"""
    df = df.copy()

    rolling_mean = df['close'].rolling(window).mean()
    rolling_std  = df['close'].rolling(window).std()
    df['zscore'] = (df['close'] - rolling_mean) / rolling_std

    df['mr_signal'] = 0
    df.loc[(df['zscore'] < -entry_z) & (df['regime'] == 1), 'mr_signal'] =  1
    df.loc[(df['zscore'] >  entry_z) & (df['regime'] == 1), 'mr_signal'] = -1

    # Exit when price returns to mean
    prev_signal = df['mr_signal'].replace(0, np.nan).ffill()
    df.loc[(abs(df['zscore']) < exit_z) & (prev_signal != 0), 'mr_signal'] = 0

    return df['mr_signal']
```

### 7.3 On-Chain Divergence Signal

```python
def onchain_divergence_signal(df):
    """
    Bullish divergence: price falling but SOPR recovering + MVRV < 1.0
    Bearish divergence: price rising but SOPR falling + MVRV > 2.5 (distribution)
    """
    df = df.copy()

    price_trend = np.sign(df['returns_7d'])
    sopr_trend  = np.sign(df['sopr'].pct_change(7))

    df['oc_signal'] = 0
    df.loc[(price_trend < 0) & (sopr_trend > 0) & (df['mvrv'] < 1.0), 'oc_signal'] =  1
    df.loc[(price_trend > 0) & (sopr_trend < 0) & (df['mvrv'] > 2.5), 'oc_signal'] = -1

    return df['oc_signal']
```

### 7.4 Composite Signal

```python
def composite_signal(df, weights={'trend': 0.4, 'mr': 0.3, 'onchain': 0.3}):
    """Combine signals with weights"""
    df = df.copy()

    df['trend_signal']   = trend_signal(df)
    df['mr_signal']      = mean_reversion_signal(df)
    df['onchain_signal'] = onchain_divergence_signal(df)

    df['composite'] = (
        df['trend_signal']   * weights['trend'] +
        df['mr_signal']      * weights['mr'] +
        df['onchain_signal'] * weights['onchain']
    )

    # Entry threshold
    df['final_signal'] = 0
    df.loc[df['composite'] >  0.25, 'final_signal'] =  1
    df.loc[df['composite'] < -0.25, 'final_signal'] = -1

    return df
```

---

## 8. Backtesting Correctly

### 8.1 Vectorized Backtesting (fast, good for iteration)

```python
def vectorized_backtest(df, signal_col='final_signal',
                        fee=0.001, slippage=0.0005,
                        initial_capital=10000.0):
    """
    Vectorized backtest — no look-ahead bias if signals
    are constructed correctly (shift(1) for next-bar execution).
    """
    df = df.copy()

    # CRITICAL: today's signal -> execute at tomorrow's open
    df['position']      = df[signal_col].shift(1).fillna(0)
    df['price_returns'] = df['close'].pct_change()

    # Cost model: fee (taker) + slippage, applied on every position change
    position_changes = df['position'].diff().abs()
    df['trade_cost']     = position_changes * (fee + slippage)
    df['strat_returns']  = df['position'] * df['price_returns'] - df['trade_cost']
    df['equity']         = initial_capital * (1 + df['strat_returns']).cumprod()

    return df

result = composite_signal(df_feat)
result = vectorized_backtest(result)
```

### 8.2 Event-Based Backtesting with vectorbt

```python
import vectorbt as vbt

entries = result['final_signal'] == 1
exits   = result['final_signal'] == -1

portfolio = vbt.Portfolio.from_signals(
    result['close'],
    entries,
    exits,
    size=0.95,          # 95% of capital per trade
    fees=0.001,         # 0.1% taker fee (Binance)
    slippage=0.0005,    # 0.05% slippage estimate
    init_cash=10000,
    freq='1D'
)

stats = portfolio.stats()
print(stats)
```

### 8.3 Common Bugs That Break Backtests

```python
# WRONG: Look-ahead bias
# Today's signal executed on today's return
df['signal']  = compute_signal(df)
df['returns'] = df['signal'] * df['price_returns']  # uses same-bar return -> cheating

# CORRECT: Shift the signal
df['position'] = df['signal'].shift(1)  # execute on next bar
df['returns']  = df['position'] * df['price_returns']

# WRONG: Data leakage in normalization
scaler = StandardScaler()
df['close_norm'] = scaler.fit_transform(df[['close']])  # fit on full dataset including future -> cheating

# CORRECT: Rolling normalization (only past data)
def rolling_zscore(series, window=252):
    return (series - series.rolling(window).mean()) / series.rolling(window).std()

df['close_norm'] = rolling_zscore(df['close'])
```

---

## 9. Evaluation Metrics

### Formulas and Benchmarks

**Sharpe Ratio:**
```
SR = (R_p - R_f) / σ_p × √T

Where:
  R_p = annualized strategy return
  R_f = risk-free rate (≈ 0 for crypto, or use USDT yield)
  σ_p = annualized return standard deviation
  T   = 365 (daily), 52 (weekly)
```

Benchmarks: SR > 1.0 is the minimum bar, > 1.5 is decent, > 2.0 is strong. Live Sharpe is typically 30–50% lower than backtest.

**Sortino Ratio** (more appropriate for crypto):
```
Sortino = (R_p - R_f) / σ_downside × √T

σ_downside = std of NEGATIVE returns only
```

Sortino doesn't penalize upside volatility, which matters when crypto spikes hard to the upside.

**Maximum Drawdown:**
```
MDD = max over t of: (peak_before_t - trough_after_t) / peak_before_t
```

**Calmar Ratio:**
```
Calmar = Annualized Return / |MDD|
```

**Profit Factor:**
```
PF = Gross Profit / Gross Loss
PF > 1.5 = viable, PF > 2.0 = good
```

```python
def compute_metrics(equity_curve, returns, risk_free=0.0, freq=365):
    """Compute all metrics at once"""
    total_return = (equity_curve.iloc[-1] / equity_curve.iloc[0]) - 1

    ann_return = (1 + total_return) ** (freq / len(returns)) - 1
    ann_vol    = returns.std() * np.sqrt(freq)
    sharpe     = (ann_return - risk_free) / ann_vol if ann_vol != 0 else 0

    downside_returns = returns[returns < 0]
    ann_downside_vol = downside_returns.std() * np.sqrt(freq)
    sortino = (ann_return - risk_free) / ann_downside_vol if ann_downside_vol != 0 else 0

    rolling_max = equity_curve.cummax()
    drawdown    = (equity_curve - rolling_max) / rolling_max
    max_dd      = drawdown.min()

    calmar = ann_return / abs(max_dd) if max_dd != 0 else 0

    wins   = returns[returns > 0]
    losses = returns[returns < 0]
    profit_factor = wins.sum() / abs(losses.sum()) if losses.sum() != 0 else np.inf
    win_rate      = len(wins) / len(returns[returns != 0]) if len(returns[returns != 0]) > 0 else 0

    metrics = {
        'Total Return':    f"{total_return:.2%}",
        'Ann. Return':     f"{ann_return:.2%}",
        'Ann. Volatility': f"{ann_vol:.2%}",
        'Sharpe Ratio':    f"{sharpe:.3f}",
        'Sortino Ratio':   f"{sortino:.3f}",
        'Max Drawdown':    f"{max_dd:.2%}",
        'Calmar Ratio':    f"{calmar:.3f}",
        'Profit Factor':   f"{profit_factor:.3f}",
        'Win Rate':        f"{win_rate:.2%}",
        'Total Trades':    f"{len(returns[returns != 0])}"
    }

    return pd.Series(metrics)

metrics = compute_metrics(result['equity'], result['strat_returns'])
print(metrics)
```

---

## 10. Walk-Forward Validation & Overfitting Prevention

This is the section most people skip, and it's the main reason strategies look good in backtests but fall apart in production.

### 10.1 Walk-Forward Architecture

```
Historical data (e.g., 2019–2025):

[==TRAIN 1==][TEST 1]
      [==TRAIN 2==][TEST 2]
            [==TRAIN 3==][TEST 3]
                  [==TRAIN 4==][TEST 4]
                        [==TRAIN 5==][TEST 5]  ← OOS Final
```

```python
def walk_forward_validation(df, train_window=365, test_window=90,
                             param_grid=None):
    """
    Walk-forward: train on a window, test on the next period.
    Repeat with a rolling window.
    """
    results = []
    n = len(df)
    start = 0

    while start + train_window + test_window <= n:
        train_end = start + train_window
        test_end  = train_end + test_window

        df_train = df.iloc[start:train_end].copy()
        df_test  = df.iloc[train_end:test_end].copy()

        # 1. Fit regime model on train only
        regimes_train, model, scaler, state_map = fit_hmm_regimes(
            df_train,
            features=['log_returns', 'volatility_7d', 'vol_ratio']
        )

        # 2. Predict regime on test (out-of-sample)
        X_test        = df_test[['log_returns', 'volatility_7d', 'vol_ratio']].dropna().values
        X_test_scaled = scaler.transform(X_test)
        regimes_test_raw = model.predict(X_test_scaled)
        regimes_test     = np.array([state_map.get(s, 1) for s in regimes_test_raw])

        df_test = df_test.iloc[-len(regimes_test):].copy()
        df_test['regime'] = regimes_test

        # 3. Generate signals and backtest on test window
        df_test = composite_signal(df_test)
        df_test = vectorized_backtest(df_test)

        period_return = df_test['strat_returns'].sum()
        period_sharpe = (df_test['strat_returns'].mean() /
                         df_test['strat_returns'].std() * np.sqrt(252)
                         if df_test['strat_returns'].std() != 0 else 0)

        results.append({
            'start':    df_test.index[0],
            'end':      df_test.index[-1],
            'return':   period_return,
            'sharpe':   period_sharpe,
            'n_trades': (df_test['final_signal'] != 0).sum()
        })

        start += test_window  # roll forward

    return pd.DataFrame(results)

wf_results = walk_forward_validation(df_feat)
print(wf_results)
print(f"\nMedian Out-of-Sample Sharpe: {wf_results['sharpe'].median():.3f}")
print(f"Positive periods: {(wf_results['return'] > 0).mean():.2%}")
```

### 10.2 Monte Carlo Simulation

```python
def monte_carlo_simulation(returns, n_simulations=1000, confidence=0.95):
    """
    Resample trade outcomes to estimate the distribution of equity curves.
    """
    n = len(returns)
    equity_curves = []

    for _ in range(n_simulations):
        shuffled = np.random.choice(returns, size=n, replace=True)
        equity   = 10000 * (1 + shuffled).cumprod()
        equity_curves.append(equity)

    equity_array = np.array(equity_curves)

    lower = np.percentile(equity_array, (1 - confidence) * 100 / 2, axis=0)
    upper = np.percentile(equity_array, 100 - (1 - confidence) * 100 / 2, axis=0)
    median = np.median(equity_array, axis=0)

    final_equities = equity_array[:, -1]
    ruin_probability = (final_equities < 5000).mean()  # < 50% of starting capital

    print(f"Median final equity: ${np.median(final_equities):,.0f}")
    print(f"5th percentile:      ${np.percentile(final_equities, 5):,.0f}")
    print(f"95th percentile:     ${np.percentile(final_equities, 95):,.0f}")
    print(f"Ruin probability (<50% capital): {ruin_probability:.2%}")

    return median, lower, upper

mc_median, mc_lower, mc_upper = monte_carlo_simulation(
    result['strat_returns'].dropna().values
)
```

---

## 11. From Backtest to Live Trading

### 11.1 Pre-Deployment Checklist

- [ ] **OOS Sharpe ≥ 0.7** (not in-sample)
- [ ] **Live Sharpe is typically 30–50% lower than backtest** — adjust expectations accordingly
- [ ] **Paper trade for at least 30 days** before deploying real capital
- [ ] **Realistic slippage model:** add 0.1–0.3% on top for market orders
- [ ] **Stress-test drawdown:** multiply backtest MDD × 2 as a conservative estimate
- [ ] **Regime model should be retrained monthly** — the market doesn't stay the same

### 11.2 Position Sizing with Kelly Criterion

```
Kelly fraction = (p × b - q) / b

Where:
  p = win rate
  b = average gain / average loss (reward-to-risk ratio)
  q = 1 - p (loss rate)
```

```python
def kelly_position_size(returns, fraction=0.5):
    """
    Full Kelly is usually too aggressive.
    Half Kelly (fraction=0.5) is more practical.
    """
    wins   = returns[returns > 0]
    losses = returns[returns < 0]

    if len(wins) == 0 or len(losses) == 0:
        return 0.0

    p = len(wins) / len(returns[returns != 0])
    q = 1 - p
    b = wins.mean() / abs(losses.mean())

    kelly_full = (p * b - q) / b
    kelly_frac = kelly_full * fraction

    return max(0, min(kelly_frac, 0.25))  # cap at 25% per trade

position_size = kelly_position_size(result['strat_returns'])
print(f"Recommended position size (Half Kelly): {position_size:.2%}")
```

### 11.3 Live Bot Skeleton

```python
import ccxt
import schedule
import time

def run_live_strategy():
    exchange = ccxt.binance({
        'apiKey':          'YOUR_KEY',
        'secret':          'YOUR_SECRET',
        'enableRateLimit': True
    })

    # 1. Fetch latest data
    ohlcv    = exchange.fetch_ohlcv('BTC/USDT', '1d', limit=300)
    df_live  = pd.DataFrame(ohlcv, columns=['timestamp','open','high','low','close','volume'])
    df_live['datetime'] = pd.to_datetime(df_live['timestamp'], unit='ms')
    df_live.set_index('datetime', inplace=True)

    # 2. Compute features
    df_live = compute_price_features(df_live)

    # 3. Predict regime (use the already-fitted model)
    X_live    = df_live[['log_returns','volatility_7d','vol_ratio']].dropna().values
    X_scaled  = scaler.transform(X_live)
    regime_pred = hmm_model.predict(X_scaled)
    df_live   = df_live.iloc[-len(regime_pred):].copy()
    df_live['regime'] = [state_map.get(s, 1) for s in regime_pred]

    # 4. Generate signal
    df_live        = composite_signal(df_live)
    current_signal = df_live['final_signal'].iloc[-1]

    # 5. Execute (with validation)
    balance      = exchange.fetch_balance()
    current_btc  = balance['BTC']['free']
    current_usdt = balance['USDT']['free']

    if current_signal == 1 and current_usdt > 100:
        order_size = current_usdt * position_size / df_live['close'].iloc[-1]
        exchange.create_market_buy_order('BTC/USDT', order_size)
        print(f"BUY {order_size:.6f} BTC")
    elif current_signal == -1 and current_btc > 0.001:
        exchange.create_market_sell_order('BTC/USDT', current_btc)
        print(f"SELL {current_btc:.6f} BTC")
    else:
        print(f"HOLD (signal={current_signal}, regime={df_live['regime'].iloc[-1]})")

# Run daily at 00:05 UTC (after daily candle closes)
schedule.every().day.at("00:05").do(run_live_strategy)
while True:
    schedule.run_pending()
    time.sleep(60)
```

---

## 12. Honest Limitations

This is the section most quant articles leave out, and it's the most important one.

### 12.1 Structural Problems

| Problem | What it means | Mitigation |
|---------|---------------|------------|
| **Overfitting** | The strategy memorized historical data. Backtest Sharpe is typically 30–50% higher than live | Walk-forward + mandatory OOS validation |
| **Regime shift** | An HMM trained on 2019–2021 data may not hold in 2024–2025 | Retrain periodically, monitor performance degradation |
| **Market impact** | Backtests don't model the fact that your own large orders move the price | Keep position size below 1% of daily volume |
| **ETF distortion** | Since January 2024, spot BTC ETFs have changed on-chain dynamics | Incorporate ETF flow data into the model |

### 12.2 Survivorship Bias

Exchange data only contains tokens that are still trading. Delisted altcoins aren't in the dataset — which means altcoin backtests will always look more optimistic than reality. The simplest fix is to restrict analysis to BTC/ETH, where historical data is the cleanest.

```python
# Use CoinGecko data if you need coverage of delisted tokens,
# or limit analysis to BTC/ETH for the most reliable history.
```

### 12.3 Fundamental Limits

No system can predict or preempt:
- Exchange hacks (Mt.Gox, FTX, Bybit Feb 2025)
- Sudden regulatory action
- Insider information (project teams, early-stage VCs)

The realistic goal is a system with a **small but consistent statistical edge** (+0.5–1% alpha per month after costs), not one that "solves" the market.

---

## References & Tools

**Libraries:**
- `ccxt` — connects to 100+ exchanges
- `vectorbt` — fast vectorized backtesting
- `hmmlearn` — Hidden Markov Models
- `backtrader` — event-based backtesting
- `quantstats` — automated performance reports

**Data Platforms:**
- Glassnode API: [docs.glassnode.com](https://docs.glassnode.com)
- CryptoQuant: [cryptoquant.com/product/api](https://cryptoquant.com/product/api)
- Dune Analytics: [dune.com](https://dune.com) (on-chain SQL, free tier)

**Academic Papers:**
- "Cryptocurrency Market Microstructure" — Annals of Operations Research (2023)
- "Pump and Dumps in the Bitcoin Era" — La Morgia et al., [arXiv:2005.06610](https://arxiv.org/abs/2005.06610)
- "Markov and HMM for Regime Detection in Crypto Markets" — [Preprints.org (2026)](https://www.preprints.org/manuscript/202603.0831)

---

*For research and educational purposes. Not financial advice.*

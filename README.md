# Momentum Trader

A Django application that runs a classic **12-1 cross-sectional momentum strategy** end to end:
it downloads daily prices from [Massive](https://massive.com) (formerly Polygon.io), ranks a stock
universe by its return over the past twelve months excluding the most recent one, turns the ranking
into rebalance signals and sends market orders to an **Alpaca paper trading** account. A vectorised
backtester and a small dashboard sit on top of the same core logic.

Inspired by the freeCodeCamp course *Python Algorithmic Trading with SnapTrade, Massive, Alpaca*,
rebuilt with a stricter separation between strategy logic, data access and execution, a test suite and
a no-lookahead backtest.

> **Paper trading only.** The Alpaca adapter is hard-wired to `paper=True`. This is an educational
> project, not investment advice.

![Dashboard](docs/dashboard.png)

## Strategy

At each month end *t* every stock gets the score

```
momentum(t) = P(t − 1) / P(t − 12) − 1
```

using month-end, split-adjusted closes. Skipping the latest month avoids the short-term reversal
effect documented by Jegadeesh (1990). The portfolio holds the top *N* names (default 10) in equal
weight. Names with negative momentum are left out by default, so in broad downturns the strategy
partly moves to cash. Live runs always form the portfolio on the last *completed* month, so a run on
2 October uses September month-end prices.

## Architecture

```
trader/
├── momentum.py          pure pandas logic: scores, target weights, rebalance plan
├── backtest.py          vectorised monthly backtest with transaction costs
├── data/providers.py    MassiveProvider (rate limited) and SyntheticProvider (offline)
├── brokers/             Broker protocol, AlpacaPaperBroker, DryRunBroker
├── services.py          DB glue: sync prices, score, build signals, execute
├── models.py            Stock, PriceBar, StrategyRun, MomentumScore, Signal, Order
├── management/commands  seed_universe, fetch_prices, run_strategy, backtest, demo
└── views.py, templates  dashboard and backtest pages
```

`momentum.py` and `backtest.py` know nothing about Django, so the same functions drive the live
pipeline, the backtest and the unit tests. Data providers and brokers are small adapters behind a
`Protocol`, which makes them easy to fake in tests and to swap later.

## Quickstart (offline, no API keys)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
python manage.py migrate
python manage.py demo          # synthetic prices → scores → dry-run signals → backtest
python manage.py runserver     # http://127.0.0.1:8000
```

Synthetic prices are uncorrelated random walks, so the demo backtest shows how the machinery works and
says nothing about real-world performance.

## Real data and paper trading

1. Create a free [Massive](https://massive.com) API key and an [Alpaca](https://alpaca.markets) paper
   account, then copy `.env.example` to `.env` and fill in the keys.
2. Load the universe and prices:

   ```bash
   python manage.py seed_universe                 # 30 US large caps, or --tickers AAPL,MSFT,...
   python manage.py fetch_prices --years 2        # free plan: 5 calls/min, about 6 minutes
   ```

3. Preview the rebalance against your real paper account, then execute:

   ```bash
   python manage.py run_strategy --broker alpaca            # dry run, nothing is sent
   python manage.py run_strategy --broker alpaca --execute  # sells first, then buys
   ```

Orders are fractional, notional market orders with `time_in_force=day`. Positions that are no longer
targeted are closed completely, including holdings outside the universe. Orders submitted outside
regular trading hours queue for the next session.

`run_strategy --execute` refuses to trade a formation month twice (override with `--force`), so a
scheduler can simply fire on every weekday of the first week and only the first run trades:

```cron
# 16:00 Zurich ≈ 10:00 New York, weekdays 1st to 7th of the month
0 16 1-7 * * [ "$(date +\%u)" -le 5 ] && cd /path/to/momentum-trader && .venv/bin/python manage.py fetch_prices --years 2 && .venv/bin/python manage.py run_strategy --broker alpaca --execute
```

## Backtest

```bash
python manage.py backtest --top-n 10 --cost-bps 5
```

writes `reports/monthly_returns.csv`, `weights.csv`, `metrics.csv` and `equity_curve.png`, and the
`/backtest/` page shows the same result interactively. Weights formed at month end *t* only earn the
return of month *t + 1*; a unit test checks this with a price jump that must not be captured early.
Costs are charged on turnover. The benchmark is the equal-weight universe.

Known simplifications: the universe is today's large caps, so the backtest carries survivorship bias;
turnover ignores intra-month drift; dividends are excluded because Massive aggregates are adjusted for
splits only. Massive's free plan covers about two years of history, which leaves roughly eleven months
of backtest after the twelve-month warm-up. A longer backtest needs a paid plan.

## Configuration

All strategy parameters live in `.env` (see `.env.example`): lookback, skip, top *N*, the positive
momentum filter, the fraction of equity to invest, the minimum trade size and transaction costs.

## Tests

```bash
pytest
ruff check .
```

The suite covers the momentum maths, the no-lookahead property, rebalance logic, both API adapters
(with fake clients) and the full DB pipeline including the dashboard. CI runs on GitHub Actions.

## License

MIT

# Testing

```powershell
pytest
```

`pyproject.toml` configures `pytest` to add `src` and the repo root to
`sys.path` (so both `trading_system` and `config` import correctly
without installing the package) and to run with coverage
(`--cov=trading_system --cov-report=term-missing`).

Unit tests (`tests/unit/`, also collected as `tests/test_*.py`) never hit
the real Kite API, PostgreSQL, or Redis -- they use an in-memory
`FakeBroker` (see `tests/conftest.py`) and pure in-memory components.
Phase 2 persistence-layer tests (`tests/unit/test_db_models.py`,
`tests/unit/test_repositories.py`, `tests/unit/test_postgres_client.py`)
run against an in-memory SQLite database (`tests/unit/conftest.py`'s
`db_engine`/`db_session` fixtures, with `PRAGMA foreign_keys=ON` enabled)
instead of a real PostgreSQL instance -- fast, hermetic, and never touches
production data. Phase 3 Redis tests (`tests/unit/test_redis_*.py`) run
against `fakeredis.FakeRedis` (`tests/unit/conftest.py`'s `redis_client`/
`redis_state`/`redis_keys` fixtures) instead of a real Redis instance, for
the same reason. Phase 6 market-data tests (`tests/unit/test_candle_aggregator.py`,
`test_indicators.py`, `test_historical_data.py`, `test_market_data_service.py`)
use the same in-memory SQLite/`fakeredis`/fake-broker doubles -- never
real Postgres/Redis/Kite. Phase 7 strategy/signal tests
(`tests/unit/test_strategy_engine.py`, `test_no_op_strategy.py`,
`test_signal_factory.py`, `test_signal_validator.py`, `test_patterns.py`,
`test_scoring.py`, `test_technical_scoring_strategy.py`) use the same
doubles plus a scripted test-only `BaseStrategy` (never a real trading
strategy). Phase 8 risk-engine tests (`tests/unit/test_risk_policy.py`,
`test_position_sizer.py`, `test_account_state.py`, `test_risk_state_store.py`,
`test_stop_loss.py`, `test_risk_engine.py`) use the same in-memory SQLite/`fakeredis` doubles
plus a scripted `AccountStateProvider` test double (never a real broker/
account balance). Phase 11's dashboard and editable-settings addendum
tests (`tests/unit/test_api.py`) use FastAPI's `TestClient` with every
dependency overridden -- an in-memory SQLite session (its own fixture,
not the shared one, since `TestClient` runs requests on a different
thread -- see [Phase 11](phases/phase-11-backtesting-paper.md)) and
trivial fakes for Postgres/Redis/Kite; never a real connection. They
cover normalized scores, metrics, protected settings writes, validation,
auditing, and runtime override precedence. Integration tests
(`tests/integration/`) are
skipped by default since they require real Postgres/Redis instances --
`test_storage_integration.py` covers Phase 2/3 connectivity,
`test_kite_market_data_integration.py` covers the Phase 4/5 instrument-sync/
tick-pipeline/option-chain flows, `test_strategy_engine_integration.py`
covers the Phase 7 `TickProcessor -> MarketDataService -> StrategyEngine
-> Signal -> PostgreSQL/Redis` flow end-to-end (with a fake `Broker`/
scripted strategy, never the real Kite API), and `test_risk_engine_integration.py`
covers the Phase 8 `Signal -> RiskEngine -> RiskDecision/TradeIntent ->
PostgreSQL/Redis` flow end-to-end. Opt in with:

```powershell
$env:RUN_INTEGRATION_TESTS = "1"
pytest tests/integration
```

Useful variations:

```powershell
pytest -k risk_manager       # run a subset of tests by keyword
pytest --cov-report=html     # generate an HTML coverage report in htmlcov/
ruff check .                 # lint
```

Run `pytest` from the repository root for the current authoritative
passing-test count.

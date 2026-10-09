# Setup & Configuration

## Installation

Requires Python 3.12+ (Windows PowerShell commands shown; adjust for
bash/zsh as needed).

```powershell
cd trading_platform
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e ".[dev]"
```

This installs the `trading_system` package in editable mode plus
`pytest`/`pytest-cov`/`pytest-mock`/`ruff` (the `[dev]` extra defined in
`pyproject.toml`). `requirements.txt` / `requirements-dev.txt` are kept
in sync for Docker and environments that prefer plain
`pip install -r requirements.txt`.

On Windows, Python's `zoneinfo` module has no bundled IANA timezone
database, so `tzdata` is included as a dependency -- no extra step is
needed, but if you ever see `ZoneInfoNotFoundError: 'Asia/Kolkata'`, run
`pip install tzdata`.

To verify the install:

```powershell
python -c "import trading_system; print(trading_system.__version__)"
```

## Configuration

Copy the example environment file and fill in real values:

```powershell
Copy-Item .env.example .env
```

Never commit `.env` (it is already listed in `.gitignore`). All settings
are loaded by `config/settings.py::Settings` (`pydantic-settings`), which
reads the `.env` file automatically and is cached process-wide via
`get_settings()`.

**Every value below is sourced from a single AWS Secrets Manager secret**
(`AWS_SECRETS_NAME`, default `victor/zerodha/kite`), maintained as flat
key-value pairs (never a nested JSON blob) -- see [AWS Secrets Manager](#aws-secrets-manager)
below for exactly how it works and which key maps to which variable.
Kite credentials are the one field group that's *always* overridden
(even when absent from the secret, default `""`); every other field
below falls back to `.env`/its built-in default when its key is absent
from the secret -- useful for local development with no AWS access at all.

| Variable | Default | Purpose |
| --- | --- | --- |
| `KITE_REQUEST_TIMEOUT` | `7` | Kite REST API request timeout, in seconds. |
| `KITE_WS_TIMEOUT` | `30` | Kite WebSocket connect timeout, in seconds. |
| `KITE_RECONNECT_ENABLED` | `true` | Whether `KiteWebSocketClient` reconnects automatically after a disconnect. |
| `KITE_MAX_RECONNECT_ATTEMPTS` | `5` | Maximum bounded reconnect attempts before giving up (never an infinite loop). |
| `POSTGRES_HOST` | `localhost` | PostgreSQL host. |
| `POSTGRES_PORT` | `5432` | PostgreSQL port. |
| `POSTGRES_DATABASE` | `algotrading` | Database name. |
| `POSTGRES_USER` | `postgres` | Database user. |
| `POSTGRES_PASSWORD` | *(empty)* | Database password. |
| `REDIS_HOST` | `localhost` | Redis host. |
| `REDIS_PORT` | `6379` | Redis port. |
| `REDIS_PASSWORD` | *(none)* | Redis password, if required. |
| `REDIS_DB` | `0` | Redis logical database index. |
| `REDIS_SSL` | `false` | Whether to connect to Redis over TLS. |
| `REDIS_SOCKET_TIMEOUT` | `5` | Redis socket read/write timeout, in seconds. |
| `REDIS_CONNECT_TIMEOUT` | `5` | Redis socket connect timeout, in seconds. |
| `AWS_REGION` | `ap-south-1` | AWS region for S3 **and** the secret in Secrets Manager. Read from `.env`/environment only -- never secret-sourced (needed just to find the secret). |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | *(none)* | AWS credentials (omit to fall back to the default boto3 credential chain, e.g. an IAM role). Read from `.env`/environment only -- **never** secret-sourced (circular). |
| `S3_BUCKET_NAME` | *(empty)* | Bucket used for snapshots/archival storage. |
| `AWS_SECRETS_NAME` | `victor/zerodha/kite` | Name of the single AWS Secrets Manager secret holding every other value in this table. Read from `.env`/environment only. |
| `DASHBOARD_ADMIN_TOKEN` | *(empty)* | Environment-only token required to save dashboard risk/scoring settings. Not sourced from AWS Secrets Manager; an empty value disables editing. |
| `OPTIONS_WATCHLIST` | *(empty)* | Comma-separated underlying symbols (index/stock) that trade via options (CE/PE), e.g. `NIFTY,BANKNIFTY`. Runs simultaneously alongside `EQUITY_WATCHLIST` -- see [`resolve_instrument_mode`](#aws-secrets-manager). |
| `EQUITY_WATCHLIST` | *(empty)* | Comma-separated underlying (stock) symbols that trade as direct cash equity instead of an option, e.g. `RELIANCE,TCS`. A symbol can only be in one of the two watchlists (validated at startup). |
| `SCORING_WEIGHT_<PATTERN_NAME>` | *(unset)* | One flat key per pattern/indicator (e.g. `SCORING_WEIGHT_HEAD_AND_SHOULDERS=40`) overrides that entry in `scoring._DEFAULT_WEIGHTS` -- see [`ScoringConfig.from_settings`](#aws-secrets-manager). |
| `SCORING_BUY_THRESHOLD` / `SCORING_SELL_THRESHOLD` | *(unset)* | Overrides `ScoringConfig.buy_score_threshold`/`sell_score_threshold` (both default `50`) when set. |
| `TRADING_MODE` | `paper` | `paper` (safe, simulated) or `live` (real broker authentication/mode; does not itself authorize orders). |
| `ORDER_EXECUTION_ENABLED` | `false` | Explicit Phase 9 order-execution permission. Live submission additionally requires `APP_ENV=production` and `TRADING_MODE=live`. |
| `EXECUTION_ORDER_TYPE` | `MARKET` | Phase 9 `MARKET`, `LIMIT`, `SL`, or `SL-M`. Stop order types require an explicit entry trigger in the intent metadata; no protective stop orders are generated. |
| `EXECUTION_PRODUCT` | `MIS` | Broker-supported `MIS`, `NRML`, or `CNC` product. |
| `EXECUTION_VALIDITY` | `DAY` | Broker-supported `DAY` or `IOC` order validity. |
| `EXECUTION_MAX_QUANTITY` | *(unset)* | Optional hard upper quantity cap, in units. Approved quantity is never resized. |
| `EXECUTION_LOCK_TTL_SECONDS` | `30` | TTL for the per-intent distributed execution lock. |
| `RECONCILIATION_LOOKBACK_HOURS` | `48` | Phase 10: how far back local orders are pulled for broker tag/id matching. |
| `RECONCILIATION_LOCK_TTL_SECONDS` | `120` | Phase 10: TTL for the global `reconciliation:global` distributed lock. |
| `RECONCILIATION_INTERVAL_SECONDS` | *(unset)* | Phase 10: enables `start_periodic_reconciliation()` when set to a positive integer; unset disables periodic reconciliation entirely. |
| `RECONCILIATION_AUTO_SYNC_ORDER_STATUS` / `_ORDER_FILL_QUANTITY` / `_ORDER_AVERAGE_FILL_PRICE` | `true` | Phase 10: per-field order auto-sync policy flags (safe categories only). |
| `RECONCILIATION_AUTO_SYNC_POSITION_QUANTITY` / `_POSITION_PRICE` / `_POSITION_STATUS` | `true` | Phase 10: per-field position auto-sync policy flags (safe categories only; an unsafe quantity increase is never governed by a flag). |
| `LOG_LEVEL` | `INFO` | Root logger level (`DEBUG`, `INFO`, `WARNING`, `ERROR`). |
| `RISK_POLICY_VERSION` | `1` | Recorded on every `RiskDecision`/`TradeIntent` for audit purposes. |
| `RISK_MAX_RISK_PER_TRADE` | *(unset)* | Max rupees at risk on a single trade. `None`/unset = not enforced -- see [Phase 8](phases/phase-08-risk-position.md). |
| `RISK_MAX_DAILY_LOSS` | *(unset)* | Max realized daily loss (rupees) before new trades are blocked. |
| `RISK_MAX_OPEN_POSITIONS` | *(unset)* | Max durably-open PostgreSQL positions. |
| `RISK_MAX_DAILY_TRADES` | *(unset)* | Max `RISK_APPROVED` trade intents per day. |
| `RISK_MAX_EXPOSURE` | *(unset)* | Max total notional (rupees) across open positions + the new trade. |
| `RISK_MAX_QUANTITY` | *(unset)* | Max quantity (units, not lots) for a single trade intent. |
| `RISK_MAX_PREMIUM_EXPOSURE` | *(unset)* | Reserved: max total long-option premium exposure. |
| `RISK_MIN_RISK_REWARD_RATIO` | *(unset)* | Minimum acceptable reward:risk ratio; rejects (`MIN_RISK_REWARD_NOT_MET`) if the actual computed ratio falls short. |
| `RISK_COOLDOWN_SECONDS` | *(unset)* | Minimum seconds between two approved trade intents for the same strategy+instrument. |
| `RISK_TRADING_WINDOW_START` / `RISK_TRADING_WINDOW_END` | *(unset)* | `"HH:MM"` (IST). Both unset = no trading-window restriction enforced. |
| `RISK_DATA_STALENESS_LIMIT_SECONDS` | *(unset)* | Optional staleness limit tighter than Phase 6's default `MarketDataStatus` classification. |
| `RISK_AVOID_FIRST_MINUTES_OF_SESSION` | *(unset)* | **Recommended for NIFTY/BANKNIFTY F&O: `15`.** Rejects trade intents within this many minutes of the 09:15 IST session open (opening-range volatility/gap risk). Unset by default -- unlike the stop-loss defaults, off by default because it depends on wall-clock/session time; opt in explicitly. |
| `RISK_AVOID_LAST_MINUTES_OF_SESSION` | *(unset)* | **Recommended for NIFTY/BANKNIFTY F&O: `15`.** Rejects trade intents within this many minutes of the 15:30 IST session close (closing-volatility/pin risk). Same opt-in rationale as above. |
| `RISK_STOP_LOSS_METHOD` | `ATR` | `ATR` / `PERCENTAGE` / `POINTS`, or empty/unset to disable stop-loss entirely. See [Phase 8](phases/phase-08-risk-position.md). |
| `RISK_STOP_LOSS_ATR_PERIOD` | `14` | ATR lookback period (Wilder's standard). |
| `RISK_STOP_LOSS_ATR_MULTIPLIER` | `1.5` | Stop distance = `ATR * this multiplier`. |
| `RISK_STOP_LOSS_PERCENTAGE` | `0.10` | Used directly for the `PERCENTAGE` method, and as the automatic fallback for `ATR` when candle history is insufficient. |
| `RISK_STOP_LOSS_POINTS` | *(unset)* | Fixed rupee stop distance; only used when `RISK_STOP_LOSS_METHOD=POINTS`. |
| `RISK_STOP_LOSS_CANDLE_INTERVAL_MINUTES` | `5` | Candle interval used for ATR when a signal doesn't specify its own timeframe. |
| `RISK_TARGET_REWARD_RATIO` | `1.5` | Target = entry +/- (stop-loss distance * this ratio); empty/unset disables target calculation entirely. |

## AWS Secrets Manager

**One secret holds every configuration value in the table above**:
`AWS_SECRETS_NAME` (default `victor/zerodha/kite`), maintained as flat
key-value pairs in the Secrets Manager console -- no nested JSON. This
means rotating a database password, tightening a risk limit, or
switching `TRADING_MODE` is a Secrets Manager update + a process
restart -- **no code change, no new image build, no `.env` edit
required.**

Why one secret, not `.env`: the Kite access token expires daily and is
refreshed out-of-band (e.g. a separate scheduled login/automation job
that writes the new token back to this same secret), so the trading
system should always read whatever is currently in AWS rather than a
possibly-stale local `.env` value -- and once that pattern exists for
Kite, every other operational value benefits from the same "no
redeploy" property.

How it works (`config/settings.py::get_settings`):

1. A bootstrap `Settings()` instance is constructed first (to read
   `aws_region`/`aws_secrets_name` from `.env`/defaults -- these two,
   plus the raw `aws_access_key_id`/`aws_secret_access_key`, are the
   *only* fields that can never be secret-sourced, since they're needed
   to find/authenticate to the secret itself).
2. `_fetch_config_secret_from_aws(secret_name, region)` calls
   `boto3.client("secretsmanager", region_name=region).get_secret_value(...)`
   using the **default AWS credential provider chain** (environment
   variables, shared config/credentials file, or an attached IAM role --
   never explicit keys passed in code).
3. `get_settings()` maps secret keys onto `Settings` fields: for most
   fields, the key is simply the field's attribute name upper-cased
   (matching `pydantic-settings`' own `.env` convention), e.g.
   `postgres_host` -> `POSTGRES_HOST`, `risk_max_risk_per_trade` ->
   `RISK_MAX_RISK_PER_TRADE`. Two exceptions: Kite credentials use the
   shorter `API_KEY`/`API_SECRET`/`ACCESS_TOKEN` keys and **always**
   override (even when absent, default `""` -- AWS is their sole source
   of truth); per-pattern scoring weights use one
   `SCORING_WEIGHT_<PATTERN_NAME>` key each (e.g.
   `SCORING_WEIGHT_HEAD_AND_SHOULDERS=40`) since there's no single dict
   value to store.
4. Any other key **absent** from the secret falls back to `.env`/the
   field's built-in default -- a secret containing only
   `POSTGRES_PASSWORD` still works, and local development never needs
   an AWS secret at all (aside from Kite credentials, which are
   unconditional).
5. If the AWS call fails (missing permissions, secret not found, no
   network) or the secret body isn't valid JSON, `get_settings()` raises
   `ConfigurationError` -- the application fails fast at startup instead
   of silently running with missing/stale credentials.

Expected keys (all optional except `API_KEY`/`API_SECRET`/`ACCESS_TOKEN`,
all flat plain strings -- maintain this secret as simple key/value rows
in the Secrets Manager console, no nested JSON needed):

```json
{
  "API_KEY": "your-kite-api-key",
  "API_SECRET": "your-kite-api-secret",
  "ACCESS_TOKEN": "your-kite-access-token",
  "POSTGRES_HOST": "prod-db.internal",
  "POSTGRES_PORT": "5432",
  "POSTGRES_DATABASE": "algotrading",
  "POSTGRES_USER": "trading_app",
  "POSTGRES_PASSWORD": "...",
  "REDIS_HOST": "prod-cache.internal",
  "REDIS_PORT": "6379",
  "REDIS_PASSWORD": "...",
  "REDIS_SSL": "true",
  "TRADING_MODE": "paper",
  "OPTIONS_WATCHLIST": "NIFTY,BANKNIFTY",
  "EQUITY_WATCHLIST": "RELIANCE,TCS",
  "RISK_MAX_RISK_PER_TRADE": "1000",
  "RISK_MIN_RISK_REWARD_RATIO": "1.5",
  "RISK_TRADING_WINDOW_START": "09:20",
  "RISK_TRADING_WINDOW_END": "15:00",
  "RISK_STOP_LOSS_METHOD": "ATR",
  "RISK_TARGET_REWARD_RATIO": "1.5",
  "SCORING_WEIGHT_HEAD_AND_SHOULDERS": "40",
  "SCORING_WEIGHT_ENGULFING": "30",
  "SCORING_BUY_THRESHOLD": "55",
  "SCORING_SELL_THRESHOLD": "55"
}
```

Two fields deserve special mention:

- `OPTIONS_WATCHLIST` / `EQUITY_WATCHLIST` are both comma-separated
  strings that **run simultaneously** in the same process -- which one
  a given underlying belongs to decides whether its `StrategyEngine`
  buys an option (`OPTIONS_WATCHLIST`, CE/PE, never the underlying
  itself) or trades the underlying/cash-equity instrument directly
  (`EQUITY_WATCHLIST`). A symbol can only be in one of the two lists --
  `Settings` raises `ConfigurationError` at startup if the same symbol
  appears in both. See `strategy/instrument_mode.py::resolve_instrument_mode`
  (an underlying in neither list safely defaults to `OPTIONS`) and
  `StrategyEngine`'s `instrument_mode` constructor argument (built by
  calling `resolve_instrument_mode` once per underlying, at the same
  point you construct the engine).
- `SCORING_WEIGHT_<PATTERN_NAME>` is a **partial override**, one flat
  key per pattern/indicator name from `strategy/scoring.py::_DEFAULT_WEIGHTS`
  -- only the patterns you add a key for are changed; everything else
  keeps its default weight. Build the final `ScoringConfig` via
  `ScoringConfig.from_settings(settings)` rather than constructing it
  directly, so these overrides take effect.

Unit tests never call AWS for real -- `tests/unit/test_settings.py`
monkeypatches `config.settings._fetch_config_secret_from_aws`.

## Trading modes

- `TRADING_MODE=paper` (**default**) -- all orders are simulated
  in-memory via `PaperExecutionHandler`. No real orders are ever placed,
  and no Kite order-placement credentials are required.
- `TRADING_MODE=live` -- selects live broker authentication/mode, but
  **does not enable order submission**. `ORDER_EXECUTION_ENABLED=true`
  and `APP_ENV=production` are additionally required by Phase 9, along
  with valid `KITE_API_KEY`/`KITE_ACCESS_TOKEN` from AWS Secrets Manager.
  The legacy signal-based `LiveExecutionHandler` fails closed; only a
  risk-approved intent through `OrderExecutionEngine` can reach the
  `KiteBrokerAdapter` after all gates pass.

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

AWS Secrets Manager supplies only the credentials and infrastructure keys
listed in [AWS Secrets Manager](#aws-secrets-manager). Watchlists, trading
mode, risk limits, and scoring settings are managed in PostgreSQL and loaded
from the dashboard settings table at process startup. Order-execution
permission and execution parameters are configured through `.env` or the
environment and are intentionally not dashboard-editable. Other
local/runtime settings use `.env` or their documented defaults. There is
no AWS override for database-managed strategy or scoring values.

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
| `AWS_SECRETS_NAME` | `victor/zerodha/kite` | AWS Secrets Manager secret containing only the allow-listed credentials and infrastructure settings. Read from `.env`/environment so the secret can be located. |
| `DASHBOARD_ADMIN_TOKEN` | *(empty)* | AWS secret key for authorizing dashboard configuration writes; an empty value disables editing. |
| `OPTIONS_WATCHLIST` | *(empty)* | PostgreSQL dashboard setting: synchronized underlying EQ instrument-token IDs. The selected index/stock EQ candles generate signals; the corresponding CE/PE contract is traded. |
| `EQUITY_WATCHLIST` | *(empty)* | PostgreSQL dashboard setting: synchronized tradable cash-EQ instrument-token IDs. Each selected instrument generates signals and is itself traded. |
| `SCORING_WEIGHT_<PATTERN_NAME>` | *(unset)* | PostgreSQL dashboard setting per pattern/indicator; overrides that entry in `scoring._DEFAULT_WEIGHTS`. |
| `SCORING_BUY_THRESHOLD` / `SCORING_SELL_THRESHOLD` | *(unset)* | PostgreSQL dashboard settings for `ScoringConfig` buy/sell thresholds (defaults `50`). |
| `TRADING_MODE` | `paper` | PostgreSQL dashboard setting: `paper` (simulated) or `live` (live market-data mode; does not itself authorize orders). |
| `ORDER_EXECUTION_ENABLED` | `false` | Environment/`.env` only; not dashboard-editable. Live submission additionally requires `APP_ENV=production` and `TRADING_MODE=live`. |
| `EXECUTION_ORDER_TYPE` | `MARKET` | Environment/`.env` only; Phase 9 `MARKET`, `LIMIT`, `SL`, or `SL-M`. Stop order types require an explicit entry trigger in the intent metadata; no protective stop orders are generated. |
| `EXECUTION_PRODUCT` | `MIS` | Environment/`.env` only; broker-supported `MIS`, `NRML`, or `CNC` product. |
| `EXECUTION_VALIDITY` | `DAY` | Environment/`.env` only; broker-supported `DAY` or `IOC` order validity. |
| `EXECUTION_MAX_QUANTITY` | *(unset)* | Environment/`.env` only; optional hard upper quantity cap, in units. Approved quantity is never resized. |
| `EXECUTION_LOCK_TTL_SECONDS` | `30` | Environment/`.env` only; TTL for the per-intent distributed execution lock. |
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

`AWS_SECRETS_NAME` (default `victor/zerodha/kite`) contains the explicit
allow-list below as flat key-value pairs. AWS supplies credentials and
infrastructure only. Watchlists, trading mode, risk policy, and scoring
settings are edited in the dashboard and persisted in PostgreSQL; these
keys are not read from AWS. Dashboard rows override corresponding local
settings when present. The AWS SDK uses its default credential provider
chain (for example, an attached IAM role); raw AWS access keys are not
read from this secret.

```json
{
  "API_KEY": "your-kite-api-key",
  "API_SECRET": "your-kite-api-secret",
  "USER_NAME": "your-broker-user",
  "PASSWORD": "...",
  "TOTP": "...",
  "ACCESS_TOKEN": "your-kite-access-token",
  "GIT_TOKEN": "...",
  "GIT_USER": "...",
  "POSTGRES_HOST": "prod-db.internal",
  "POSTGRES_PORT": "5432",
  "POSTGRES_DATABASE": "algotrading",
  "POSTGRES_USER": "trading_app",
  "POSTGRES_PASSWORD": "...",
  "REDIS_HOST": "prod-cache.internal",
  "REDIS_PORT": "6379",
  "REDIS_PASSWORD": "...",
  "REDIS_SSL": "true",
  "DASHBOARD_ADMIN_TOKEN": "..."
}
```

No other secret keys are applied to `Settings`. In particular, AWS keys
such as `OPTIONS_WATCHLIST`, `EQUITY_WATCHLIST`, `TRADING_MODE`,
`RISK_*`, or `SCORING_*` are ignored. Configure scoring weights in the
dashboard with `SCORING_WEIGHT_<PATTERN_NAME>`; unspecified weights use
the strategy defaults. If the AWS call fails or its body is not valid
JSON, `get_settings()` raises `ConfigurationError` instead of silently
using missing credentials.

For an `OPTIONS_WATCHLIST` underlying, the cash/index instrument is the
signal source: scoring analyzes its candles and price, then bullish
signals select the corresponding CE and bearish signals the PE. The
selected option's own price is used for risk and execution. For
`EQUITY_WATCHLIST`, the same tradable cash-equity instrument is used for
analysis and execution. The dashboard lets you search synchronized EQ
instruments and displays the selected name, symbol, exchange, segment,
and token; only the exact instrument token is stored in the watchlist.
This removes symbol-alias ambiguity, but the token must still exist in
the current synchronized instrument master. Indices are signal-only and
cannot be in the equity execution path. OPTIONS entries also require
synchronized future CE and PE contracts that meet the configured minimum
days-to-expiry rule.

Before deploying token-based watchlists to a database with existing
symbol values, synchronize NSE and NFO instruments using the currently
running application version, then run the migration script first in
dry-run mode and review every mapping:

```powershell
python scripts/migrate_watchlists_to_tokens.py
python scripts/migrate_watchlists_to_tokens.py --apply
```

The migration converts only unique matches and refuses to apply any
changes if a value is missing, ambiguous, unsuitable for its watchlist,
or assigned to both modes. Apply the migration successfully before
starting the token-based application version. Fresh databases also need
their initial symbol-based default converted by this step after instrument
synchronization.

Unit tests monkeypatch the AWS fetch function and do not access AWS.

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

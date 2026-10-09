# Phase 1 — Configuration, Logging & Application Foundation

Phase 1 delivers the foundational layer only: configuration, logging,
and a minimal application lifecycle with a health check. It intentionally
does **not** touch PostgreSQL, Redis, the Kite API/WebSocket, or any
trading logic -- those are wired into the codebase already for later
phases (see `trading_system/engine.py`) but are not part of the Phase 1
startup path (`trading_system/main.py` currently uses
`trading_system/application.py::Application` instead of `TradingEngine`).

**What Phase 1 provides:**

- `config/settings.py::Settings` -- typed, environment-driven
  configuration (`app_env`, `trading_mode`, `log_level`, plus Kite/
  PostgreSQL/Redis/AWS fields for later phases), with clear validation
  errors instead of silent misconfiguration.
- `trading_system/utils/logger.py` -- centralized structured JSON logging
  to stdout and a rotating log file, with automatic redaction of
  sensitive fields.
- `trading_system/health.py` -- a small, extensible health-check registry
  with only the application-level check registered for now.
- `trading_system/application.py::Application` -- a `start()`/`stop()`
  lifecycle with clear extension points (`TODO` markers) for Phase 2+.
- `trading_system/main.py` -- loads configuration, configures logging,
  starts the application, handles `SIGINT`/`SIGTERM` for graceful
  shutdown, and returns a non-zero exit code on configuration failure.

#### Installation

```powershell
cd trading_platform
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e ".[dev]"
```

#### Configuration

Copy `.env.example` to `.env` and fill in values as needed
(everything has a safe default -- Phase 1 runs with no `.env` at all).
`.env` is already listed in `.gitignore` and must never be committed;
only `.env.example` (placeholders only) is tracked.

```powershell
Copy-Item .env.example .env
```

#### Running

```powershell
python -m trading_system.main
```

Starts the application, logs the environment and trading mode, runs the
health check, and then waits for `Ctrl+C` (`SIGINT`) or `SIGTERM` to shut
down gracefully.

#### Trading mode

`TRADING_MODE` defaults to **`paper`** and is always the safe fallback
when unset. `live` is accepted as a valid configuration value but **no
live trading is implemented yet** -- Phase 1 never places any order,
paper or otherwise. Setting an unsupported value (e.g.
`TRADING_MODE=production`) fails fast with a `ConfigurationError`.

#### Testing

```powershell
pytest
```

Phase 1 tests live in `tests/unit/test_settings.py`,
`tests/unit/test_logger.py`, `tests/unit/test_health.py`,
`tests/unit/test_application.py`, and `tests/unit/test_main.py`. None of
them connect to real AWS, PostgreSQL, Redis, or Kite.

See the original prompt: [specs/01-phase1.md](../specs/01-phase1.md).

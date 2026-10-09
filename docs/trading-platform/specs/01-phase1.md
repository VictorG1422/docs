# Phase 1 — Configuration, Logging & Application Foundation

We are now implementing **PHASE 1 only** of the algorithmic trading system.

The repository and folder structure already exist.

## **Objective**
Build and verify the application's foundational layer:

1. Configuration management
2. Environment variable handling
3. Configuration validation
4. Trading-mode safety
5. Centralized logging
6. Application startup
7. Graceful shutdown
8. Basic health-check framework

Do NOT implement database logic, Redis logic, Kite WebSocket logic, trading strategy, order execution, risk management, or live trading in this phase.

---

# **1. Inspect Existing Repository First**
Before changing anything:

- Inspect the existing folder structure.
- Inspect existing Python files.
- Reuse existing files where appropriate.
- Do not unnecessarily create duplicate modules.
- Do not rewrite working code.
- Follow the existing architecture.

After inspection, briefly explain which existing files you will modify/create and why.

Then implement Phase 1.

---

# **2. Configuration Management**
Implement centralized configuration using environment variables.

Use the existing configuration module if one already exists.

Configuration should include at least:

```
KITE_API_KEY
KITE_API_SECRET
KITE_ACCESS_TOKEN

POSTGRES_HOST
POSTGRES_PORT
POSTGRES_DATABASE
POSTGRES_USER
POSTGRES_PASSWORD

REDIS_HOST
REDIS_PORT
REDIS_PASSWORD

AWS_REGION
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
S3_BUCKET_NAME

TRADING_MODE
LOG_LEVEL
APP_ENV
```

Do not hardcode credentials.

Do not put secrets directly into Python source code.

---

# **3. .env Support**
Use a standard environment configuration approach.

If `.env` loading is used, use an appropriate Python library such as `python-dotenv` or the project's existing configuration dependency.

Ensure:

```
.env
```
is ignored by Git.

The repository should contain:

```
.env.example
```
with placeholder values only.

Example:

```
KITE_API_KEY=
KITE_API_SECRET=
KITE_ACCESS_TOKEN=

POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DATABASE=trading
POSTGRES_USER=
POSTGRES_PASSWORD=

REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_BUCKET_NAME=

TRADING_MODE=paper
LOG_LEVEL=INFO
APP_ENV=development
```

Do not put real credentials anywhere.

---

# **4. Configuration Validation**
Configuration should be validated when the application starts.

Separate:

### **Required configuration**
Configuration required for the selected application mode.

### **Optional configuration**
Configuration that is not required until the relevant component is enabled.

For example:

- PostgreSQL credentials may not need to be validated before PostgreSQL initialization is implemented.
- Redis credentials may not need to be validated before Redis initialization.
- AWS credentials may not need to be validated before S3 initialization.

Do not make Phase 1 unnecessarily impossible to run.

However, if a configuration value is required for the current startup process, fail clearly with an understandable error.

---

# **5. Trading Mode Safety**
Implement:

```
TRADING_MODE=paper
```
as the default.

Supported modes:

```
paper
live
```

For this phase, **live trading must not be implemented**.

Add a safety mechanism so that:

```
paper
```
is always the default when `TRADING_MODE` is missing.

If an unsupported value is supplied, fail with a clear configuration error.

Example invalid:

```
TRADING_MODE=production
```

Allowed:

```
TRADING_MODE=paper
TRADING_MODE=live
```

Do not make live mode easier to accidentally activate.

If useful, expose a configuration property such as:

```
settings.is_paper_trading
settings.is_live_trading
```

---

# **6. Strongly Typed Configuration**
Use typed configuration where practical.

Prefer a configuration object such as:

```
Settings
```
rather than accessing:

```
os.getenv(...)
```
throughout the application.

The rest of the application should eventually be able to do something conceptually like:

```
settings.trading_mode
settings.log_level
settings.kite_api_key
```
instead of directly reading environment variables.

---

# **7. Logging System**
Implement centralized logging.

Create or update the existing logger module.

Requirements:

### **Log levels**
Support:

```
DEBUG
INFO
WARNING
ERROR
CRITICAL
```

The level should come from:

```
LOG_LEVEL
```

Default:

```
INFO
```

---

# **8. Logging Format**
Logs should be useful for a trading system.

Include at least:

```
timestamp
log level
logger/component name
message
```

Example conceptually:

```
2026-08-11 09:00:01 | INFO | application | Application starting
2026-08-11 09:00:01 | INFO | configuration | Configuration loaded
2026-08-11 09:00:01 | INFO | application | Trading mode: PAPER
```

Do NOT log:

- API keys
- API secrets
- access tokens
- database passwords
- AWS secrets
- Redis passwords

---

# **9. Logger Usage**
Do not use random `print()` statements for application events.

Use the centralized logger.

Avoid creating a new logging configuration independently in every module.

There should be one consistent logging setup.

Modules should obtain a logger appropriate to their component/module.

---

# **10. File Logging**
If appropriate for the existing architecture, support file logging.

Logs should go into:

```
logs/
```

Do not commit generated log files.

Update `.gitignore` accordingly.

Use log rotation if implementing file logging so logs cannot grow indefinitely.

Do not over-engineer this.

---

# **11. Application Startup**
Implement/update:

```
src/trading_system/main.py
```

The application should have a clear startup sequence.

Conceptually:

```
main()
  ↓
Load configuration
  ↓
Initialize logging
  ↓
Validate configuration
  ↓
Log application environment
  ↓
Log trading mode
  ↓
Start application
```

For Phase 1, do not initialize PostgreSQL, Redis, Kite, or other external services yet.

Use placeholders for future phases.

---

# **12. Application Lifecycle**
Create a clean application lifecycle.

Conceptually:

```
application.start()
application.stop()
```

Startup should:

- initialize required Phase 1 components
- log successful startup

Shutdown should:

- stop active components
- release resources where applicable
- log shutdown

The design should make it easy to add:

```
PostgreSQL
Redis
Kite
WebSocket
Strategy Engine
```
in later phases.

---

# **13. Graceful Shutdown**
Handle:

```
SIGINT
SIGTERM
```

For example:

- Ctrl+C
- EC2 process termination
- container termination later

The application should not simply crash without cleanup.

Expected flow:

```
SIGTERM
   ↓
Shutdown requested
   ↓
Stop application components
   ↓
Log shutdown
   ↓
Exit
```

Do not introduce unnecessary asynchronous complexity unless required.

---

# **14. Health Check Framework**
Create a simple health-check abstraction.

It should eventually support checks for:

```
Application
PostgreSQL
Redis
Kite
S3
```

But in Phase 1 only implement the application-level health check.

Example conceptual result:

```
{
  "status": "healthy",
  "application": "trading_system"
}
```

Do not create a web server/API just for this unless the existing project already requires one.

A simple callable health-check function/class is sufficient for now.

---

# **15. Error Handling**
Configuration errors should be clearly distinguishable from unexpected application errors.

Example:

```
ConfigurationError
```
should be used where appropriate.

Startup should:

1. Log the error.
2. Avoid exposing secrets.
3. Exit with a non-zero status code.

Do not silently continue with invalid configuration.

---

# **16. Testing**
Create/update pytest tests for Phase 1.

At minimum test:

### **Configuration**

- Default trading mode is paper.
- Valid `paper` mode works.
- Valid `live` mode is accepted as a configuration value.
- Invalid trading mode fails.
- Environment variables are loaded correctly.
- Required configuration validation works.
- Defaults work correctly.

### **Logging**

- Logger initializes correctly.
- Configured log level is respected.
- Sensitive configuration values are not written to logs.

### **Application**

- Application starts successfully with valid configuration.
- Application shuts down cleanly.
- Configuration failure causes startup failure.

Do not connect to real AWS, PostgreSQL, Redis, or Kite during these tests.

Use mocks where necessary.

---

# **17. Dependencies**
Only add dependencies that are actually required for Phase 1.

Do not add:

- database libraries
- Redis clients
- Kite clients
- AWS SDK
- trading libraries

unless they are already required by existing code.

Keep dependencies minimal.

Update:

```
requirements.txt
```
or the project's existing dependency configuration appropriately.

---

# **18. README Update**
Update the README with a Phase 1 section containing:

### **Installation**
How to create/activate the Python environment and install dependencies.

### **Configuration**
Explain:

```
.env
.env.example
```

### **Running**
Show the command to start the application.

### **Trading mode**
Clearly state:

```
Default = paper
```
and explain that live trading is not implemented yet.

### **Testing**
Show how to run:

```
pytest
```

---

# **19. Security Requirements**
Before finishing, verify:

- `.env` is in `.gitignore`.
- No credentials are hardcoded.
- No credentials are printed.
- No credentials appear in README.
- No credentials appear in tests.
- No access tokens are committed.
- No generated logs are committed.

Search the repository for obvious credential leaks before completing the task.

---

# **20. Definition of Done**
Phase 1 is complete only when:

- Configuration class works.
- `.env.example` exists.
- `.env` is ignored.
- Configuration validation works.
- Paper mode is the default.
- Live mode cannot accidentally activate by default.
- Centralized logging works.
- Sensitive values are not logged.
- Application starts successfully.
- Application shuts down gracefully.
- SIGINT/SIGTERM are handled.
- Basic health check exists.
- Unit tests pass.
- README is updated.
- No unnecessary dependencies were added.
- No trading logic was implemented.
- No real broker/API calls are made.

---

# **IMPORTANT DEVELOPMENT RULE**
Implement **PHASE 1 ONLY**.

Do not start Phase 2.

Do not implement:

- PostgreSQL
- Redis
- Kite API
- WebSocket
- instruments
- option chain
- strategy
- signals
- orders
- positions
- SL/Target
- live trading

At the end, report:

1. Files created.
2. Files modified.
3. Dependencies added.
4. Tests created.
5. Test results.
6. How to run the application.
7. Any architectural decisions made.
8. Any TODOs remaining for Phase 2.

Do not proceed to the next phase without my approval.

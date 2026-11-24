---
name: kite-api-reference
description: Document Kite Connect wrapper modules - function reference tables, exception hierarchy, rate limits, docstring format. Use when writing or updating API reference docs for code that calls KiteConnect.
---

# Kite Connect API Reference Docs

Per-function doc block: purpose (one line), underlying `kite.<method>()` call, parameter table (name/type/required/description), return type/shape, exceptions raised, example call.

Exception hierarchy to reference: `KiteException` -> `GeneralException`, `TokenException` (auth), `PermissionException` (subscription), `OrderException` (rejected order), `InputException` (bad params), `DataException` (no data/bad symbol), `NetworkException`, `TwoFAException`.

Rate limits to note in module docstrings: historical data ~3 req/sec, quote/LTP ~1 req/sec, orders ~10 req/sec practical, WebSocket ~3000 tokens/connection.

Docstring format (Google style): Args/Returns/Raises, with Raises naming the specific KiteException subtype and trigger condition.

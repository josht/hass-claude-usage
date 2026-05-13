# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Home Assistant custom integration (`hass_claude_usage`) that polls Anthropic's OAuth-only `/api/oauth/usage` endpoint and exposes usage metrics as HA sensors. There is no application server, no database, no test suite — just the integration package under `custom_components/hass_claude_usage/` plus a bundled dashboard YAML.

`AGENTS.md` is the authoritative design doc: it documents OAuth specifics, the usage endpoint response shape, every architectural decision, and the manual test checklist. Read it before non-trivial changes.

## Commands

```bash
# Formatting / linting (no test suite exists)
black custom_components/hass_claude_usage/
isort custom_components/hass_claude_usage/
ruff check --fix custom_components/hass_claude_usage/

# Pre-commit runs all three plus standard hooks
pre-commit install
pre-commit run --all-files

# Deploy to a local HA install (edit DEST in the script for your host)
./deploy.sh
```

There is no Python test runner. Verification is the manual HA checklist in AGENTS.md plus the GitHub Actions in `.github/workflows/` (`hass-validate.yml` runs hassfest; `validate.yml` runs HACS validation).

## Architecture at a glance

Five Python files under `custom_components/hass_claude_usage/`, wired together via Home Assistant's standard integration plumbing:

- `__init__.py` — entry-point setup, `ClaudeUsageCoordinator` (a `DataUpdateCoordinator` subclass), token refresh, and `_parse_usage()` which flattens the API JSON into a dict keyed by sensor name.
- `config_flow.py` — OAuth 2.0 + PKCE flow. Single user-facing step: show the authorize URL, accept the pasted `code#state` string, exchange for tokens. `async_step_reauth_confirm` reuses the same code path via `_build_oauth_url()`. PKCE verifier/state live on the flow instance — `_exchange_code()` validates state unconditionally.
- `sensor.py` — one `ClaudeUsageSensor` entity per `SENSOR_DEFINITIONS` row. All sensors share one device. `available` returns False when the key is absent from coordinator data (free-tier accounts get an empty API response and their sensors correctly show unavailable). The `api_error` sensor is special-cased: it is always available and reports `coordinator.last_update_success` as 0/1.
- `const.py` — every OAuth/API URL, config key, and the `SENSOR_DEFINITIONS` table. Adding a sensor = adding a tuple here plus a translation key.
- `manifest.json` / `strings.json` / `translations/` — HA metadata and UI strings.

Data flow per poll: `Coordinator._async_update_data()` → `_ensure_valid_token()` (refresh if <60s to expiry) → GET `/api/oauth/usage` with `anthropic-beta: oauth-2025-04-20` → `_parse_usage()` returns a flat dict → each `ClaudeUsageSensor` reads its key out of `coordinator.data`.

## Error handling contract

The distinction matters and is easy to get wrong:

- **Auth problems → `ConfigEntryAuthFailed`** (HA shows the reauth UI). This includes: 401 on the usage fetch, missing refresh token, refresh request returning non-OK, refresh response missing `access_token`.
- **Transient / network → `UpdateFailed`** (HA retries on next interval, sensors show unavailable).

Do not raise `UpdateFailed` for auth issues — it silently retries forever and the user never sees the reauth prompt.

## OAuth gotchas

- The redirect URI returns the code as `code#state` (fragment). `_exchange_code` splits on `#`; both halves must be present and `state` must match the value generated when the URL was built.
- PKCE verifier and state are generated lazily inside `_build_oauth_url()` and persisted on the flow instance — do not regenerate them between showing the URL and exchanging the code.
- Token refresh uses the same `OAUTH_TOKEN_URL` as initial exchange; the response may or may not include a new `refresh_token` (fall back to the existing one).
- Anthropic rate-limits `/api/oauth/usage` aggressively. A few dozen requests in a minute can trigger a ~24h backoff that also affects Claude Code and claude.ai. Keep `DEFAULT_UPDATE_INTERVAL` at 300s and bound the options-flow input at 60–3600.

## Project conventions worth keeping

From `AGENTS.md`:
- Atomic commits — one logical change each.
- Three similar lines beats a premature abstraction. Don't DRY for its own sake.
- No "manufactured by Anthropic" framing in device info — Anthropic provides the API, the maintainer maintains the integration.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`zpdatafetch` is a Python library and set of four CLI tools for fetching Zwift-related cycling data. Despite the package name, it ships **four independent packages**, each with its own CLI entry point (defined in `pyproject.toml` `[project.scripts]`):

| Package        | CLI      | Data source                    | Auth model                                              |
| -------------- | -------- | ------------------------------ | ------------------------------------------------------- |
| `zdatafetch`   | `zdata`  | Zwift's unofficial mobile API  | OAuth2 password grant (`zdatafetch/auth.py: ZwiftAuth`) |
| `zpdatafetch`  | `zpdata` | ZwiftPower.com                 | HTML form login + scraped `cache3` JSON endpoints       |
| `zrdatafetch`  | `zrdata` | Zwiftracing.app API            | `authorization` header; rate-limited (standard/premium) |
| `zsdatafetch`  | `zsdata` | Zwift Status (Statuspage.io v2)| None — public API                                       |

A fifth top-level package, `shared/`, holds base classes and utilities used by all four (see below).

## Commands

Uses [`uv`](https://astral.sh/uv) for the toolchain. A `justfile` provides shortcuts.

```sh
uv sync --all-groups            # install all deps (dev + optional extras)

uv run pytest                   # run the full test suite
uv run pytest -m "not live"     # skip tests that make real network calls (see Testing)
uv run pytest test/test_zpdatafetch/test_zp.py            # one file
uv run pytest test/test_zpdatafetch/test_zp.py::test_name # one test
uv run pytest --cov=zpdatafetch --cov-report=html         # coverage

uv run ruff check src test      # lint
uv run ruff format src          # format
uv run ty check src test        # type check (Astral's `ty`, NOT mypy/pyright)

just                            # runs ruff + ty + pytest (default recipe)
uv build                        # build sdist + wheel into dist/
```

Note: `just` recipes source `.venv/bin/activate` directly rather than using `uv run`, so they require a `.venv` (created by `uv sync`).

## Import layout — read this before writing imports

`src/` is a flat package root (`pyproject.toml` sets `package-dir = {"" = "src"}` and `pytest` sets `pythonpath = ["src"]`). Consequences:

- The four data packages **and** `shared` are all top-level. Imports look like `from shared.http_client import ...`, `from zpdatafetch.zp import ZP` — `shared` is NOT namespaced under any package.
- Running a module directly needs `src` on the path: `uv run src/zpdatafetch/zp.py` (or `PYTHONPATH=src python ...`). Running `src/zpdatafetch/zp.py` prints a `200` if ZwiftPower login works — a quick credential smoke test.

## Architecture

### The `shared/` package (start here)

All four packages are built on abstractions in `shared/`:

- `http_client.py` — `BaseHTTPClient` / `AsyncBaseHTTPClient` (ABCs) using the **template-method pattern**: subclasses override `_create_client`, `_before_request`, `_after_request`, `_on_close`. Also `fetch_with_retry_sync` / `fetch_with_retry_async` — exponential backoff that retries on connection errors, timeouts, and 5xx, but re-raises 4xx immediately.
- `config.py` — `BaseConfig` (ABC) wrapping the system **keyring**. Subclasses set the keyring domain and credential fields. `_test_domain_override` swaps in a test domain (used by conftest).
- `cli.py` — `create_base_parser()` builds the argparse parser shared by every CLI (`-v`/`-vv`, `--log-file`, `--raw`, `--json`, `--noaction`, `--sync`, `cmd`, `id...`), plus shared validation/dispatch helpers.
- `exceptions.py` (`NetworkError`, `AuthenticationError`, `ConfigError`), `validation.py` (ID validation, datetime→epoch parsing), `json_helpers.py`, `logging.py`, `error_helpers.py`.

### Sync + async unification (anyio)

Every fetcher class exposes **both** a sync `fetch()` and an async `afetch()`. `fetch()` internally runs the async parallel path via `asyncio.run(self._fetch_parallel(...))`; it raises if called from within a running event loop (use `afetch()` there). Async code uses [`anyio`](https://anyio.readthedocs.io/), so it runs under **asyncio (default) or trio** — the test suite parametrizes both via the `anyio_backend` fixture.

`ClassName.set_sync_mode(True)` (class-level flag) forces a separate sequential, non-parallel code path (`_fetch_sequential`) — this backs the CLI `--sync` flag and is the intended way to debug fetches.

### Fetchers vs. dataclasses (naming convention)

Within each package, files split by role:

- `*fetch.py` (e.g. `zpcyclistfetch.py`, `zrriderfetch.py`) — **fetcher** classes that do the network work, inherit the base data object, and return native dataclass objects.
- other `zp*.py` / `zr*.py` (e.g. `zpcyclist.py`, `zrrider.py`) — **pure dataclasses** (data containers, no fetch logic; often with `from_dict`).

Public names in `__init__.py` include **backwards-compatibility aliases**: `Cyclist = ZPCyclistFetch`, `Result = ZPResultFetch`, `Team`, `Signup`, `League`, `Primes`, `Sprints`, and the old `Async*` names are aliased to the unified classes (`AsyncCyclist is Cyclist`). Keep these aliases working when refactoring.

### Session sharing / connection pooling

Driver/session classes — `ZP` / `AsyncZP` (ZwiftPower), `ZR_obj` / `AsyncZR_obj` (Zwiftracing) — own the `httpx2` client and its connection pool. Multiple fetchers can share one session (avoiding repeat logins) via `set_session()` (async) / `set_zp_session()` / `set_zr_session()` (sync), or by constructing with `shared_client=True`. Always release class-level shared clients with `close_shared_session()`.

### Credentials (keyring domains)

Set interactively with `<tool> config`, or directly with `keyring set <domain> <field>`:

- `zpdatafetch`: `username`, `password`
- `zrdatafetch`: `authorization`
- `zdatafetch`: Zwift account credentials
- `zsdata`: none required

## Testing

- Tests live under `test/`, mirroring the packages: `test_zpdatafetch/`, `test_zrdatafetch/`, `test_zdatafetch/`, `test_zsdatafetch/`, `test_shared/`.
- The root `test/conftest.py` autouse fixture installs a `PlaintextKeyring` and fake test credentials, so tests never touch real secrets.
- `test/live/` contains tests marked `@pytest.mark.live` that make **real API calls**. Skip them with `-m "not live"`.
- Async tests run twice (asyncio + trio) via the `anyio_backend` fixture.
- Network is stubbed with `httpx2.MockTransport` (the non-live suite makes no real HTTP calls); the ZwiftPower login-form parse is covered against the `test/fixtures/login_page.html` fixture.

## Conventions

- **Requires Python ≥ 3.14** (including free-threaded 3.14t). Use modern syntax (`dict[str, X]`, `X | None`), not `typing.Dict`/`Optional`. The `BaseExceptionGroup`/`exceptiongroup` shims for older Pythons in the fetcher modules are now dead code and safe to drop.
- Ruff: line length **80**, **2-space** indent, **single quotes**. Lint set is `E,W,F,I,UP,ANN` — full type annotations are enforced (per-file `ANN`/`E501` exceptions are declared in `pyproject.toml`).
- Google-style docstrings.

## Release

Version lives in `pyproject.toml`. Push a tag matching `release_*` (e.g. `release_$(git rev-parse --short HEAD)`) to `main`. CI runs the test matrix on Linux/macOS/Windows across all Python versions; `build_publish.yml` then triggers off `linux-test` completion, verifies the other OS workflows passed, and publishes to PyPI via trusted publishing. See `BUILD.md` for the full flow.

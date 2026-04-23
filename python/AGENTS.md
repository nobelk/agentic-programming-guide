# AGENTS.md

Instructions for AI coding agents working in this repository. Read fully before making changes.

## Ground Rules

- **uv** is the only package/project manager. No `pip`, `poetry`, `pipenv`, `conda`, or `requirements.txt`.
- **TDD is mandatory.** Red → Green → Refactor. No production code without a failing test first, except trivial boilerplate or pure refactors.
- **Match existing conventions** in this repo even when they differ from these defaults. Consistency wins.
- **Commit in the green state.** All tests and checks must pass.

## Toolchain

| Concern       | Tool                                |
|---------------|-------------------------------------|
| Deps & envs   | `uv`                                |
| Lint + format | `ruff check` / `ruff format`        |
| Types         | `mypy --strict` (or pyright)        |
| Tests         | `pytest` + `pytest-cov` + `hypothesis` |
| Security      | `pip-audit`                         |

## Daily Commands

```bash
uv sync --frozen --all-extras --dev     # install
uv add <pkg>                            # runtime dep
uv add --dev <pkg>                      # dev dep
uv run pytest -x --ff                   # test (fail fast, failed-first)
uv run ruff check --fix . && uv run ruff format .
uv run mypy src
uv run pip-audit
```

Never edit `uv.lock` by hand. Always commit it.

## Project Layout

Use the `src/` layout. Keep `domain/` free of I/O; isolate side effects to `adapters/`.

```
src/<pkg>/
  domain/      # pure logic, no I/O, no framework imports
  adapters/    # db, http, filesystem — all I/O lives here
  services/    # orchestration between domain and adapters
tests/
  unit/ integration/ e2e/
```

## Code Standards

### Must

- Fully type-annotate every public function, method, and module-level value.
- Pass `mypy --strict` (or configured equivalent).
- Use PEP 604 (`str | None`) and PEP 585 (`list[int]`) syntax.
- Use `pathlib` for paths, f-strings for formatting, `logging` (never `print`) for diagnostics.
- Raise **specific** exceptions derived from a project base class. Use `raise ... from e`.
- Validate external input once, at the boundary, with pydantic. Trust it internally.
- Centralize config in a single `Settings` object (`pydantic-settings`). Secrets via env vars only.

### Must Not

- Use `pip install` in instructions, docs, or scripts.
- Write untyped public APIs.
- Catch bare `Exception` and swallow it.
- Use mutable default arguments.
- Perform network, I/O, or state mutation at module import time.
- Build deep inheritance trees. Prefer composition, Protocols, and callables.
- Hand-roll singletons or DI frameworks. Modules are singletons; constructor injection is the DI container.
- Commit secrets, API keys, or `.env` files with real values.

### Pythonic Patterns

- **Strategy** → pass a callable or `Protocol`-typed object.
- **Factory** → a plain function dispatching on an enum.
- **Adapter / Repository** → `Protocol` in `domain/`, implementation in `adapters/`.
- **Singleton** → module or `@functools.cache` on a factory.
- **Observer** → `asyncio.Queue` or a typed event bus.
- **RAII** → `with` / `contextlib`.

## Testing

- `pytest` only. No `unittest.TestCase` in new code.
- Name tests as behaviors: `test_returns_none_when_user_not_found`.
- Arrange–Act–Assert, separated by blank lines. One logical assertion per test.
- Use `pytest.mark.parametrize` instead of loops in tests.
- Mock at **your** seams (pass a fake `Protocol` implementation). Do not monkey-patch deep into third-party internals.
- Branch coverage ≥ 85% on `domain/` and `services/`.
- Reach for `hypothesis` for parsers, serializers, and invariants.

**Test pyramid:** many fast unit tests, fewer integration tests (use `respx`, `testcontainers`), a handful of e2e.

Do not test: third-party libraries, trivial getters, dataclass equality, private methods.

## Performance

**Measure before optimizing.** Use `cProfile`, `py-spy`, `scalene`, or `memray`.

| Workload          | Tool                                    |
|-------------------|-----------------------------------------|
| I/O-bound async   | `asyncio` + `httpx` / `asyncpg` / `aiofiles` |
| I/O-bound simple  | `ThreadPoolExecutor`                    |
| CPU-bound         | `ProcessPoolExecutor`, NumPy/Polars, or Rust via PyO3 |

- Use `asyncio.TaskGroup` (3.11+) over loose `create_task`.
- Never block inside async — use `asyncio.to_thread` for necessary blocking calls.
- `__slots__` on hot-path dataclasses. `functools.cache` for pure idempotent functions.

## Agent Workflow

Every task:

1. Read `pyproject.toml`, existing structure, and existing tests first.
2. Confirm or state assumptions; ask one focused clarifying question only if blocked.
3. Write the failing test. Run it. Confirm it fails for the right reason.
4. Write the minimum code to pass. Refactor. Keep green.
5. Run the full gate locally:
   ```bash
   uv run ruff check . && uv run ruff format --check .
   uv run mypy src
   uv run pytest --cov
   uv run pip-audit
   ```
6. Update docstrings and README if behavior or interfaces changed.
7. Summarize: what changed, tests added, coverage delta, remaining TODOs.

## Reject-On-Sight Output

Do not produce any of these:

- `pip install ...` instructions
- `requirements.txt` as dependency source of truth
- Untyped public functions
- `print()` used as logging
- `except Exception: pass`
- Tests covering only the happy path
- Business logic inside `__init__`
- Module-level side effects at import time
- Production code without a corresponding test

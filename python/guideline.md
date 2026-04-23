# Python Engineering Guidelines for Autonomous Agents

A directive reference for writing **idiomatic, production-grade, high-performance Python** using established design patterns and test-driven development. Agents MUST follow these rules unless the user explicitly overrides a specific item.

> **Conventions used below**
> - **MUST / MUST NOT** — non-negotiable
> - **SHOULD / SHOULD NOT** — default behavior; deviate only with justification
> - **MAY** — permitted when the situation warrants

---

## 1. Tooling & Project Management (uv is the default)

### 1.1 Package & Environment Management

- **MUST** use `uv` (Astral) as the package and project manager. No `pip`, `pip-tools`, `poetry`, `pipenv`, `conda`, or `virtualenv` invocations unless the user specifies otherwise.
- **MUST** pin the Python version in `.python-version` and in `pyproject.toml` under `requires-python`.
- **MUST** commit `uv.lock` to source control for applications. For libraries, commit it for CI reproducibility but do not ship it in the wheel.
- **MUST NOT** edit `uv.lock` by hand.

**Canonical commands:**

```bash
# Initialize
uv init --package my_project          # for libraries
uv init --app my_project              # for applications

# Add / remove dependencies
uv add httpx pydantic
uv add --dev pytest ruff mypy
uv add --optional docs mkdocs-material
uv remove requests

# Lock & sync
uv lock                                # refresh lockfile
uv sync --frozen                       # install exactly from lockfile (CI)
uv sync --all-extras --dev             # dev install

# Run
uv run python -m my_project
uv run pytest
uv run ruff check .

# Tools (global CLIs, isolated envs)
uv tool install pre-commit
uv tool run ruff format .              # shorthand: uvx ruff format .
```

### 1.2 `pyproject.toml` (the single source of truth)

**MUST** use PEP 621 metadata. **MUST NOT** maintain `setup.py`, `setup.cfg`, or `requirements*.txt` alongside it.

```toml
[project]
name = "my_project"
version = "0.1.0"
description = "..."
readme = "README.md"
requires-python = ">=3.12"
license = { text = "MIT" }
authors = [{ name = "Team", email = "team@example.com" }]
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.7",
]

[project.optional-dependencies]
docs = ["mkdocs-material>=9"]

[dependency-groups]                    # PEP 735 — uv-native dev groups
dev = [
    "pytest>=8",
    "pytest-cov>=5",
    "pytest-asyncio>=0.23",
    "hypothesis>=6",
    "ruff>=0.6",
    "mypy>=1.11",
    "pre-commit>=3.7",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "W", "I", "N", "B", "UP", "SIM", "RUF", "PL", "PT", "ANN"]
ignore = ["ANN101", "ANN102"]

[tool.mypy]
strict = true
python_version = "3.12"
warn_unused_ignores = true
warn_return_any = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config --cov=src --cov-report=term-missing"
asyncio_mode = "auto"
```

### 1.3 Required Dev Toolchain

| Concern            | Tool                    | Notes |
|--------------------|-------------------------|-------|
| Linting + imports  | **ruff**                | Replaces flake8, isort, pyupgrade, pydocstyle |
| Formatting         | **ruff format**         | Replaces black |
| Type checking      | **mypy** (strict) or **pyright** | Pick one per repo |
| Testing            | **pytest**              | Plus `pytest-cov`, `pytest-asyncio`, `hypothesis` |
| Pre-commit hooks   | **pre-commit**          | Run ruff, mypy, end-of-file-fixer |
| Security           | **pip-audit** via `uv`  | `uv run pip-audit` in CI |

---

## 2. Project Structure

**MUST** use the `src/` layout for anything shippable. It prevents accidental imports from the working directory and mirrors installed behavior.

```
my_project/
├── .python-version
├── pyproject.toml
├── uv.lock
├── README.md
├── src/
│   └── my_project/
│       ├── __init__.py
│       ├── py.typed              # mark as typed (PEP 561)
│       ├── __main__.py           # entry for `python -m my_project`
│       ├── config.py
│       ├── domain/               # pure business logic, no I/O
│       ├── adapters/             # I/O boundaries (db, http, fs)
│       ├── services/             # orchestration
│       └── cli.py
├── tests/
│   ├── conftest.py
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── .github/workflows/ci.yml
```

- **MUST** keep `domain/` free of I/O and framework imports. It is pure functions and data types.
- **MUST** isolate side effects to `adapters/`.
- **SHOULD** follow a hexagonal / ports-and-adapters arrangement for anything non-trivial.

---

## 3. Idiomatic Python

### 3.1 Language Features — Use These

- **f-strings** for formatting. Never `%` or `.format()`.
- **Pathlib** for filesystem paths. Never `os.path` string concatenation.
- **`match` / `case`** for structural pattern matching (Python ≥ 3.10) when branching on shape.
- **Walrus (`:=`)** only when it removes a duplicated expression — do not shoehorn it.
- **PEP 604 union syntax**: `str | None`, not `Optional[str]` or `Union[str, None]`.
- **PEP 585 builtins as generics**: `list[int]`, not `List[int]`.
- **Comprehensions and generator expressions** in place of manual `for`+`append` loops.
- **`enumerate`, `zip`, `itertools`** rather than manual indexing.
- **`collections.abc`** for type annotations of containers you accept (`Iterable`, `Mapping`, `Sequence`).

### 3.2 Antipatterns — Avoid These

- Mutable default arguments (`def f(x=[])`). Use `None` + sentinel.
- `except:` or `except Exception:` without rethrow, logging, or narrow justification.
- `from module import *` outside of `__init__.py` re-exports.
- `print()` for diagnostics in library or service code — use `logging` / `structlog`.
- Manual `__init__` boilerplate when `@dataclass` or `pydantic.BaseModel` fits.
- `os.environ["X"]` scattered through the codebase — centralize in a `Settings` object.
- Global mutable state. Module-level constants are fine; module-level dicts you `.update()` are not.

### 3.3 Data Classes — Choose Deliberately

| Use case                                         | Choose                       |
|--------------------------------------------------|------------------------------|
| Internal, trusted data, no validation needed     | `@dataclass(frozen=True, slots=True)` |
| External input (API, config, user data)          | `pydantic.BaseModel`         |
| Tagged/sum types                                 | `enum.Enum` or `Literal[...]` |
| Sentinel values                                  | `typing.Final` + module const |

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class Money:
    amount: int            # store as minor units; never float for currency
    currency: str
```

### 3.4 Typing — Strict by Default

- **MUST** fully annotate all public functions, methods, and module-level values.
- **MUST** pass `mypy --strict` (or equivalent pyright config) in CI.
- **MUST** prefer `Protocol` over ABCs for structural typing — it keeps adapters decoupled.
- **SHOULD** use `typing.Final`, `Literal`, `TypedDict`, `NewType`, `assert_never` where they clarify intent.
- **MUST NOT** use `Any` except at true system boundaries, and document why.

```python
from typing import Protocol

class UserRepository(Protocol):
    def get(self, user_id: int) -> User | None: ...
    def save(self, user: User) -> None: ...
```

---

## 4. Design Patterns (Pythonic Implementations)

Python's first-class functions, duck typing, and context managers make many Gang-of-Four patterns lighter-weight than in Java/C++. Reach for the Pythonic form first.

### 4.1 Patterns You Will Use Often

- **Strategy** → pass a callable or a `Protocol`-typed object. Don't build a class hierarchy for a single method.
- **Factory** → a plain function returning the right concrete type, usually dispatching on an enum.
- **Adapter** → a thin wrapper class whose interface is a `Protocol` defined in `domain/`.
- **Repository** → hides persistence behind a `Protocol`; the in-memory implementation is your test double.
- **Dependency Injection** → constructor injection of protocol-typed collaborators. No DI frameworks needed; Python is already a DI container.
- **Observer / Pub-Sub** → `asyncio.Queue`, `blinker`, or a typed event bus.
- **Builder** → only when object construction genuinely has many optional steps. Otherwise, keyword arguments + dataclasses win.
- **Context Manager** (`with` / `contextlib`) → the Pythonic RAII; use it for any acquire/release pair.
- **Singleton** → use a **module** (imports are cached) or `functools.lru_cache` on a factory. Never hand-roll `__new__` tricks.
- **Iterator / Generator** → prefer generators (`yield`) to custom iterator classes.
- **Decorator** → for cross-cutting concerns: caching, retry, timing, auth. Preserve signatures with `functools.wraps`.

### 4.2 Architectural Patterns

- **Hexagonal / Ports & Adapters** — default for services.
- **CQRS** — when read and write models diverge meaningfully.
- **Unit of Work** — for transactional consistency across repositories.
- **Result / Either types** for expected failure branches in hot paths where exceptions are costly; otherwise exceptions are fine.

### 4.3 Composition Over Inheritance

- **SHOULD** prefer composition. Inherit only for genuine is-a relationships or to share framework contracts (e.g., `pydantic.BaseModel`).
- **MUST NOT** build deep inheritance trees. Two levels is a smell; three is a mistake.

---

## 5. Test-Driven Development

### 5.1 The Loop

**MUST** follow Red → Green → Refactor for all new behavior:

1. **Red** — write the smallest failing test that expresses the next bit of behavior.
2. **Green** — write the minimum production code to pass. Duplication is acceptable here.
3. **Refactor** — eliminate duplication, clarify names, improve structure. Tests must remain green.

- **MUST NOT** write production code without a failing test first, except for: exploratory spikes (which get thrown away), trivial boilerplate (imports, `__init__.py`), or pure refactors.
- **MUST** commit in the green state.

### 5.2 Test Structure

- **MUST** use `pytest`. No `unittest.TestCase` classes in new code.
- **MUST** follow **Arrange–Act–Assert**, separated by blank lines.
- **MUST** name tests as behavioral assertions: `test_returns_none_when_user_not_found`, not `test_get_user_1`.
- **MUST** keep each test focused on one behavior. One logical assertion per test.
- **SHOULD** organize tests to mirror the `src/` tree.

```python
def test_transfer_debits_source_and_credits_destination() -> None:
    # Arrange
    bank = Bank(accounts={"a": Account(balance=100), "b": Account(balance=0)})

    # Act
    bank.transfer(src="a", dst="b", amount=30)

    # Assert
    assert bank.accounts["a"].balance == 70
    assert bank.accounts["b"].balance == 30
```

### 5.3 Fixtures, Parametrization, and Mocks

- **SHOULD** use fixtures for setup reused across tests; scope them (`function`, `module`, `session`) deliberately.
- **MUST** use `pytest.mark.parametrize` instead of for-loops inside tests.
- **MUST** mock at the boundary you own — pass a fake `Protocol` implementation rather than monkey-patching deep internals.
- **MUST NOT** use `unittest.mock.patch` to reach into third-party internals as a primary strategy. If you feel the need, your seams are wrong.

### 5.4 Test Pyramid

- Many fast **unit** tests (pure domain, no I/O). Target < 10 ms each.
- Fewer **integration** tests (real DB, real HTTP via `respx`/`httpx.MockTransport`, containers via `testcontainers`).
- A small number of **end-to-end** tests driving the real entry point.

### 5.5 Coverage & Property-Based Testing

- **MUST** enforce branch coverage ≥ 85% on `domain/` and `services/`. Coverage on adapters is less meaningful.
- **SHOULD** use `hypothesis` for invariants, parsers, serializers, and anything with a non-trivial input space.

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs: list[int]) -> None:
    assert sorted(sorted(xs)) == sorted(xs)
```

### 5.6 What Not to Test

- Third-party libraries.
- Trivial getters/setters and auto-generated dataclass equality.
- Implementation details (private methods, internal state). Test through public interfaces so refactors don't break the suite.

---

## 6. Performance

> **First rule: measure.** Do not optimize without a profile showing where the time goes. `cProfile`, `py-spy`, `scalene`, and `memray` are the defaults.

### 6.1 Algorithmic Choices

- **MUST** pick the right data structure: `set`/`dict` for membership and lookup, `collections.deque` for FIFO, `heapq` for priority queues, `bisect` for sorted inserts.
- **MUST** avoid O(n²) loops over lists for membership checks.
- **SHOULD** use generators to stream large data instead of materializing lists.

### 6.2 Concurrency Model — Pick the Right One

| Workload              | Tool                                          |
|-----------------------|-----------------------------------------------|
| I/O-bound, many tasks | `asyncio` + `httpx`/`aiofiles`/`asyncpg`      |
| I/O-bound, simple     | `concurrent.futures.ThreadPoolExecutor`       |
| CPU-bound             | `concurrent.futures.ProcessPoolExecutor`, `multiprocessing`, or offload to NumPy/Polars/Rust |
| CPU-bound (3.13+)     | Free-threaded build (`--disable-gil`) where supported |

- **MUST NOT** mix blocking calls inside async code. Use `asyncio.to_thread` for necessary blocking calls.
- **SHOULD** prefer `anyio` when writing library code that needs to support both `asyncio` and `trio`.
- **MUST** use `async with` / `async for` and structured concurrency (`asyncio.TaskGroup` in 3.11+) over fire-and-forget `asyncio.create_task`.

### 6.3 High-Leverage Techniques

- **`functools.lru_cache` / `functools.cache`** for pure, idempotent functions with bounded input domains.
- **`__slots__`** on dataclasses used in hot paths to cut memory and attribute access cost.
- **NumPy / Polars / PyArrow** for numeric and columnar data. A vectorized one-liner usually beats a hand-rolled C extension.
- **Cython, Rust (PyO3), or C extensions** only when profiling proves the bottleneck and vectorization doesn't apply.
- **Lazy imports** inside functions for heavy optional dependencies to keep CLI start-up fast.

### 6.4 Memory

- Stream large files; don't `.read()` them whole.
- Use `__slots__` on small, numerous objects.
- Beware closures capturing large locals.
- Profile with `tracemalloc` or `memray` before speculating.

---

## 7. Error Handling

- **MUST** raise **specific** exceptions. Derive from a project-level base (`class MyProjectError(Exception): ...`) so callers can catch the whole surface cleanly.
- **MUST** let exceptions propagate when the current layer cannot add meaningful context or recovery.
- **SHOULD** favor **EAFP** (try/except) over **LBYL** (pre-check) for concurrent or racy resources.
- **MUST** use `raise ... from e` to preserve cause chains. Never silently swallow.
- **MUST NOT** use exceptions for normal control flow in hot paths.
- **SHOULD** validate external input once, at the boundary, with pydantic or explicit parsers. Trust it internally.

```python
class OrderError(Exception): ...
class InsufficientStockError(OrderError): ...

try:
    reserve(item_id, qty)
except StockRepositoryError as e:
    raise InsufficientStockError(f"cannot reserve {qty} of {item_id}") from e
```

---

## 8. Logging, Configuration, Observability

### 8.1 Logging

- **MUST** use the stdlib `logging` module (or `structlog` wrapping it) with a named logger per module: `logger = logging.getLogger(__name__)`.
- **MUST** log structured key-value pairs, not interpolated strings, for machine-parseable output.
- **MUST NOT** log secrets, credentials, full request bodies containing PII, or raw tokens.
- **SHOULD** use levels correctly: `DEBUG` for developer detail, `INFO` for lifecycle events, `WARNING` for recoverable anomalies, `ERROR` for failed operations, `CRITICAL` for process-level failures.

### 8.2 Configuration

- **MUST** load configuration through a single typed `Settings` object, typically `pydantic-settings.BaseSettings`.
- **MUST** source values from environment variables (12-factor). Support `.env` for local dev only.
- **MUST NOT** hardcode secrets. Ever.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_")
    database_url: str
    log_level: str = "INFO"
```

### 8.3 Observability

- **SHOULD** instrument services with **OpenTelemetry** for traces, metrics, and logs.
- **SHOULD** emit a health endpoint and a readiness endpoint for any service.
- **SHOULD** attach request/correlation IDs to every log line within a request scope.

---

## 9. Security

- **MUST** run `pip-audit` (or `uv run pip-audit`) in CI and fail on known vulnerabilities in dependencies.
- **MUST** use parameterized queries for all SQL. Never string-concatenate user input.
- **MUST** validate and constrain all external input at the boundary.
- **MUST** use `secrets` module for tokens and nonces — never `random`.
- **MUST NOT** pickle untrusted data.
- **MUST NOT** use `subprocess.run(..., shell=True)` with untrusted input.
- **SHOULD** enable Dependabot/Renovate for automated dependency bumps.

---

## 10. Documentation

- **MUST** write docstrings for all public modules, classes, and functions. Use Google or NumPy style — be consistent per project.
- **MUST** include a `README.md` with: what it does, how to install with `uv`, how to run tests, how to run the app.
- **SHOULD** keep docstrings behavioral ("returns the balance after applying the transaction") not mechanical ("loops over entries").
- **SHOULD** use `mkdocs-material` with `mkdocstrings` for external-facing libraries.
- **MUST NOT** restate the type signature in the docstring — types belong in annotations.

---

## 11. CI/CD Baseline

Every repo **MUST** include a CI pipeline that, on every push and PR:

1. Sets up the pinned Python via `uv python install`.
2. Runs `uv sync --frozen --all-extras --dev`.
3. Runs `uv run ruff check .` and `uv run ruff format --check .`.
4. Runs `uv run mypy src`.
5. Runs `uv run pytest` with coverage, failing below the agreed threshold.
6. Runs `uv run pip-audit`.
7. Builds the package with `uv build` if it is a library.

Matrix-test across the lowest and highest supported Python versions.

---

## 12. Agent Workflow Checklist

When an agent starts a Python task, walk this list **in order**:

1. **Understand the requirement.** If ambiguous, ask one focused clarifying question; otherwise proceed and state assumptions.
2. **Inspect the project.** Read `pyproject.toml`, existing structure, existing tests, and `README.md` before touching code. Match the established conventions even when they differ from these defaults — consistency wins.
3. **If starting fresh**, run `uv init` with the correct flavor (app vs. package), add dev dependencies, and commit a minimal passing skeleton.
4. **Write the failing test first.** Name it as a behavior. Run it and confirm it fails for the expected reason.
5. **Implement minimally.** Make the test pass. No speculative generality.
6. **Refactor.** Extract, rename, remove duplication. Tests must stay green.
7. **Extend.** Repeat the loop for the next behavior. Commit at each green step.
8. **Finalize.** Run the full toolchain locally: `ruff check`, `ruff format`, `mypy`, `pytest --cov`, `pip-audit`. All must pass.
9. **Document.** Update docstrings, README, and any changelog entry.
10. **Summarize.** Report what changed, which tests were added, measured coverage, and any remaining TODOs or assumptions.

---

## 13. Anti-Checklist — Reject These Outputs

An agent's output **MUST NOT** include any of the following:

- `pip install` commands in instructions (use `uv add`).
- `requirements.txt` as the dependency source of truth.
- Untyped public functions.
- `print` statements used as logging in service/library code.
- Tests that exercise only the happy path.
- Exception handlers that catch `Exception` and `pass`.
- Business logic inside `__init__` methods.
- Module-level side effects (network calls, file writes) at import time.
- Hand-rolled singletons, hand-rolled DI containers, or deep inheritance hierarchies.
- Code committed without a corresponding test (except trivial boilerplate or pure refactors).

---

## 14. Quick Reference Card

```bash
# Start a project
uv init --app my_service && cd my_service
uv add fastapi pydantic-settings httpx
uv add --dev pytest pytest-cov pytest-asyncio hypothesis ruff mypy pre-commit

# Everyday loop
uv run pytest -x --ff          # fail fast, failed-first
uv run ruff check --fix .
uv run ruff format .
uv run mypy src

# Before pushing
uv lock --check                 # lockfile up to date?
uv run pytest --cov
uv run pip-audit
```

---

**Governing principle:** write code that a teammate can read, test, and change six months from now without fear. Idiomatic Python, tested before it is written, measured before it is optimized, and shipped behind clear interfaces — in that order.

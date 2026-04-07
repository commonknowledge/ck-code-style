# Python Code Style Guidelines

## Tooling

| Tool | Purpose |
|------|---------|
| `uv` | Dependency, environment, and module management |
| `pytest` | Testing — all new features must have unit tests |
| `ruff` | Linting and formatting |
| `mypy` | Type checking (strict mode) |
| `bandit` | Security analysis |

---

## Project Setup
```bash
# Create project and module directories
mkdir my-project
cd my-project
mkdir src/my_project  # Replace `my_project` with actual package name

# Initialise uv project
uv init

# Add dev dependencies — always run this, even for simple prototypes; no exceptions
uv add --dev pytest ruff mypy bandit hatchling

# Create a virtual environment and sync
uv sync
```

Also add the hatchling build system to `pyproject.toml`:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

The resulting structure:
```
my-project/
├── src/
│   └── my_project/       # Replace with actual package name
│       └── __init__.py
├── tests/
│   └── __init__.py
├── pyproject.toml
└── README.md
```

> **If a project already exists, respect its structure.** Do not recreate directories, rename modules, or reorganise files unless explicitly asked. Adapt to the existing layout.

---

## Running the App

Define a named entry point in `pyproject.toml`:

```toml
[project.scripts]
start = "my_project.app:main"
```

The target must be a callable with no required arguments:

```python
def main() -> None:
    """Start the application."""
    init_db()
    app.run()
```

Then run with:

```bash
uv run start
```

Use `start` as the conventional script name for the primary entry point. Add additional named scripts (e.g. `worker`, `migrate`) for other runnable processes.

---

## Configuration

Add the following to `pyproject.toml`:
```toml
[tool.ruff]
line-length = 101
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "S", "B", "A", "C4", "PT", "RUF", "SIM"]

[tool.ruff.format]
quote-style = "double"

[tool.mypy]
strict = true
python_version = "3.12"

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.bandit]
targets = ["src"]
skips = ["B101"]  # B101: allow assert in test files
```

> **Keep version values current.** Update `target-version` and `python_version` to match the Python version in use (e.g. `py314` / `3.14`). Do not leave them pinned to an older version if the project runs on something newer.

---

## Dependencies

Use `uv add` to manage dependencies:
```bash
# Add a runtime dependency
uv add requests

# Add a dev dependency
uv add --dev pytest-cov

# Remove a dependency
uv remove requests

# Sync environment to lockfile
uv sync
```

Never edit `pyproject.toml` dependency lists by hand — always go through `uv`.

---

## Code Conventions

### Naming

- **Modules and packages:** `snake_case` (e.g. `user_auth.py`)
- **Classes:** `PascalCase` (e.g. `TokenValidator`)
- **Functions and variables:** `snake_case` (e.g. `fetch_user_profile`)
- **Constants:** `UPPER_SNAKE_CASE` (e.g. `MAX_RETRY_COUNT`)
- **Private/internal:** prefix with a single underscore (e.g. `_parse_token`)

### Docstrings

Use **Google style** docstrings on all public modules, classes, and functions:
```python
def fetch_user(user_id: int, *, include_inactive: bool = False) -> User:
    """Fetch a user by their ID.

    Args:
        user_id: The unique identifier of the user.
        include_inactive: If True, return users even if deactivated.

    Returns:
        The matching User object.

    Raises:
        UserNotFoundError: If no user matches the given ID.
    """
```

### Error Handling

- **Define custom exceptions** for domain-specific errors. Place them in a dedicated `exceptions.py` module or alongside the relevant domain module.
- **Never use bare `except:`** — always catch a specific exception type.
- **Never silently swallow exceptions.** At minimum, log the error.
- Prefer raising exceptions over returning error codes or `None` sentinels for failure cases.
```python
# Good
class UserNotFoundError(Exception):
    """Raised when a requested user does not exist."""

try:
    user = repository.get(user_id)
except DatabaseConnectionError:
    logger.exception("Failed to connect while fetching user %s", user_id)
    raise

# Bad
try:
    user = repository.get(user_id)
except:
    pass
```

### Logging

- Use the `logging` module — **never `print()` for operational output.**
- Get a module-level logger: `logger = logging.getLogger(__name__)`
- Use appropriate levels: `debug` for tracing, `info` for normal operations, `warning` for recoverable issues, `error`/`exception` for failures.
- Use lazy formatting (`logger.info("Fetched user %s", user_id)`) — not f-strings inside log calls.

---

## Type Safety

All code must pass `mypy` in strict mode. This means:

- All function arguments and return types must be annotated.
- No use of `Any` without an explicit `# type: ignore` comment explaining why.
- If `mypy` fails, this may be because third-party packages do not bundle their own type annotations. Run this check to see if bundled type annotations exist for a package:
```bash
uv run python -c "import importlib.metadata; files = importlib.metadata.files('<package>') or []; print(any('py.typed' in str(f) for f in files))"
```
If this does not print `True`, try installing the stub with `uv add --dev types-<package>`.

```bash
uv run mypy src
```

---

## Linting and Formatting

Run `ruff` for both formatting and linting:
```bash
# Format
uv run ruff format .

# Lint (with auto-fix)
uv run ruff check --fix .
```

Ruff replaces `black`, `isort`, `flake8`, and several other tools. Do not add these separately.

---

## Testing

All new features must include unit tests in the `tests/` directory, mirroring the module structure — always, even for simple prototypes; no exceptions.
```
src/my_project/auth.py  →  tests/test_auth.py
```

Run tests:
```bash
uv run pytest

# With coverage
uv run pytest --cov=src --cov-report=term-missing
```

---

## Security

Run `bandit` against the main module before committing:
```bash
uv run bandit -r src
```

---

## CI Checklist

Before merging, all of the following must pass:
```bash
uv run ruff format --check .
uv run ruff check .
uv run mypy src
uv run bandit -r src
uv run pytest
```
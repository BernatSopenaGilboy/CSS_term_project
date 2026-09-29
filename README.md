# CSS Term Project

## Requirements

- Python 3.12+ (see `.python-version`)
- [`uv`](https://docs.astral.sh/uv/) as the package/environment manager for this project

## Getting Started with `uv`

This project uses `uv` instead of `pip`/`venv`/`poetry`. `uv` manages the virtual environment, the Python version, and dependencies together, driven by `pyproject.toml` and a `uv.lock` lockfile — everyone gets the exact same environment.

### 1. Install `uv`

If you don't have it yet:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

(See the [official docs](https://docs.astral.sh/uv/getting-started/installation/) for other install methods, e.g. `pipx install uv`, `brew install uv`.)

### 2. Set up the project

From the repo root:

```bash
uv sync
```

This reads `pyproject.toml` (and `uv.lock` if present), installs the correct Python version if needed, creates a `.venv/` in the project, and installs all dependencies into it. Run this after cloning the repo and after pulling changes that touch `pyproject.toml`/`uv.lock`.

### 3. Run the project

Use `uv run` instead of activating the venv manually — it makes sure you're always running inside the project's environment with the right dependencies:

```bash
uv run main.py
```

You can run any script, module, or tool the same way, e.g. `uv run pytest`, `uv run python -m src.some_module`.

### 4. Managing dependencies

Don't `pip install` by hand — always go through `uv` so `pyproject.toml`/`uv.lock` stay in sync for everyone.

```bash
# Add a runtime dependency
uv add numpy pandas

# Add a dev-only dependency (test/lint tools, not needed at runtime)
uv add --dev pytest

# Remove a dependency
uv remove some-package

# Install/refresh the environment to match the lockfile exactly
uv sync
```

`uv add`/`uv remove` update both `pyproject.toml` and `uv.lock` automatically — commit both files.

### 5. The lockfile

`uv.lock` pins exact versions of every dependency (and transitive dependency) so the whole team runs identical code. **Always commit `uv.lock`** when it changes. Never hand-edit it — let `uv add`/`uv remove`/`uv sync` regenerate it.

### 6. Python version

`.python-version` pins this project to Python 3.12. `uv` will download and manage that interpreter automatically on `uv sync`/`uv run` — no need to install Python separately or manage it with `pyenv`.

### Quick reference

| Task | Command |
|---|---|
| Install/update environment from lockfile | `uv sync` |
| Run a script/tool in the project env | `uv run <command>` |
| Add a dependency | `uv add <package>` |
| Add a dev dependency | `uv add --dev <package>` |
| Remove a dependency | `uv remove <package>` |
| Run the test suite | `uv run pytest` |

## Code Standards

Before contributing, read [`MANIFESTO.md`](./MANIFESTO.md) — it defines what "good code" means for this project (clean code, SOLID, testing, etc.) and is enforced during review.

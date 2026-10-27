# Ruff

*Ultra-fast Python linter and formatter written in Rust.*

[Ruff](https://github.com/astral-sh/ruff) is an extremely fast Python linter and formatter written in Rust. It effectively replaces tools like Flake8, Black, isort, pydocstyle, pyupgrade, autoflake and many others, while offering performance improvements of 10x to 100x.

## Why Adopt Ruff?

- **Blazing speed**: 10–100x faster than traditional Python linters due to its Rust architecture
- **All-in-one solution**: Combines linting, formatting and import sorting in a single tool
- **Modern Python**: Full support for Python 3.8+ with advanced type-hint handling
- **Optimized for FastAPI**: Built-in rules for async/await patterns and Pydantic models
- **Seamless integration**: Works with editors, CI/CD pipelines and pre-commit hooks
- **Automatic fixes**: Auto-fixes many style violations

## Installation

```sh
# Add Ruff as a development dependency
uv add --dev ruff

# Or install globally with uv
uv tool install ruff

# Alternative installation with pip
pip install ruff
```

## Basic Usage

```sh
# Lint your code
uv run ruff check .

# Automatically fix fixable issues
uv run ruff check --fix .

# Format your code according to standards
uv run ruff format .

# Combined linting and formatting (recommended workflow)
uv run ruff check --fix . && uv run ruff format .

# Watch mode for development
uv run ruff check --watch .
```

## Configuration

Create a `ruff.toml` file at your project root or configure directly in `pyproject.toml`:

```toml
# pyproject.toml
[tool.ruff]
target-version = "py38"
line-length = 88

exclude = [
    ".git",
    ".venv",
    ".mypy_cache",
    ".pytest_cache",
    "__pycache__",
    "build",
    "dist",
]

show-fixes = true

[tool.ruff.lint]
select = [
    "E",     # pycodestyle errors
    "W",     # pycodestyle warnings
    "F",     # Pyflakes (logical errors)
    "I",     # isort (import sorting)
    "B",     # flake8-bugbear (potential bugs)
    "C4",    # flake8-comprehensions
    "UP",    # pyupgrade (modernize code)
    "ARG",   # flake8-unused-arguments
    "SIM",   # flake8-simplify
    "RUF",   # Ruff-specific rules
]

ignore = [
    "E501",    # Line too long (handled by formatter)
    "B008",    # Function calls in default values (useful for FastAPI)
]

fixable = ["ALL"]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = [
    "S101",    # Use of assert detected
    "ARG",     # Unused function arguments (fixtures)
]
"__init__.py" = [
    "F401",    # Unused import (re-export)
]

[tool.ruff.lint.isort]
split-on-trailing-comma = true
known-first-party = ["app", "core", "api"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
docstring-code-format = true
```

### FastAPI-Specific Rules

```toml
[tool.ruff.lint]
select = [
    "ASYNC",  # Async/await best practices
    "FAST",   # FastAPI-specific rules
    "TRIO",   # Async code best practices
    "RUF006", # Detect mutations of default values
]

[tool.ruff.lint.per-file-ignores]
"app/api/**/*.py" = [
    "B008",  # Allow function calls in defaults (Depends(), Body(), etc.)
]
```

## VS Code Integration

Install the [Ruff extension](https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff) and add to your `settings.json`:

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "ruff.lint.args": ["--config=pyproject.toml"],
  "ruff.format.args": ["--config=pyproject.toml"],
  "ruff.enable": true,
  "ruff.organizeImports": true
}
```

## PyCharm / IntelliJ Integration

1. Install the Ruff plugin from the marketplace
2. Configure in **Settings > Tools > External Tools**
3. Add Ruff as an external formatter/linter

## Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.0
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format
```

```sh
pip install pre-commit
pre-commit install
pre-commit run --all-files
```

## CI/CD — GitHub Actions

```yaml
# .github/workflows/quality.yml
name: Code Quality
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - name: Install dependencies
        run: uv sync --dev --all-extras

      - name: Run Ruff lint
        run: uv run ruff check . --output-format=github

      - name: Check formatting
        run: uv run ruff format --check .
```

## CI/CD — GitLab CI

```yaml
# .gitlab-ci.yml
ruff-lint:
  stage: test
  image: python:3.11
  before_script:
    - pip install uv
    - uv sync --dev
  script:
    - uv run ruff check .
    - uv run ruff format --check .
  rules:
    - if: $CI_MERGE_REQUEST_ID
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

## CI/CD — Jenkins Pipeline

```groovy
pipeline {
    agent any
    stages {
        stage('Quality Check') {
            steps {
                sh 'uv sync --dev'
                sh 'uv run ruff check . || true'
                sh 'uv run ruff format --check .'
            }
        }
    }
}
```

## Common Workflows

```sh
# Full quality check
uv run ruff check --fix .
uv run ruff format .

# Dry-run (no modifications)
uv run ruff check .
uv run ruff format --check .

# Run on specific files
uv run ruff check app/main.py app/models/

# Get rule explanation
uv run ruff rule F401
uv run ruff rule --all  # List all rules

# Show violation statistics
uv run ruff check --statistics .

# Watch changes (only modified files)
uv run ruff check $(git diff --name-only --diff-filter=ACM | grep '\.py$')

# Show active configuration
uv run ruff check --show-settings

# Export configuration as JSON
uv run ruff check --show-settings --output-format=json > config.json
```

## Performance Optimizations

For large projects, consider these optimizations:

```toml
[tool.ruff]
# Enable cache (enabled by default)
cache-dir = ".ruff_cache"

# Respect .gitignore to limit search scope
respect-gitignore = true
```

```sh
# Parallel execution using multiple cores
RAYON_NUM_THREADS=8 uv run ruff check .

# Incremental analysis (only changed files)
uv run ruff check --diff $(git diff --name-only HEAD)
```

## Migration from Other Tools

Ruff eases migration from traditional tools:

```sh
# Compare formatting with Black
ruff format --diff .

# Check isort compatibility
ruff check --select=I --diff .
```

## Resources

- [Official Documentation](https://docs.astral.sh/ruff/)
- [Full List of Rules](https://docs.astral.sh/ruff/rules/)
- [FAQ — comparison with Flake8, Black and isort](https://docs.astral.sh/ruff/faq/)
- [Editor Integrations](https://docs.astral.sh/ruff/integrations/)
- [Performance Comparison](https://github.com/astral-sh/ruff#benchmarks)

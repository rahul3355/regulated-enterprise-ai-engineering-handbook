# Python Packaging and Dependency Management

## Why Poetry

Poetry is the modern Python dependency manager. It provides deterministic builds, lock files, virtual environment management, and publishing. For banking services, Poetry ensures reproducible builds across development, staging, and production.

## Project Configuration

```toml
# pyproject.toml
[tool.poetry]
name = "banking-genai-service"
version = "1.0.0"
description = "GenAI-powered banking document analysis service"
authors = ["Engineering Team <engineering@bank.com>"]
license = "Proprietary"
readme = "README.md"
packages = [{ include = "banking_api", from = "src" }]

[tool.poetry.dependencies]
python = "^3.11"

# Web framework
fastapi = "^0.110.0"
uvicorn = { extras = ["standard"], version = "^0.27.0" }

# Data validation
pydantic = "^2.6.0"
pydantic-settings = "^2.1.0"

# Database
sqlalchemy = { extras = ["asyncio"], version = "^2.0.0" }
asyncpg = "^0.29.0"
alembic = "^1.13.0"

# Cache
redis = { extras = ["hiredis"], version = "^5.0.0" }

# AI/ML
openai = "^1.12.0"
langchain = "^0.1.0"

# Task queue
celery = "^5.3.0"

# Utilities
python-dateutil = "^2.8.0"
orjson = "^3.9.0"
pydantic = "^2.6.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0.0"
pytest-asyncio = "^0.23.0"
pytest-cov = "^4.1.0"
pytest-mock = "^3.12.0"
hypothesis = "^6.97.0"
httpx = "^0.27.0"
factory-boy = "^3.3.0"

# Linting and formatting
ruff = "^0.2.0"
mypy = "^1.8.0"
pre-commit = "^3.6.0"

# Type stubs
types-redis = "^4.6.0"
types-python-dateutil = "^2.8.0"

[tool.poetry.scripts]
start = "banking_api.main:main"
migrate = "scripts.migrate:main"
seed = "scripts.seed_data:main"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

# Tool configurations
[tool.ruff]
target-version = "py311"
line-length = 100
select = ["E", "F", "I", "N", "W", "B", "C4", "UP"]

[tool.ruff.isort]
known-first-party = ["banking_api"]

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true

[[tool.mypy.overrides]]
module = ["celery.*", "langchain.*"]
ignore_missing_imports = true

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-v --tb=short"
```

## Dependency Management

```bash
# Install dependencies
poetry install                    # Install all dependencies
poetry install --only main        # Production only (no dev)
poetry install --with dev         # Include dev dependencies

# Add new dependency
poetry add openai                 # Add to main dependencies
poetry add --group dev pytest     # Add to dev dependencies

# Update dependencies
poetry update                     # Update all within version constraints
poetry update openai              # Update specific package

# Lock dependencies (reproducible builds)
poetry lock                       # Generate poetry.lock
poetry lock --no-update           # Re-lock without updating versions

# Export for Docker/CI
poetry export -f requirements.txt --output requirements.txt
poetry export -f requirements.txt --only main --without-hashes > requirements.txt
```

## Virtual Environment

```bash
# Poetry manages virtual environments automatically
poetry env info                   # Show current environment
poetry env use python3.11         # Use specific Python version
poetry env list                   # List all environments

# Run commands in virtual environment
poetry run pytest                 # Run pytest in venv
poetry run python -m banking_api  # Run module
poetry shell                      # Activate shell in venv
```

## Docker Deployment

```dockerfile
# Dockerfile - Multi-stage build
FROM python:3.11-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    POETRY_VERSION=1.7.1 \
    POETRY_HOME="/opt/poetry" \
    POETRY_VIRTUALENVS_CREATE=false

# Install Poetry
RUN pip install "poetry==$POETRY_VERSION"

WORKDIR /app

# Stage 1: Install dependencies
FROM base AS dependencies
COPY pyproject.toml poetry.lock ./
RUN poetry install --no-interaction --no-ansi --only main

# Stage 2: Production image
FROM base AS production
COPY --from=dependencies /app/.venv /app/.venv
COPY src/ ./src/
COPY alembic/ ./alembic/
COPY alembic.ini ./

# Create non-root user
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD ["uvicorn", "banking_api.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

## CI/CD Integration

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Poetry
        run: pip install poetry==1.7.1

      - name: Install dependencies
        run: poetry install --with dev

      - name: Run linter
        run: poetry run ruff check src/

      - name: Run type checker
        run: poetry run mypy src/

      - name: Run tests
        run: poetry run pytest --cov=banking_api --cov-fail-under=90

      - name: Build package
        run: poetry build
```

## Publishing (Internal Package Registry)

```bash
# Configure internal package registry
poetry config repositories.internal https://pypi.bank.internal/
poetry config http-basic.internal username password

# Build package
poetry build

# Publish to internal registry
poetry publish --repository internal

# Install from internal registry
poetry config repositories.internal https://pypi.bank.internal/
poetry add banking-common --source internal
```

## Common Mistakes

1. **No lock file**: `poetry.lock` not committed, builds not reproducible
2. **Pinning too strictly**: `==1.0.0` instead of `^1.0.0` blocks security updates
3. **Not separating dev/prod**: Dev dependencies in production images
4. **Ignoring Python version**: Not specifying minimum Python version
5. **No multi-stage Docker**: Large images with build tools included
6. **Not using export**: Not exporting requirements.txt for non-Poetry tools

## Interview Questions

1. How do you ensure reproducible builds across development and production?
2. Design a CI/CD pipeline that installs dependencies, runs tests, and builds Docker.
3. What is the difference between `^1.0.0` and `~1.0.0` version constraints?
4. How do you manage internal Python packages across banking teams?
5. Why use multi-stage Docker builds for Python services?

## Cross-References

- `./fastapi.md` - FastAPI service packaging
- `./pytest.md` - Testing with Poetry-managed dependencies
- `../cicd-devops/` - CI/CD integration
- `../infrastructure/` - Docker and Kubernetes deployment

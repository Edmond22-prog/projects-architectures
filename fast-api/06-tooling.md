# Outillage

## uv

- Un seul fichier de dépendances : `pyproject.toml`, verrouillé par `uv.lock` (commité).
- Groupes : `[project.dependencies]` pour le runtime, `[dependency-groups] dev` pour tests et qualité, `tasks` optionnel pour Celery.
- Commandes standard : `uv sync`, `uv run <cmd>`, `uv add <pkg>`. Jamais de `pip install` dans la doc ni les Dockerfiles.
- Version Python épinglée dans `.python-version`.

## Qualité

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "ASYNC"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = ["integration: nécessite des dépendances réelles"]
```

- `ruff` pour lint et format (remplace black, isort, flake8).
- `mypy` optionnel, activé projet par projet ; si activé, `strict = true` sur `services/` et `models/` au minimum.
- `pre-commit` avec ruff est recommandé mais pas imposé.

## Configuration (`settings.py`)

- `pydantic-settings`, une classe `Settings`, instanciée une fois (`settings = Settings()`).
- Toute valeur vient de l'environnement ; `.env` uniquement en local, jamais commité ; `.env.example` commité et à jour.
- Pas de valeur par défaut pour un secret. Une valeur par défaut pour tout ce qui est réglage (limites, délais, périodes).
- Toute variable déclarée est **utilisée**. Une variable lue puis ignorée est un bug.

## Docker

Une image par service, multi-stage :

```dockerfile
FROM python:3.12-slim AS base
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .
ENV PATH="/app/.venv/bin:$PATH"

FROM base AS api
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]

FROM base AS worker
CMD ["celery", "-A", "celery_app", "worker", "-Q", "default", "-l", "info"]
```

- Même image, cibles différentes (`api`, `worker`, `beat`). Pas de divergence de dépendances entre API et worker.
- `docker-compose.yml` local avec la base NoSQL, Redis/Valkey et les cibles du service. Les volumes de données sont nommés, le `storage/` runtime est monté et ignoré par git.
- `/health` est utilisé comme `HEALTHCHECK`.

## CI

Pipeline minimal, chaque étape bloquante :

1. `uv sync --frozen`
2. `uv run ruff check . && uv run ruff format --check .`
3. `uv run pytest -m "not integration"`
4. `docker build --target api .` (et `worker` si présent)
5. Optionnel : `uv run pytest -m integration` avec services en conteneurs.

Le pipeline tourne sur chaque PR et sur la branche principale. Un merge est bloqué si une étape échoue.

## Dépôt

```text
.env.example
.gitignore            # .env, storage/, .venv, __pycache__
.python-version
CLAUDE.md
Dockerfile
docker-compose.yml
docs/architecture/
pyproject.toml
uv.lock
```

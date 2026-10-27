# Règles du service — FastAPI

Ce fichier est lu par les agents de code avant toute modification. Les règles sont impératives. La documentation détaillée est dans `docs/architecture/`.

## Blocs activés dans ce projet

Cocher ce qui s'applique. Un bloc non coché ne doit pas être introduit sans décision documentée dans `docs/architecture/decisions.md`.

- [ ] Tâches de fond (Celery + Redis/Valkey) → `tasks/`, `celery_app.py`
- [ ] Multi-tenant (`customer_id` sur chaque écriture)
- [ ] Authentification inter-services (`X-Internal-Key`)
- [ ] Authentification utilisateur (JWT/OAuth2)
- [ ] Événements inter-services (Streams `events:<service>`)
- [ ] Lib partagée (`libs/common` ou package interne)

Base de données : `________` (Mongo / Elasticsearch / JSON local / autre)

## Couches

- `webapp/` : HTTP uniquement. Valide, authentifie, appelle un service, répond.
- `services/` : métier et accès données. Aucun import de FastAPI. Chaque collection appartient à un seul service.
- `models/` : Pydantic, enums, documents persistés. Aucune logique.
- `tasks/` : orchestration de traitements longs via `services/`. Jamais de logique dupliquée.
- Un accès base en dehors de `services/` est une erreur.

## Réponses HTTP

- JSON brut, pas d'enveloppe `{success, data, error}`.
- `response_model` Pydantic sur chaque route ; `id` et non `_id` ; dates ISO 8601 UTC.
- Routes clients sous `/v1/...`, routes inter-services sous `/internal/...`, `/health` public.
- Opération longue : persister `PENDING`, enfiler, répondre `202 {id, status}`, exposer un `GET` de suivi.
- Ne jamais documenter une route comme anonyme parce qu'un Gateway est devant.

## Erreurs

- Les services lèvent `ServiceError` et ses sous-classes (`NotFound`, `ValidationError`, `Conflict`, `Forbidden`, `UpstreamError`) depuis `services/errors.py`.
- Le gestionnaire global de `webapp/errors.py` traduit en HTTP. Une route ne traduit pas elle-même.
- Format d'erreur : `{"detail": "...", "code": "..."}`.
- Aucune trace ni secret dans une réponse ou un log.

## Dépendances et configuration

- `get_<x>()` (singleton, API) et `make_<x>()` (instance neuve, tâches). Une tâche ferme sa ressource dans un `finally`.
- `settings.py` avec `pydantic-settings`. Toute variable déclarée est utilisée. Pas de défaut pour un secret.
- Clients externes dans `services/clients/`, timeout explicite, erreurs converties en `UpstreamError`.

## Tâches de fond (si activé)

- Nom `<service>.<verbe>_<objet>`, module dans `include`, file dans `task_routes`.
- Arguments = primitives JSON. Binaires relus depuis le stockage via `storage_ref`.
- Statuts `PROCESSING` / `DONE` / `ERROR` persistés. Aucun chemin ne laisse une ressource en cours.
- Idempotente sur rejeu, ou protégée par un verrou avec expiration.
- `reject_on_worker_lost=False` + gestionnaire `task_failure` pour les tâches de lot ; défaut sinon. `visibility_timeout` > `task_time_limit`.
- Échec de publication d'événement : journalisé, non bloquant.

## Multi-tenant (si activé)

- Tenant obligatoire en paramètre de chaque méthode de service qui lit ou écrit. Pas de défaut.
- Ressource d'un autre tenant → `404`.

## Événements (si activé)

- Type `<domaine>.<action>.v<N>`. Rupture de schéma → `N+1`, ancien type maintenu en transition.
- Chaque flux long : `started`, `completed`, `failed`.

## Tests

- `uv run pytest` passe sans aucune dépendance vivante. Fakes en mémoire dans `tests/fakes/`.
- Toute modification ajoute un test au niveau touché (service, endpoint ou coroutine de tâche).
- Scénarios minimaux : nominal, erreurs métier avec code HTTP, isolement tenant, statut `ERROR` persisté, rejeu idempotent.

## Avant de livrer

```bash
uv run ruff check . && uv run ruff format --check . && uv run pytest
```

## Ce que l'agent ne fait pas

- Introduire une dépendance, un bloc optionnel ou une collection sans la documenter.
- Modifier une réponse `/v1` de façon incompatible : créer `/v2`.
- Mettre un objet Pydantic ou un binaire dans un argument de tâche.
- Lire une variable de configuration sans l'honorer.
- Contourner `services/` pour accéder aux données.

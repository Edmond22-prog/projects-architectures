# Tests

## Lancer la suite

```bash
uv run pytest            # suite complète
uv run pytest -x -q      # arrêt au premier échec
uv run pytest -k dce     # sous-ensemble
```

`pyproject.toml` déclare `pytest`, `pytest-asyncio` et `httpx` en dépendances de développement. `asyncio_mode = "auto"` est fixé dans la configuration pytest : les tests `async def` s'exécutent sans décorateur.

## Principe

La suite s'exécute **sans aucune dépendance vivante** : ni Mongo, ni Elasticsearch, ni Redis/Valkey, ni service aval. Tout est remplacé par des doubles. Un test qui a besoin d'une vraie base est marqué `@pytest.mark.integration` et exclu par défaut.

## Organisation

```text
tests/
├── conftest.py              # sys.path, fixtures communes, fakes
├── fakes/
│   ├── database.py          # base NoSQL minimale en mémoire
│   ├── storage.py           # stockage en mémoire
│   ├── bus.py               # bus d'événements qui enregistre les publications
│   └── clients.py           # clients externes scriptables
├── test_<domaine>_scenarios.py
└── integration/             # [optionnel] tests contre des conteneurs réels
```

Convention de nommage : `test_<domaine>_scenarios.py`. Un fichier par domaine, les scénarios sont des fonctions nommées par comportement (`test_upload_rejects_oversized_file`).

## Trois niveaux de test

| Niveau | Cible | Comment |
| --- | --- | --- |
| Service | `services/` | Instancier le service avec les fakes, appeler les méthodes, vérifier l'état du fake |
| Endpoint | `webapp/` | `httpx.AsyncClient` + `app.dependency_overrides` pour injecter fakes et principal |
| Tâche | `tasks/` | Appeler la coroutine interne `_run(...)` directement, jamais via un worker |

Les tâches Celery ne sont pas testées via `.delay()` ni `CELERY_ALWAYS_EAGER`. On teste la coroutine et on vérifie les statuts persistés dans le fake.

## Fakes plutôt que mocks

- Un **fake** implémente le même contrat que la vraie dépendance avec un stockage en mémoire (dict). Il est réutilisable et rend les tests lisibles.
- Un **mock** (`unittest.mock`) est réservé aux appels ponctuels dont seul le fait d'avoir été appelé compte.
- Les fakes exposent des helpers d'inspection (`fake_bus.published`, `fake_db.collection("x").docs`).

## Scénarios obligatoires

Toute fonctionnalité livrée couvre au minimum :

- Le chemin nominal.
- Les erreurs métier attendues et leur code HTTP (`404`, `409`, `400`).
- L'isolement tenant en multi-tenant : une ressource d'un autre tenant renvoie `404`.
- Pour une tâche : statut `ERROR` persisté en cas d'exception, idempotence sur rejeu, verrou périmé repris.
- Pour un lot : un élément en échec n'empêche pas les autres.
- Pour une perte de worker : les marqueurs « en cours » sont basculés.
- Pour un upload : taille, type MIME, doublons.
- Pour un compare-and-set : conflit détecté → `409`.

## Authentification dans les tests

`conftest.py` fournit deux fixtures : `internal_client` (en-têtes internes valides, tenant de test) et `user_client` (principal utilisateur injecté via `dependency_overrides`). Un test vérifie explicitement qu'une route refuse l'accès sans auth (`401`).

## Avant livraison

1. Ajouter un test au niveau du composant modifié.
2. `uv run pytest` complet, vert.
3. `uv run ruff check . && uv run ruff format --check .` (voir `06-tooling.md`).

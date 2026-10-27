# Architecture générique FastAPI

Ce dossier décrit le socle commun à tous les projets FastAPI. Il est prescriptif : un agent de code ou un développeur qui crée ou modifie un service doit s'y conformer. Les écarts sont possibles mais doivent être documentés dans le `docs/architecture/` du projet concerné.

## Principes

1. **Trois couches, un sens unique** : `webapp/` (HTTP) → `services/` (métier + données) → `models/` (schémas). Les tâches de fond (`tasks/`) sont un second point d'entrée qui appelle `services/`, jamais `webapp/`.
2. **Accès données uniquement dans `services/`**. Aucune requête Mongo/Elasticsearch/fichier JSON dans une route ou une tâche.
3. **JSON brut en réponse**, pas d'enveloppe. Les erreurs passent par un gestionnaire global.
4. **Tout est asynchrone côté API** (`async def`, clients async). Les tâches de fond sont synchrones à l'entrée et exécutent une coroutine.
5. **Blocs optionnels** : tâches de fond, multi-tenant, événements inter-services, lib partagée. Chaque bloc est activé explicitement et documenté quand il l'est.
6. **Explicite plutôt qu'implicite** : statut persistant, événement terminal, stratégie de reprise, propriétaire d'une donnée — tout est nommé avant d'être codé.

## Arborescence de référence

```text
<service>/
├── app.py              # Fabrique FastAPI, montage des routeurs, gestionnaires d'erreurs, lifespan
├── settings.py         # Pydantic Settings, lecture des variables d'environnement
├── deps.py             # Fabriques : base de données, stockage, clients externes, bus
├── celery_app.py       # [optionnel] Configuration Celery
├── webapp/
│   ├── auth.py         # Dépendances d'authentification (interne et/ou utilisateur)
│   ├── errors.py       # Exceptions métier → réponses HTTP
│   └── endpoints/      # Un module par domaine : dces.py, users.py, ...
├── services/           # Un module par domaine ; porte l'accès données et la logique métier
├── models/             # Schémas Pydantic, enums, documents persistés
├── tasks/              # [optionnel] Tâches Celery, une par module
├── tests/
│   ├── conftest.py
│   └── test_*_scenarios.py
├── docs/architecture/  # Copie adaptée de ce dossier + décisions spécifiques
├── Dockerfile
├── pyproject.toml      # Géré par uv
└── CLAUDE.md           # Règles cibles pour les agents de code
```

## Fichiers de ce dossier

| Fichier | Contenu |
| --- | --- |
| [`01-layers.md`](01-layers.md) | Couches, responsabilités, accès données, flux |
| [`02-api-contract.md`](02-api-contract.md) | Conventions HTTP, authentification, codes, versionnement |
| [`03-exceptions.md`](03-exceptions.md) | Hiérarchie d'erreurs, gestionnaire global, échecs de tâches |
| [`04-background-tasks.md`](04-background-tasks.md) | Celery/Redis (ou Valkey), files, idempotence, événements optionnels |
| [`05-testing.md`](05-testing.md) | Suite de tests, doubles, conventions |
| [`06-tooling.md`](06-tooling.md) | uv, Docker, CI, qualité |
| [`CLAUDE.md`](CLAUDE.md) | Règles condensées à placer à la racine d'un service |

## Comment utiliser ce socle sur un nouveau projet

1. Copier `CLAUDE.md` à la racine du service et cocher les blocs optionnels activés.
2. Copier ce dossier dans `docs/architecture/` et remplacer les exemples par les domaines réels.
3. Ajouter un fichier `decisions.md` pour tout écart ou choix spécifique (base retenue, stratégie d'auth, événements).

# Couches et responsabilités

## Les trois couches

| Couche | Répertoire | Responsabilité | Interdit |
| --- | --- | --- | --- |
| HTTP | `webapp/` | Valider les entrées, authentifier, appeler un service, mettre en file, renvoyer une réponse | Accès données, logique métier, appel direct à un client externe |
| Métier + données | `services/` | Règles métier, accès à la base, stockage, clients externes | Dépendre de FastAPI (`Request`, `HTTPException`), connaître le format HTTP |
| Schémas | `models/` | Pydantic : entrées, sorties, documents persistés, enums | Logique, I/O |

Les tâches de fond (`tasks/`) sont une quatrième zone : elles orchestrent des traitements longs en appelant `services/`. Elles ne dupliquent jamais la logique d'un service.

## Règles d'accès aux données

- Chaque collection/index/fichier est possédé par **un seul service** (`services/<domaine>.py`). Un autre service qui a besoin de ces données passe par ce service, pas par la base.
- Les identifiants internes (`_id` Mongo, `_id` Elasticsearch) sont convertis en `id` par les modèles ou les services avant toute exposition.
- Les classes de service reçoivent leurs dépendances (base, stockage, clients) par le constructeur. Elles ne les instancient pas elles-mêmes.

## Fabriques (`deps.py`)

Deux fabriques distinctes par ressource partagée, parce que l'API et les tâches n'ont pas le même cycle de vie :

| Fabrique | Usage | Cycle de vie |
| --- | --- | --- |
| `get_<ressource>()` | API FastAPI, via `Depends` | Singleton créé au `lifespan`, fermé à l'arrêt |
| `make_<ressource>()` | Tâches de fond, scripts | Instance neuve, fermée dans un `finally` |

Exemples : `get_database()` / `make_database()` pour Mongo (Motor), `get_search()` / `make_search()` pour Elasticsearch (`AsyncElasticsearch`), `get_storage()` / `make_storage()` pour le stockage de fichiers.

Le backend réel est choisi par `settings.py`. Si une variable de configuration existe (`STORAGE_BACKEND`, `DB_BACKEND`), le code **doit** l'honorer : pas de valeur lue puis ignorée.

## Stockage NoSQL : conventions par moteur

| Moteur | Client | Points d'attention |
| --- | --- | --- |
| MongoDB | Motor | Index déclarés dans un module `services/indexes.py` et créés au démarrage ; compare-and-set sur les documents à état |
| Elasticsearch / OpenSearch | client async officiel | Mappings versionnés ; alias de lecture/écriture pour réindexer sans coupure |
| Fichiers JSON / stockage local | module `services/storage.py` | Écriture atomique (fichier temporaire + rename) ; jamais de chemin absolu dans les données persistées, seulement une `storage_ref` |

## Multi-tenant (bloc optionnel)

Quand un produit est multi-tenant :

- Chaque document persisté porte `customer_id` (ou l'équivalent choisi par le projet).
- Chaque méthode de service qui lit ou écrit prend le tenant en paramètre **obligatoire**. Pas de valeur par défaut.
- Le tenant est extrait dans `webapp/auth.py` (en-tête, claim JWT ou chemin) et injecté via `Depends`.
- Les tâches reçoivent le tenant en argument explicite.

Quand un produit ne l'est pas, on n'ajoute pas de `customer_id` « au cas où ».

## Flux de référence

```text
HTTP → webapp/endpoints → services → base NoSQL / stockage
                         ↘ tâche de fond → services → base / stockage → [événements]
```

Les binaires ne transitent jamais par la file de tâches : la route enregistre le fichier via le stockage, puis transmet sa `storage_ref`. Le worker relit le binaire.

## Lib partagée (bloc optionnel)

Un utilitaire est extrait dans une lib partagée (`libs/common` ou package interne) **uniquement** s'il est utilisé par au moins deux services en même temps. Candidats typiques : clients HTTP inter-services, validation de clé interne, bus d'événements, helpers Pydantic.

Un projet isolé embarque ses utilitaires dans `services/` ou `webapp/`. On ne crée pas de lib partagée en anticipation.

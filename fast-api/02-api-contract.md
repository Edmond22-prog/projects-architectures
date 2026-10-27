# Contrat HTTP

## Conventions de réponse

- **JSON brut.** Pas d'enveloppe `{success, data, error, meta}`. Une ressource est renvoyée telle quelle, une liste est un tableau ou un objet `{items, total, next_cursor}` quand la pagination l'exige.
- Chaque route déclare un `response_model` Pydantic. Les `dict` nus sont tolérés uniquement sur les routes techniques (`/health`).
- Les identifiants sont exposés sous `id`, jamais `_id`.
- Les dates sont en ISO 8601 UTC avec suffixe `Z`.
- Les messages d'erreur (`detail`) sont en anglais, courts, sans trace technique.

## Authentification : deux modes, un seul module

`webapp/auth.py` expose deux dépendances. Un projet active l'une, l'autre ou les deux.

| Mode | Dépendance | Mécanisme | Erreurs |
| --- | --- | --- | --- |
| Inter-services | `require_internal` | `X-Internal-Key` comparé à `INTERNAL_API_KEY` (comparaison en temps constant) | `401` clé absente/invalide |
| Utilisateur | `require_user` | Bearer JWT (ou OAuth2) vérifié localement ou via un fournisseur | `401` jeton absent/invalide, `403` droits insuffisants |

En multi-tenant, les deux dépendances renvoient un objet `Principal` qui porte `tenant_id` et, pour un utilisateur, `user_id` et `roles`. Pour le mode interne, le tenant vient de `X-Customer-Id` (`400` si absent).

Règles :

- Une route non authentifiée est une exception documentée (`/health`, `/docs`, webhooks signés).
- Quand un Gateway authentifie l'utilisateur en amont, le service exige **quand même** les en-têtes internes. Ne jamais documenter une route comme anonyme parce qu'elle est derrière un Gateway.
- `/docs`, `/redoc`, `/openapi.json` sont désactivés en production (`DOCS_ENABLED=false`) sauf besoin explicite.

## Organisation des routes

```text
/health                       # public, liveness/readiness
/v1/<ressource>               # routes exposées aux clients (via Gateway ou direct)
/internal/<ressource>         # routes réservées aux autres services
```

- Un routeur par domaine dans `webapp/endpoints/`, préfixe et tags définis dans le module, montage dans `app.py`.
- Les routes `/internal/...` ne sont jamais exposées par le Gateway.
- Versionnement par préfixe `/v1`. Une rupture de contrat crée `/v2` ; on ne modifie pas une réponse existante de façon incompatible.

## Codes HTTP

| Code | Usage |
| --- | --- |
| `200` | Lecture, mise à jour, action synchrone terminée |
| `201` | Création d'une ressource |
| `202` | Traitement mis en file ; la réponse contient un identifiant de suivi |
| `204` | Suppression réussie sans corps |
| `400` | Entrée invalide, état incompatible, tenant absent |
| `401` | Non authentifié |
| `403` | Authentifié mais non autorisé |
| `404` | Ressource absente **dans le périmètre du tenant** (ne pas distinguer « existe ailleurs ») |
| `409` | Conflit : opération déjà en cours, compare-and-set échoué, doublon |
| `422` | Réservé à la validation Pydantic automatique de FastAPI |
| `502` | Dépendance aval en erreur |
| `503` | Service non prêt (readiness) |

## Opérations longues

Une route qui déclenche une tâche de fond :

1. Valide, persiste l'état initial (`status=PENDING`), enfile la tâche.
2. Renvoie `202` avec `{id, status}`.
3. Expose une route de suivi `GET /v1/<ressource>/{id}` qui reflète le statut persisté.

Le client ne doit jamais rester bloqué dans un état « en cours » : toute tâche a un état terminal (voir `03-exceptions.md`).

## Idempotence

- `PUT` est idempotent par construction. Une réécriture avec un contenu canonique identique renvoie `200` ; un contenu divergent sur une ressource immuable renvoie `409`.
- `POST` de création accepte un en-tête `Idempotency-Key` optionnel quand des rejeux sont possibles (webhooks, uploads).

## Pagination

Curseur opaque (`next_cursor`) par défaut. `limit` plafonné côté serveur (valeur dans `settings.py`). L'offset est réservé aux petites listes bornées.

## OpenAPI

L'OpenAPI générée par FastAPI est la référence de schéma. Les docs d'architecture décrivent les conventions et les flux, pas le détail de chaque paramètre.

# Erreurs et échecs de traitement

## Hiérarchie d'exceptions métier

Les services lèvent des exceptions **indépendantes de HTTP**, définies dans `services/errors.py` :

```python
class ServiceError(Exception):
    """Base. Porte un `code` stable et un `message` lisible."""
    code = "service_error"

class NotFound(ServiceError):        code = "not_found"
class ValidationError(ServiceError): code = "validation_error"
class Conflict(ServiceError):        code = "conflict"
class Forbidden(ServiceError):       code = "forbidden"
class UpstreamError(ServiceError):   code = "upstream_error"   # dépendance aval
```

Un service n'importe jamais `fastapi.HTTPException`. Un domaine peut dériver ses propres sous-classes (`DceNotFound(NotFound)`), mais toujours sous l'une des bases ci-dessus.

## Gestionnaire global (`webapp/errors.py`)

`app.py` enregistre un gestionnaire par base, qui traduit vers HTTP :

| Exception | Code HTTP |
| --- | --- |
| `NotFound` | `404` |
| `ValidationError` | `400` |
| `Conflict` | `409` |
| `Forbidden` | `403` |
| `UpstreamError` | `502` |
| `ServiceError` (autre) | `500`, message générique, détail dans les logs |

Format de réponse d'erreur, uniforme :

```json
{"detail": "Resource not found", "code": "not_found"}
```

Les `HTTPException` restent possibles dans `webapp/` pour les cas purement HTTP (auth, en-têtes), mais une route ne doit pas traduire elle-même une erreur de service : c'est le rôle du gestionnaire.

Les exceptions non prévues sont journalisées avec la trace et renvoient `500` sans fuite d'information. En développement, `DEBUG=true` peut réactiver la trace dans la réponse.

## Échecs de dépendances aval

Les clients externes (autre service, SharePoint, OCR, fournisseur d'auth) sont encapsulés dans `services/clients/`. Chaque client :

- convertit ses erreurs transport/HTTP en `UpstreamError` (ou `NotFound` si la ressource aval est absente et que c'est le sens attendu) ;
- applique un timeout explicite ;
- ne relance pas lui-même : les retries sont décidés par l'appelant (tâche) et bornés.

## Tâches de fond (bloc optionnel)

Chaque tâche est synchrone à l'entrée et appelle une coroutine via `asyncio.run()`. Règles :

1. **État terminal garanti.** Si le corps échoue, la tâche persiste un statut `ERROR` (ou `FAILED`) sur la ressource concernée et, si des événements sont activés, publie l'événement terminal d'échec. Aucun chemin ne laisse une ressource en `PROCESSING`.
2. **Granularité d'échec choisie.** Une tâche par lot isole les échecs par élément : un document raté n'arrête pas le lot et porte son propre marqueur d'erreur. Une tâche unitaire bascule la ressource entière.
3. **Perte de worker traitée.** Un gestionnaire `task_failure` (ou `worker_lost`) rebascule les marqueurs laissés « en cours » vers un état terminal. Voir `04-background-tasks.md` pour les réglages `acks_late` / `reject_on_worker_lost`.
4. **Échecs secondaires non bloquants.** Une publication d'événement ou une notification qui échoue est journalisée et n'annule pas le travail déjà persisté.
5. **Suppression tolérante.** Les suppressions de fichiers renvoient `False` en cas d'absence au lieu de lever ; le nettoyage est idempotent.

## Journalisation

- Un logger par module (`logging.getLogger(__name__)`), format JSON en production.
- Chaque log d'erreur porte l'identifiant de ressource, le tenant si applicable et un `request_id` / `task_id` de corrélation.
- Pas de secret ni de contenu de document dans les logs.

# Tâches de fond, files et événements

Bloc **optionnel**. Il est activé quand le service doit exécuter des traitements longs, planifiés ou découplés de la requête HTTP. Sinon, `tasks/` et `celery_app.py` n'existent pas.

## Quand l'activer

| Besoin | Solution |
| --- | --- |
| Action courte (< 1 s), sans garantie de reprise | `BackgroundTasks` de FastAPI, dans la route |
| Traitement long, lot, planification, reprise après crash | Celery + Redis (ou Valkey) |

On n'introduit pas Celery pour un envoi de mail ponctuel. On n'utilise pas `BackgroundTasks` pour un traitement qui doit survivre à un redémarrage.

## Configuration Celery de référence (`celery_app.py`)

```python
app = Celery("<service>", broker=settings.BROKER_URL, backend=settings.BROKER_URL)
app.conf.update(
    task_serializer="json", result_serializer="json", accept_content=["json"],
    task_acks_late=True,
    task_reject_on_worker_lost=True,
    worker_prefetch_multiplier=1,
    worker_max_tasks_per_child=50,
    task_soft_time_limit=settings.TASK_SOFT_LIMIT,   # ex. 6900
    task_time_limit=settings.TASK_HARD_LIMIT,        # ex. 7200
    broker_transport_options={"visibility_timeout": settings.VISIBILITY_TIMEOUT},  # > hard limit
    include=["tasks.<module>", ...],
    task_routes={"<service>.<tache>": {"queue": "<file>"}, ...},
)
```

Redis et Valkey sont interchangeables ici : même URL `redis://`, même transport. Le choix est porté par `BROKER_URL`.

### Réglages par tâche

- Le défaut (`acks_late` + `reject_on_worker_lost`) redélivre une tâche après perte de worker. C'est le bon choix pour une tâche **idempotente et unitaire**.
- Une tâche de **lot** qui marque chaque élément « en cours » doit surcharger `reject_on_worker_lost=False` et s'appuyer sur un gestionnaire `task_failure` pour basculer les marqueurs vers un état terminal. Sinon, une redélivrance infinie est possible.
- `visibility_timeout` doit dépasser `task_time_limit`, sinon le broker redélivre une tâche encore en cours.

## Conventions de nommage

- Nom de tâche : `<service>.<verbe>_<objet>` (ex. `documents.chunk_dce`, `documents.sync_bible`).
- Une file par nature de charge : `default`, `heavy` (CPU/longue), `io` (synchronisations externes), `scheduled`. Un projet peut en nommer d'autres mais chaque file est listée dans `task_routes`.
- Un module par domaine dans `tasks/`.

## Données transmises à une tâche

- **Primitives JSON uniquement** : identifiants, tenant, références de stockage, listes d'identifiants. Jamais d'objet Pydantic, jamais de binaire.
- Le worker relit tout ce dont il a besoin depuis la base ou le stockage via `services/`.
- Signature type : `task(resource_id: str, tenant_id: str | None = None, **options)`.

## Cycle de vie d'une tâche

```python
@app.task(name="<service>.<tache>", bind=True)
def task(self, resource_id: str, tenant_id: str) -> None:
    asyncio.run(_run(resource_id, tenant_id))

async def _run(resource_id, tenant_id):
    db = make_database()
    try:
        service = XService(db, ...)
        await service.set_status(tenant_id, resource_id, "PROCESSING")
        await service.process(tenant_id, resource_id)
        await service.set_status(tenant_id, resource_id, "DONE")
    except Exception:
        await service.set_status(tenant_id, resource_id, "ERROR")
        raise
    finally:
        db.client.close()
```

Obligations :

1. `make_<ressource>()` (instance neuve), fermée dans `finally`.
2. Statut persistant à l'entrée, à la sortie et en cas d'échec.
3. Idempotence : rejouer la tâche sur une ressource déjà `DONE` est un no-op ou une reprise sûre.
4. Verrou pour les tâches exclusives (synchronisations) : verrou en base avec expiration, libéré en `finally`, considéré périmé après `LOCK_TTL`.

## Planification (Celery Beat)

- Déclarée dans `celery_app.py` (`beat_schedule`), période lue depuis `settings.py`.
- Une tâche planifiée qui enfile des sous-tâches vérifie le verrou avant d'enfiler.
- Attention aux tenants : une planification qui itère sur des tenants « connus » ne découvre pas un tenant jamais vu. Documenter la source de vérité des tenants.

## Événements inter-services (bloc optionnel)

Activé uniquement dans une architecture multi-services qui a besoin de notifier. Convention recommandée si activé :

- Transport : Redis/Valkey Streams, un stream par service producteur (`events:<service>`).
- Type : `<domaine>.<action>.v<N>` ; une rupture de schéma incrémente `N`, l'ancien type reste émis pendant une période de transition.
- Enveloppe : `type`, `event_id`, `ts`, `resource_id`, `payload`, plus `tenant_id` en multi-tenant.
- Chaque flux long émet au minimum un événement de début, un de fin et un d'échec (`<x>.started.v1`, `<x>.completed.v1`, `<x>.failed.v1`). Le `progress` est optionnel.
- La publication est non bloquante : un échec du bus est journalisé, le traitement persisté reste valide.
- Le producteur et les consommateurs sont listés dans `docs/architecture/decisions.md` du projet.

Sans ce bloc, le suivi d'état passe uniquement par les routes `GET` de statut.

## Checklist pour une nouvelle tâche

- [ ] Module ajouté à `include`, nom ajouté à `task_routes`, file choisie
- [ ] Arguments = primitives JSON
- [ ] `make_*()` + `finally`
- [ ] Statuts `PROCESSING` / `DONE` / `ERROR` persistés
- [ ] Comportement défini en cas de redélivrance (idempotence ou verrou)
- [ ] Réglage `reject_on_worker_lost` justifié
- [ ] Événement terminal si le bloc événements est actif
- [ ] Test au niveau de la coroutine interne (`05-testing.md`)

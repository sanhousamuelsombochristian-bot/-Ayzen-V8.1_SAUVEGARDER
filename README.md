# AyzenAI — workspace Phase 1 / Phase 2 draft

## Statut
- Phase 1 : déclarée validée par l’utilisateur selon ses résultats; le rapport distingue les résultats rapportés des mesures non fournies.
- Migration `0002_phase2_app.py` : brouillon préparé, **non appliqué à aucune base**.
- Aucun backend FastAPI, route métier, frontend, RAG, entraînement ni application Phase 2 n’est présent.

## Protection des bases
- Les migrations/tests de développement doivent viser exclusivement la base locale `ayzenai_test`.
- Le test `tests/test_migration_schema.py` est destructif pour cette base : il exige le rôle et la base exacts `ayzenai_test`, hôte loopback et port 5432.
- Ne jamais exécuter un downgrade sur `ayzenai`; `safrica_digits` et `verqia_pilot` sont hors périmètre.
- `0002` ne doit être appliquée à `ayzenai` qu’après tests réussis, backup vérifié et approbation explicite.

## Contenu
- `0001_phase1_core.py` : migration de fondation 11 tables. Ne pas modifier après application.
- `0002_phase2_app.py` : mapping `ayzen_omni_light` vers `qwen2.5:1.5b`, projets/memberships, backfill protégé, tokens d’action d’auth et usage; brouillon non appliqué.
- `tests/test_migration_schema.py` : test de migration avec conversation et inférence préexistantes, strictement limité à `ayzenai_test`.
- `PHASE2-PLAN.md` : périmètre, contrats API, contrôles d’accès, tests et critères d’acceptation.
- `docs/phase2-migration-audit.md` : audit statique, limites de couverture et requêtes read-only d’audit des droits.

## Exécution locale contrôlée
Depuis cette racine, activer le venv, définir `AYZENAI_TEST_DATABASE_URL` vers `ayzenai_test`, vérifier visuellement l’URL, puis lancer uniquement :

```powershell
pytest -q -p no:cacheprovider tests/test_migration_schema.py
```

Ce projet exporté n’affirme pas que la migration a été testée ou appliquée sur la machine cible.

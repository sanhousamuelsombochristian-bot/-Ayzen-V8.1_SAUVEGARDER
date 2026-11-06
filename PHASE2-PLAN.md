# AyzenAI — audit statique de `0002_phase2_app` et `test_migration_schema.py`

**Nature de l’audit : lecture du code livré dans l’espace de travail, sans connexion à PostgreSQL local.** L’utilisateur rapporte `1 passed in 7.13s`; je n’ai pas relancé ce test sur son PC. Cet audit décrit ce que le code affirme et les lacunes que le résultat d’un seul test ne couvre pas.

## Fichiers examinés

- `database/migrations/versions/0002_phase2_app.py`
- `tests/test_migration_schema.py`
- `database/migrations/env.py`
- `alembic.ini`
- `PHASE2-PLAN.md`

## Ce que la migration fait

1. Crée `projects`, `project_memberships` et `auth_action_tokens`.
2. Ajoute les colonnes Phase 2 aux utilisateurs, sessions, conversations et runs.
3. Crée un projet personnel et un membership `owner` par utilisateur existant.
4. Rattache les conversations à leur propriétaire, refuse de continuer s’il reste une conversation orpheline, puis rattache les runs via la conversation ou l’utilisateur.
5. Ajoute la FK et le `NOT NULL` de `conversations.project_id` seulement après le backfill.
6. Change `ayzen_omni_light` uniquement si sa valeur courante est exactement `qwen3:4b`; la migration échoue plutôt que d’écraser une autre valeur. Le downgrade rétablit `qwen3:4b` uniquement si la valeur est encore `qwen2.5:1.5b`.
7. Le downgrade refuse certains cas de données Phase 2 (projets non personnels, memberships supplémentaires/révoqués, comptes vérifiés, familles de sessions tournées et usage/idempotence non nul), puis supprime les objets Phase 2. Ce garde-fou n’est pas un substitut à une sauvegarde.

## Ce que le test vérifie

- Le garde `_test_url()` refuse les noms de base autres que `ayzenai_test` et les hôtes non-loopback.
- La configuration Alembic du test reçoit explicitement `AYZENAI_TEST_DATABASE_URL`; `use_config_url` empêche `DATABASE_URL` de la base principale de la remplacer dans `env.py`.
- Au démarrage, le test exécute `downgrade base` sur cette base seulement, monte `0001`, crée un utilisateur, une conversation et un run de test, puis monte `head`.
- Il vérifie 11 tables Phase 1, 14 tables métier au head, plusieurs colonnes, index, FK et contraintes, le mapping léger `qwen2.5:1.5b`, un rattachement à un projet, zéro conversation orpheline, un downgrade vers 0001 et une nouvelle montée à head.
- Il laisse normalement la base de test à head. Il ne migre pas `ayzenai` ni les bases `safrica_digits`/`verqia_pilot` lorsque l’URL est correctement configurée.

## Lacunes à corriger ou accepter explicitement avant application à `ayzenai`

1. **Garde de cible renforcée dans la version actuelle du test.** Elle accepte uniquement le rôle `ayzenai_test`, la base `ayzenai_test`, l’hôte loopback et le port 5432; des tests purs vérifient les refus de `ayzenai`, `safrica_digits`, `verqia_pilot`, hôtes distants, port différent et rôle `postgres`. Ces nouvelles assertions sont syntaxiquement vérifiées mais le test doit être relancé sur le PC.
2. **Backfill ownership renforcé.** Le test compare `projects.owner_user_id` avec l’UUID du user fixture, vérifie que la conversation et le run pointent sur le même projet, et compare le titre.
3. **Préservation après downgrade renforcée.** Le test compare maintenant le propriétaire/titre de conversation et le propriétaire/conversation/status d’inférence, pas seulement les nombres de lignes.
4. **Refus de downgrade ajouté.** Le test rend un utilisateur vérifié, exige le refus du downgrade, puis vérifie que la révision reste `0002` et que la donnée n’a pas été perdue. Les autres prédicats du garde (projet non personnel, membership, rotation, usage) ne sont pas tous testés indépendamment.
5. **Le mapping en base n’est pas le routage runtime.** Le test vérifie `model_profiles.model_name`; il n’existe pas encore de backend/adaptateur qui prouve qu’une requête logique `ayzen_omni_light` appelle réellement Ollama avec `qwen2.5:1.5b`.
6. **Permissions multi-tenant : pas vérifiables par cette migration.** Les rôles `owner/admin/member` sont des valeurs de données; les contrôles d’accès applicatifs n’existent pas encore. Les tests d’isolation HTTP doivent attendre le backend. UUID seul n’est jamais une autorisation.
7. **Suppression physique risquée :** la FK `conversations.project_id → projects.id` utilise `ON DELETE CASCADE`. Un `DELETE FROM projects` pourrait effacer conversations et, par cascade, messages. Le plan prévoit une suppression logique, mais ce n’est pas une contrainte de base. Avant production, décider d’interdire le hard delete avec `ON DELETE RESTRICT`/`NO ACTION` ou de spécifier une purge explicite, sauvegardée et auditée. Ne pas éditer une migration déjà appliquée en test; si correction nécessaire, la faire dans une nouvelle révision puis la tester.
8. **Les tokens d’action ne sont pas inclus dans le garde de downgrade.** Les supprimer invaliderait des liens de vérification/reset en cours. C’est généralement récupérable (réémission), mais ce comportement doit être documenté/testé.
9. **L’URL est limitée à loopback et au nom, mais pas au port PostgreSQL.** Si plusieurs clusters locaux existent, un autre service sur `localhost:<port>` pourrait partager ce nom. Pour un garde plus strict, exiger le port 5432 attendu ou un identifiant de cluster explicitement configuré.

## Vérification effective des droits PostgreSQL

La migration testée démontre que `ayzenai_test` peut effectuer le DDL requis sur sa propre base. Elle ne prouve pas que les rôles ne peuvent pas se connecter aux trois autres bases. Depuis une session `psql` en superuser sur n’importe quelle base du même cluster, exécuter cette requête en lecture seule :

```sql
SELECT r.rolname, d.datname,
       has_database_privilege(r.rolname, d.datname, 'CONNECT') AS can_connect
FROM pg_roles AS r
CROSS JOIN pg_database AS d
WHERE r.rolname IN ('ayzenai', 'ayzenai_test')
  AND d.datname IN ('ayzenai', 'ayzenai_test', 'safrica_digits', 'verqia_pilot')
ORDER BY r.rolname, d.datname;
```

Cible recommandée : `ayzenai` peut se connecter uniquement à `ayzenai`; `ayzenai_test` uniquement à `ayzenai_test`. Vérifier séparément les droits de schéma/table dans chaque base, sans révoquer quoi que ce soit avant de comprendre les rôles nécessaires aux autres applications :

```sql
-- À exécuter dans chaque base, comme superuser :
SELECT current_database(),
       has_schema_privilege('ayzenai', 'public', 'USAGE') AS app_usage,
       has_schema_privilege('ayzenai', 'public', 'CREATE') AS app_create,
       has_schema_privilege('ayzenai_test', 'public', 'USAGE') AS test_usage,
       has_schema_privilege('ayzenai_test', 'public', 'CREATE') AS test_create;
```

Ne pas coller ici de mot de passe, URL avec secret, token OAuth ou contenu utilisateur.

## Conclusion d’audit

Le test positif rapporté est cohérent avec le chemin nominal du code et confirme le mapping en test. Il ne valide pas encore les refus de mauvais targets, toutes les branches de downgrade, les droits effectifs sur les autres bases ou le routage Ollama par profil logique. La migration n’a pas été appliquée à la base principale par cet audit. Recommandation : ajouter les tests de garde/données, décider du comportement de suppression physique, puis refaire un run complet uniquement sur `ayzenai_test` avant toute demande d’approbation pour `ayzenai`.

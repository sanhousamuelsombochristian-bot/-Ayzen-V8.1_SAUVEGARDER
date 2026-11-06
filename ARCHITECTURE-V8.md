# AyzenAI — Architecture V8, Phase 1

**Statut : Architecture V8 approuvée; Phase 1 déclarée terminée par l’utilisateur selon les résultats réels qu’il a rapportés.** Les faits et les limites de mesure figurent dans [le rapport de validation](docs/hardware-validation-report.md). Je n’ai pas accès au PC Windows pour les vérifier indépendamment. Aucun développement applicatif Phase 2 n’a commencé.

## 1. Périmètre et décisions validées

- Windows 11, 64 Go de RAM, SSD; GPU Intel indiqué mais modèle/VRAM inconnus. Fonctionnement **CPU-first**; aucune accélération graphique n’est présumée.
- Ollama est le moteur local de référence. Un adaptateur isolera le moteur. llama.cpp est une option ultérieure. vLLM/CUDA n’est pas requis ni retenu pour cette machine.
- Quantification initiale Q4_K_M, contexte limité à 8 192 tokens au démarrage (16 384 seulement après mesures), un seul modèle chargé simultanément.
- Aucun entraînement from scratch, fine-tuning, réponse simulée, collecte implicite, affirmation de qualité sans benchmark exécuté, secret codé en dur ou exposition réseau avant durcissement.
- Arena compare et note des réponses fournies au système; elle ne choisit pas d’outils et ne mène pas d’actions autonomes.

## 2. Rôle des modèles AyzenAI — sans promesse multimodale prématurée

| Nom logique | Rôle réel en Phase 1 | Configuration candidate | Limite déclarée |
|---|---|---|---|
| `ayzen_omni_light` | Profil logique de secours; mapping actuellement observé en DB | DB : `qwen3:4b`; modèle réellement testé via Ollama : `qwen2.5:1.5b` (986 MB rapportés) | Ces deux faits ne prouvent pas que le profil logique a appelé le modèle testé. Une migration/configuration versionnée doit aligner le mapping; `qwen3:4b` reste indisponible si non installé. |
| `ayzen_omni_text` | Omni One — texte | `qwen3:8b`, Q4_K_M | Modèle texte; ne génère pas d’images ni de vidéo. |
| `ayzen_omni_vision` | Compréhension d’images / pages documentaires | `qwen3-vl:4b`, Q4_K_M | Entrée visuelle; pas de promesse de génération visuelle, de voix ou de compréhension vidéo native en Phase 1. |
| `ayzen_thinking` | Profil raisonnement, maths et analyse | `qwen3:8b` avec paramètres de réflexion testés | Profil de service/évaluation, pas un modèle Ayzen entraîné distinct. Un modèle plus gros n’est retenu qu’après benchmark. |
| `ayzen_arena` | Évaluateur de candidats | Service d’évaluation structuré; peut utiliser des profils disponibles | Produit une comparaison argumentée, pas une vérité objective. |

**Omni complet** — texte, image, voix, documents, génération créative — est une vision de phases ultérieures composée de capacités distinctes. Traitement vidéo futur = extraction de frames puis analyse par lots; ce n’est pas une génération/compréhension vidéo native.

Les modèles Qwen3 mentionnés dans la table (8B texte et VL 4B) restent des candidats à tester, pas des modèles déclarés prêts. À titre de comparaison, leurs pages Ollama indiquent environ 5,2 Go pour `qwen3:8b` [Ollama](https://ollama.com/library/qwen3:8b) et 3,3 Go pour `qwen3-vl:4b` [Ollama](https://ollama.com/library/qwen3-vl:4b). Ces chiffres décrivent la taille des poids, pas la mémoire de fonctionnement, la vitesse ou la qualité sur ce PC. L’utilisateur rapporte que `qwen2.5:1.5b` répond réellement, mais la requête DB confirme que le profil `ayzen_omni_light` pointe encore sur `qwen3:4b`; l’association du profil au modèle réellement testé reste à corriger par migration/configuration versionnée.

### Règle de sélection et de repli
1. Tester d’abord `ayzen_omni_light`; il est obligatoire dans la configuration.
2. Tester les profils 8B et vision séparément, un par un. Leur présence dans la base ou dans `ollama list` ne signifie pas qu’ils sont prêts.
3. À chaque requête, vérifier la disponibilité réelle. Si le profil demandé est absent, ne passe pas le test de charge ou dépasse le budget de latence configuré, proposer `ayzen_omni_light` avec consentement/configuration explicite ou retourner une erreur claire. Aucun fallback silencieux.
4. La migration enregistre des profils logiques par défaut, **pas** le fait que les poids soient installés ou qu’une génération ait réussi. La readiness est une mesure d’exécution séparée.

## 3. Arborescence cible

```text
AyzenAI/
├── frontend/                  # React + Vite + TypeScript (après approbation)
├── backend/                   # FastAPI (après approbation)
├── inference/                 # interface fournisseur et adaptateurs Ollama
├── database/                  # PostgreSQL, Alembic, tests de schéma
├── storage/                   # futur stockage privé des médias
├── workers/                   # tâches asynchrones, phases ultérieures
├── training/                  # QLoRA/datasets, Phase 3+
├── tests/                     # schéma, sécurité, intégration, benchmark
├── docs/
├── alembic.ini
└── .env.example
```

Monolithe modulaire d’abord. PostgreSQL 18.6 est la version cible installée et rapportée sur le poste; Redis/Celery et Docker Compose ne sont pas des prérequis de Phase 1. L’adaptateur Ollama local tourne côté serveur; le navigateur ne contacte jamais `localhost` directement.

### Compatibilité PostgreSQL 18.6
Les versions rapportées dans le venv Windows (`SQLAlchemy 2.1.6`, `Alembic 1.20.1`, `psycopg 3.3.6`) sont cohérentes avec l’usage PostgreSQL : le dialecte SQLAlchemy documente psycopg 3 [SQLAlchemy](https://docs.sqlalchemy.org/en/21/dialects/postgresql.html), Alembic 1.20 documente sa compatibilité SQLAlchemy 2.x [Alembic](https://alembic.sqlalchemy.org/en/latest/front.html), et Psycopg 3 annonce PostgreSQL 10–18 [Psycopg](https://www.psycopg.org/). Le schéma de Phase 1 n’utilise aucune fonctionnalité spécifique récente et pgvector n’est pas chargé.

**Limite importante :** cette compatibilité documentaire ne prouve pas que cette migration particulière passe sur l’instance locale; seule l’exécution `alembic upgrade/downgrade` et les tests sur PostgreSQL 18.6 le démontreront. Pour la Phase 2, pgvector devra être installé avec un binaire/build correspondant à PostgreSQL 18. Le dépôt pgvector fournit des instructions Windows avec `PGROOT` pointant sur le dossier `PostgreSQL\\18` et le build `nmake`; les tags récents documentent la cible 18 [pgvector](https://github.com/pgvector/pgvector). Cela ne constitue pas une installation ni une activation de l’extension dans cette Phase 1.

## 4. Détection matérielle

Rapport runtime `HardwareReport` : OS et build, architecture, CPU/cœurs, RAM totale/libre, volumes et espace SSD libre, GPU/vendor/modèle/mémoire dédiée ou partagée, backend détecté, versions Ollama, modèles/digests/quantifications installés, puis résultats distincts `detected`, `load_tested`, `inference_tested` avec horodatage et erreur.

Sous Windows, combiner données système/PowerShell et contrôles Ollama; l’accélération Intel n’est pas présumée. Une valeur inconnue reste `unknown`, jamais « GPU absent » par déduction. La readiness de génération est établie uniquement par un test borné réel. Les contrôles santé ordinaires ne chargent pas de gros modèle.

## 5. Schéma PostgreSQL et comptage des tables

### Phase 1 — migration `0001_phase1_core.py`
**11 tables exactement**, sans extension `pgvector` :
1. `users`
2. `auth_identities`
3. `sessions`
4. `consent_records`
5. `model_profiles`
6. `conversations`
7. `messages`
8. `inference_runs`
9. `arena_reviews`
10. `arena_candidates`
11. `audit_events`

UUID, dates UTC, clés étrangères et contraintes de validation. Emails rendus uniques sans tenir compte de la casse. Mots de passe Argon2id calculés par l’application; seuls les refresh tokens hachés sont persistés. Aucun prompt/contenu de réponse dans l’audit.

### Phases ultérieures — prévision explicite, pas encore migrée
- **Phase 2 RAG : +4 tables planifiées**, `storage_objects`, `documents`, `document_chunks`, `embedding_profiles`; total prévisionnel **15**. L’extension `pgvector` et le vecteur d’embedding sont ajoutés sur `document_chunks`; l’extension n’est pas une table.
- **Phase 3 apprentissage : +3 tables planifiées**, `dataset_versions`, `dataset_items`, `training_jobs`; total prévisionnel **18**.
- **Phase 4 évaluation continue : +3 tables planifiées**, `benchmark_suites`, `benchmark_cases`, `benchmark_runs`; total prévisionnel **21**.

Les nombres 15/18/21 sont des prévisions d’architecture, pas un schéma livré. Toute évolution réelle passe par une migration versionnée et révisée. Aucun total « 27 tables » n’est retenu dans cette version.

## 6. Politique de conservation et suppression proposée

Ces durées sont les valeurs par défaut à valider avant toute exposition réseau; elles ne constituent pas un avis juridique. Déploiement public soumis à la revue de conformité applicable en Côte d’Ivoire et dans les marchés visés.

| Donnée | Conservation par défaut | Suppression / retrait |
|---|---|---|
| Conversations et messages | 90 jours après la dernière activité | Export JSON sur demande; suppression manuelle immédiate de l’accès, purge primaire sous 24 h; purge automatique au terme des 90 jours. |
| Fichiers futurs | Tant qu’ils sont rattachés à une conversation active, maximum 90 jours sans activité | Effacement de l’objet, dérivés et miniatures à la suppression; traitement temporaire purgé sous 24 h. |
| Journaux techniques | 30 jours, sans prompts, réponses, clés ni données personnelles non nécessaires | Rotation automatique; demande de suppression appliquée aux identifiants joignables. |
| Audit sécurité | 180 jours, métadonnées minimales seulement | Accès restreint; purge automatique après 180 jours. |
| Consentements | Pendant le compte, puis 12 mois après sa fermeture sous forme minimale (finalité/version/date) | Retrait immédiatement appliqué aux collectes futures et aux nouveaux jeux de données. Aucun entraînement Phase 1. |
| Sauvegardes | Rotation maximale de 30 jours, chiffrées et non accessibles en usage normal | Suppression définitive des systèmes actifs sous 24 h; les copies expirent au plus tard sous 30 jours. |

Suppression du compte : export préalable possible, puis purge des données actives sous 24 h et expiration des sauvegardes au plus tard sous 30 jours. Aucun réentraînement ni promesse d’« oubli » de modèles n’est inclus; l’utilisation ultérieure de données pour apprentissage nécessitera consentement/licence, traçabilité et procédure distincte.

## 7. Contrats API `/api/v1/`

JSON/Pydantic v2, UUID texte, horodatages ISO-8601 UTC. Erreur uniforme : `{"error":{"code":"...","message":"...","request_id":"..."}}`.

- `GET /health/live`, `GET /health/ready`
- `GET /system/hardware` (administrateur local)
- `GET /models`
- `POST /auth/register`, `/auth/login`, `/auth/google`, `/auth/refresh`, `/auth/logout`
- `POST /conversations`, `GET /conversations`, `GET /conversations/{id}`, `DELETE /conversations/{id}`, export utilisateur
- `POST /chat/completions` (messages, modèle logique autorisé, limites, streaming SSE optionnel)
- `POST /arena/evaluations`
- `GET /consents`, `PUT /consents/{purpose}`

Authentification par cookie HttpOnly/Secure/SameSite, CSRF si cookie; Google OIDC uniquement si paramétré. Contrôles de propriété systématiques, quotas et pagination. Une défaillance de moteur produit une erreur explicite, jamais une sortie factice.

## 8. Contrat d’adaptateur local

Interface logique : `list_models()`, `health()`, `generate(request)`, `stream(request)`. Ollama est le premier adaptateur; URL configurable (`http://127.0.0.1:11434` par défaut côté backend), modèle logique autorisé, paramètres bornés, timeouts, concurrence et limites définis par configuration. URL non-loopback refusée par défaut pour prévenir SSRF. Clé optionnelle par environnement, jamais en base ni dans les logs.

Erreurs normalisées : `provider_unavailable`, `model_not_installed`, `model_load_failed`, `timeout`, `rate_limited`, `invalid_request`, `provider_error`. Retry borné uniquement avant émission du flux et sur erreurs transitoires. Pas de retry après premier token. Rapporter le modèle/digest réellement utilisé, latence et tokens si disponibles; ne jamais journaliser contenu ou secret.

## 9. Rapport de benchmark obligatoire

Un modèle ne peut pas être marqué « prêt » sans rapport conservé, daté et reproductible comprenant :
- machine/OS/versions Ollama, modèle exact, tag et digest, quantification et taille du fichier;
- RAM avant/après et pic observé, temps de téléchargement séparé, temps de chargement;
- test réel réussi/échoué, prompt de test non sensible, contexte et paramètres;
- au minimum 30 générations mesurées : latence p50/p95, tokens/seconde, erreurs/timeouts;
- résultats qualité du benchmark privé versionné : français, raisonnement, maths vérifiables, rédaction, robustesse/sécurité; score et protocole;
- décision `retained`, `reserve` ou `rejected`, seuils appliqués et signataire.

Les seuils de performance sont à fixer après une première mesure sur la machine réelle. Les nombres ne sont pas à inventer ni déduire de la taille des poids. Modèle absent ou test non exécuté = `not tested`, jamais `ready`.

## 10. Tests de sécurité et d’acceptation

### Sécurité
Argon2id; rotation et détection de réutilisation de refresh token; CSRF/XSS/SQLi; rate limiting; contrôle d’accès par propriétaire/IDOR; limites d’entrée; SSRF; fuite de secrets; journalisation; révocation du consentement. Documents et exécution de code hors Phase 1. Le contenu fourni par l’utilisateur est toujours une donnée non fiable; détection de prompt injection en défense en profondeur uniquement.

### Tests Phase 1
1. Unitaires : profils/logique de fallback, validation, erreurs/timeouts.
2. Schéma : migration from-zero/upgrade/downgrade, 11 tables, contraintes, index et profils logiques initiaux.
3. Intégration : PostgreSQL local, Ollama absent, tag absent, load-test échoué, génération réelle du profil léger.
4. Sécurité : auth/session, isolation utilisateur, CSRF, rate-limit, SSRF, fuite de secret et absence de contenu en logs.
5. Benchmark réel selon la section 9.

Les critères initiaux exigeaient installation reproductible, migration/test, génération réelle, sécurité et rapport de mesures. Selon les preuves rapportées par l’utilisateur, la Phase 1 est déclarée terminée pour son périmètre fondation (upgrade, 11 tables, génération locale). Les résultats de downgrade/pytest et les mesures RAM/GPU/latence complètes ne sont pas inclus dans les éléments reçus; la déclaration de fin de Phase 1 n’autorise donc pas encore de revendication de performance ni de déploiement public.

## 11. Fichiers livrés et statut d’exécution

- `ARCHITECTURE-V8.md` — ce document, corrigé.
- `database/migrations/versions/0001_phase1_core.py` — schéma initial 11 tables et profils logiques par défaut.
- `alembic.ini`, `database/migrations/env.py` — exécution Alembic via `DATABASE_URL`.
- `.env.example` — variables de configuration sans secrets.
- `tests/test_migration_schema.py` — tests opt-in contre une base PostgreSQL dédiée `*_test`.
- `docs/windows-install-phase1.md` — commandes Windows préalables à l’implémentation.
- `docs/hardware-validation-report.md` — résultats de Phase 1 renseignés d’après les sorties rapportées; RAM/GPU/disque/latences restent à mesurer.

Dans cet espace de travail, j’ai mis à jour les livrables et effectué une vérification syntaxique Python; je n’ai pas relancé les tests sur le PC Windows ni sur PostgreSQL 18.6. L’utilisateur rapporte que la migration et une génération Ollama réelle ont réussi. Le contrôle postérieur à `path_separator = os` doit encore être exécuté sur le PC pour confirmer l’absence d’avertissement.

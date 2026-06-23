# Project Config — /tech-pm-ia

> Configuration project-specific consommée par `/tech-pm-ia` (Mode 0.c).
> Édite ce fichier pour adapter le comportement du skill à ton projet.
> Auto-chargé à chaque invocation de `/tech-pm-ia`.

## Project meta

| Clé | Valeur |
|---|---|
| **PROJECT_NAME** | jarvis |
| **ID_PREFIX** | JAR <!-- items numérotés JAR-001, JAR-002... --> |
| **LANGUE** | français |
| **GIT_REMOTE** | # (non configuré — repo pas encore créé) |

## Notion integration (optionnel)

> # (non configurée) — roadmap suivie en local dans REPRISE.md pour l'instant.
> Si une roadmap Notion est créée plus tard, renseigner ici NOTION_ROADMAP_URL + data source IDs (ne JAMAIS inventer les IDs).

| Clé | Valeur |
|---|---|
| **NOTION_ROADMAP_URL** | # (non configurée) |
| **NOTION_ROADMAP_DATA_SOURCE** | # (non configurée) |

## Roadmap phases

Liste ordonnée des phases du projet (utilisée par Mode 1 et Mode 4) :

- Phase 0 — Provisioning & infra serveur (Scaleway, Docker, DNS, TLS)
- Phase 1 — Moteur Hermes + Honcho (build on-box, mémoire self-hosted)
- Phase 2 — Canaux & UI (Slack, WhatsApp Baileys, hermes-webui)
- Phase 3 — Perso CEO & contexte (USER.md Michael Martin, skills)
- Phase 4 — Observabilité & runbooks (sync repo, recreate/rollback, monitoring)

## Roadmap schema (référence — si passage à Notion)

| Propriété | Type | Valeurs |
|---|---|---|
| Titre | Title | Texte libre |
| ID | Unique ID | JAR-xxx (auto) |
| Phase | Select | (cf. section Roadmap phases ci-dessus) |
| Priorité | Select | Critique, High, Medium, Low |
| Effort | Select | Small, Medium, Large |
| Statut | Status | Pas commencé, En cours, Terminé |
| Type | Select | Infra, Skill, Doc, Integration, Feature |
| Dépendances | Rich text | Noms des items prérequis |
| Description | Rich text | Texte libre |

## Conventions de commit

| Clé | Valeur |
|---|---|
| **COMMIT_PREFIXES** | feat, fix, refactor, docs, chore, style, test, perf, build, ci |
| **COMMIT_SCOPE_REQUIRED** | false |
| **CHANGELOG_FORMAT** | Keep a Changelog 1.1.0 |

## Notes

- Ce fichier est **lu** par `/tech-pm-ia` au Mode 0.c.
- Roadmap en local (REPRISE.md) tant que Notion n'est pas câblé → Modes Notion désactivés.
- Toute modification prend effet à la prochaine invocation du skill.

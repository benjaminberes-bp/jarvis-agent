# Changelog — jarvis

Toutes les évolutions notables de ce projet sont consignées ici.

Format inspiré de [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/).
Fork sans cycle de release → **jalons datés, pas de tags SemVer**.

## [Non publié]

### Added
- **2026-06-23** — Création du projet **Jarvis** (instance Hermes dédiée à l'usage personnel de Michael Martin, CEO Bienprêter). Dossier `claude-projects/jarvis/` + docs de suivi : `CLAUDE.md`, `DECISIONS.md`, `REPRISE.md`, `CHANGELOG.md`, `.claude/tech-pm-ia.config.md`.
- **2026-06-23** — Choix d'architecture initiaux actés (cf. `DECISIONS.md`) : nouvelle instance Scaleway dédiée (pas le box Alfred saturé), single-user, rebrand au build (sed Dockerfile), UI **hermes-webui** (nesquena, vérifiée purpose-built pour Nous Research Hermes), WhatsApp via bridge **Baileys** (risque ban assumé, numéro dédié jetable), Slack natif, Honcho self-hosted.
- **2026-06-23** — Estimation initiale du chantier : ~4,5–8 j-ingé + ~30–80 €/mo infra Scaleway. Roadmap locale en 5 phases dans `REPRISE.md` (Notion non câblé).

- **2026-06-23** — Décisions complémentaires actées : auth UI **native hermes-webui** (pas de magic-link) ; repo = **fork neuf de `nousresearch/hermes-agent`** (pas clone d'Alfred) ; onboarding **voie B** (interview par Jarvis → analyse transcript hors-serveur). Question set d'onboarding rédigé : `docs/onboarding-ceo.md`.
- **2026-06-23** — **Setup repo exécuté** : fork `nousresearch/hermes-agent` → `benjaminberes-bp/jarvis-agent` (origin + `upstream` pour re-merge), cloné dans `claude-projects/jarvis-agent/`. Docs de suivi greffés (aucun fichier upstream écrasé). Commit `41491ef88` sur branche `chore/scaffolding-suivi`, push origin, **PR #1** ouverte vers `main`. **Dir projet = désormais `claude-projects/jarvis-agent/`** (l'ancien `jarvis/` vide).

### À venir
- Porter d'Alfred la *technique* : sed rebrand Hermes→Jarvis + bake Honcho du `Dockerfile`, runbooks `deploy/` recreate/rollback, `scripts/sync-from-box.sh`.
- Phase 0 — Provisioning Scaleway (instance, Docker, DNS, TLS).

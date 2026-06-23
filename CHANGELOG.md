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

### À venir
- Phase 0 — Provisioning Scaleway (instance, Docker, DNS, TLS).
- Création du repo git + remote (fork vs clone à trancher).

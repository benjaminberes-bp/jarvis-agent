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
- **2026-06-23** — **Setup repo exécuté** : fork `nousresearch/hermes-agent` → `benjaminberes-bp/jarvis-agent` (origin + `upstream` pour re-merge), cloné dans `claude-projects/jarvis-agent/`. Docs de suivi greffés (aucun fichier upstream écrasé). **PR #1** mergée vers `main`. **Dir projet = désormais `claude-projects/jarvis-agent/`** (l'ancien `jarvis/` vide, supprimé).
- **2026-06-23** — **Instance Scaleway `jarvis-prod` provisionnée** (Phase 0) : STANDARD3-X4C-16G (4 vCPU dédiés, 16 Go), Ubuntu 24.04 LTS, Block Storage 100 Go, zone **nl-ams-1 (Amsterdam)** (PAR/MIL en rupture), IPv4 `51.15.106.239`, ~131 €/mo HT. Clé SSH dédiée **`jarvis_prod`** (ed25519 sans passphrase, alias `ssh jarvis-prod`).
- **2026-06-23** — **Accès SSH `jarvis_prod` rendu stable/reboot-proof** : clé placée dans **`/root/.ssh/instance_keys`** (ré-injectée par `scw-fetch-ssh-keys` à chaque boot). **Gotcha Scaleway documenté** (cf. `DECISIONS.md`) : `authorized_keys` est généré depuis le metadata d'instance (figé à la création — ajout de clé projet *après* ne se propage pas) + `instance_keys` ; **ne jamais éditer `authorized_keys` à la main** (wipé au fetch → a causé un lockout en session, récupéré via `alfred_par1`).
- **2026-06-23** — **Durcissement OS** (Phase 0) : `apt upgrade` (78 MAJ), **swapfile 4 Gi** + `vm.swappiness=10` (prudence ollama/Honcho sur RAM sans swap), **ufw actif** (22/80/443 ALLOW, deny incoming par défaut), **fail2ban actif** (jail sshd, bantime 1h / maxretry 5 / backend systemd). sshd déjà key-only (`passwordauthentication no`). unattended-upgrades actif.
- **2026-06-23** — **Docker installé** (Phase 0) : repo officiel Docker, **Docker CE 29.6.0** + **Compose v5.1.4** + buildx + containerd, service enabled, smoke `hello-world` OK. ⚠️ Noté : Docker bypass ufw (ports `-p` contournent les règles) → à gérer en Phase 2 (publier en `127.0.0.1:` derrière Caddy ou `ufw-docker`).

### Changed
- **2026-06-23** — **Dockerfile : bake Honcho** (branche `feat/port-alfred-technique`, PR #2). `--extra honcho` ajouté à `uv sync` (sinon perdu au recreate). **Rebrand au build Hermes→Jarvis ÉCARTÉ** (≠ Alfred) : Jarvis sert l'UI via hermes-webui (pas le dashboard baked) → sed `web_dist` inutile ; libellés `.py` = CLI non vus ; identité chat = `SOUL.md` custom (écrase `default_soul.py`). Dockerfile diverge d'upstream d'**1 ligne** → merges quasi-triviaux. CLIs/MCP marketing non portés. Build non testé (pas de serveur).

### À venir
- `scripts/sync-from-box.sh` + runbooks `deploy/` recreate/rollback : **différés** à Phase 0/4 (params serveur désormais connus — IP `51.15.106.239`).
- Phase 0 (reste) — durcissement étape B (activer ufw 22/80/443 + jail fail2ban sshd), install Docker, DNS + Caddy/TLS.

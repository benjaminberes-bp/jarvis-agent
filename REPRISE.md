# Prompt de reprise — jarvis

> Créé le 2026-06-23. Suivi : DECISIONS.md + CHANGELOG.md + CLAUDE.md (orientation Claude Code).

## Contexte rapide

**jarvis** = nouvelle instance Hermes dédiée à l'**usage personnel de Michael Martin (CEO Bienprêter)**. Distinct d'Alfred (agent marketing collectif). Single-user. **Phase 0 close + Phase 1 build on-box validé** : serveur Scaleway `jarvis-prod` live, image `jarvis:latest` buildée + bootée (s6, honcho baké). Reste Phase 1 = Honcho self-hosted, puis canaux (Phase 2) et onboarding CEO (Phase 3).

> ⚠️ **Dir projet = `claude-projects/jarvis-agent/`** (l'ancien `jarvis/` est vide). Toute session de travail se lance depuis `jarvis-agent/`. Remotes : `origin`=`benjaminberes-bp/jarvis-agent`, `upstream`=`nousresearch/hermes-agent` (re-merge).

Composants visés (estimation ~4,5–8 j-ingé) : serveur **Scaleway** + moteur **Hermes** (build on-box, rebrand au build) + **Honcho** self-hosted (mémoire) + **Slack** (natif) + **WhatsApp** (bridge Baileys natif) + UI **hermes-webui** (nesquena, purpose-built pour ce moteur).

## 🆕 Dernière session (2026-06-24) — HONCHO + SLACK + WEBUI LIVE (Phase 1 close, Phase 2 quasi close)

> ✅ Build on-box (item 4) · ✅ **Honcho self-hosted** (item 5, status OK) · ✅ **Slack natif** (item 6, smoke DM end-to-end OK) · ✅ **hermes-webui** (item 8) : two-container, partage `jarvis-data`, auth native, brandé Jarvis, `127.0.0.1:8787` (tunnel SSH). Conteneurs live : `jarvis` (agent, gateway run, réseaux honcho-net+hermes-net), stack `honcho-stack-*`, `hermes-webui`.
> ✅ **Tuning fait** : providers aux openrouter/nous coupés (→ anthropic/haiku, logs propres), **modèle défaut opus-4.6 conservé** (choix owner). **WhatsApp (item 7) EN PAUSE** (confirmer avec Michael). **Phase 3 = collaboratif strict** (rien en solo).
> Secrets : `jarvis-prod:/opt/data/.env` (agent) + `/opt/webui.env` (webui password), 600, jamais commit.

### 🐳 Build & boot (cette session)
- Repo cloné on-box : **`/opt/jarvis-agent`** (HTTPS, repo public, branche `feat/port-alfred-technique`, HEAD `759ae605`). Scripts s6 en LF (`.gitattributes` + checkout Linux) → pas de casse s6.
- `docker build --build-arg HERMES_GIT_SHA=$(git rev-parse HEAD) -t jarvis:latest .` → exit 0. Log : `/opt/jarvis-build.log`.
- Boot : `docker run -d --name jarvis --restart unless-stopped -v jarvis-data:/opt/data jarvis:latest sleep infinity`. Services s6 (`main-hermes`+`dashboard`) supervisés indépendamment du CMD.
- `memory.provider: ''` laissé vide (défaut local) → wiring Honcho = item 5. `SOUL.md` = défaut (persona Jarvis différé onboarding).

### Session précédente (2026-06-23) — PHASE 0 BOUCLÉE (provisioning + accès + durcissement + Docker)
> ✅ Instance live, accès SSH stable (reboot-proof via `instance_keys`), OS durci (ufw+fail2ban+swap+MAJ), Docker CE 29.6.0 installé. DNS/TLS différé Phase 2 (domaine = sous-domaine `bienpreter.com`).
> ⚠️ Incident résolu : injection clé manuelle dans `authorized_keys` wipée par `scw-fetch` (gotcha Scaleway) → lockout récupéré via `alfred_par1`, fix durable = `instance_keys`. Cf. DECISIONS + mémoire `scaleway-ssh-instance-keys`.


### 🖥️ Instance `jarvis-prod` CRÉÉE (Scaleway, payante, live)
- **IPv4 `51.15.106.239`** · IPv6 `2001:bc8:1640:79f8:dc00:ff:fe6b:ab8f` · DNS `a9ed5871-a979-4cc2-be88-1d04d00b4f90.pub.instances.scw.cloud`
- ID `a9ed5871-a979-4cc2-be88-1d04d00b4f90` · type **STANDARD3-X4C-16G** (4 vCPU dédiés, 16 Go, = Alfred) · zone **nl-ams-1 (Amsterdam)** ⚠️ (PAR/MIL en rupture de stock) · OS **Ubuntu 24.04 LTS** · disque **Block Storage 5K 100 Go** · IPv4+IPv6 publiques · ~**131 €/mo HT** (€0,1797/h, horaire → arrêtable).
- Projet Scaleway `bienpreter-ai` (org After Infinity), même projet qu'Alfred. ⚠️ **Données en NL** (UE/RGPD OK), pas en France.
- Reste Phase 0 : durcissement OS de base + install Docker + DNS/Caddy/TLS (item 2).

### 🔑 Clé SSH dédiée `jarvis_prod` — ✅ ACCÈS STABLE (reboot-proof)
- Locale (AKUMABen) : `~/.ssh/jarvis_prod` (ed25519, **sans passphrase**, fp `SHA256:gTUbtyjXMjf5f18ArryQH8ZxiDYOMl7/e5m7vql4soc`). Alias `~/.ssh/config` → **`ssh jarvis-prod`** marche sans passphrase.
- ✅ **Clé installée durablement dans `/root/.ssh/instance_keys`** sur l'instance → ré-injectée par `scw-fetch-ssh-keys` à chaque boot. Vérifié reboot-proof.
- ✅ Pubkey aussi aux **clés projet Scaleway** (`jarvis-prod`, ID `007493cd-e5db-471d-9056-80bc227bacf5`) — sans effet sur CETTE instance (metadata figé à la création) mais utile pour toute instance re-créée.
- ⚠️ **GOTCHA Scaleway** (cf. DECISIONS) : `authorized_keys` est **généré par `scw-fetch-ssh-keys`** depuis le metadata d'instance (figé création) + `/root/.ssh/instance_keys`. **Ne jamais éditer `authorized_keys` à la main** (wipé au prochain fetch — ça a causé un lockout en session). Nouvelle clé d'accès → l'ajouter à **`instance_keys`**. **Recovery de secours** = `alfred_par1` (interactif, dans le metadata figé → survit aux fetch).

### Session repo (même jour, antérieure) — setup repo + port Dockerfile
- ✅ **Fork** `nousresearch/hermes-agent` → `benjaminberes-bp/jarvis-agent` + clone dans `claude-projects/jarvis-agent/`. `upstream` câblé pour re-merge.
- ✅ **6 docs suivi greffés** (aucun fichier upstream écrasé — vérifié). **PR #1 mergée** vers `main`.
- ✅ **Dockerfile : bake Honcho** (branche `feat/port-alfred-technique`, **PR #2**) : `--extra honcho` ajouté. **Rebrand sed ÉCARTÉ** (≠ Alfred) — Jarvis UI = hermes-webui pas le dashboard baked, identité par `SOUL.md` custom. Dockerfile diverge d'upstream d'1 ligne. ⚠️ **Build non testé** (pas de serveur).
- ⏭️ Sync-from-box.sh + runbooks `deploy/` **différés** (besoin params serveur Scaleway).
- 📌 **À faire onboarding** : rédiger un `SOUL.md` Jarvis (persona CEO) → écrase `default_soul.py` « Hermes Agent ».

### Session précédente (2026-06-23) — bootstrap
- 📁 Docs de suivi rédigés + choix d'archi actés (cf. `DECISIONS.md`) : Scaleway dédié, single-user, rebrand au build, **UI=hermes-webui** (purpose-built Hermes, zéro glue), **WhatsApp=Baileys** (numéro dédié jetable obligatoire), Slack natif, Honcho self-hosted. Estimation ~4,5–8 j-ingé + ~30–80 €/mo.

## Prochaines actions (roadmap locale — Notion non câblé)

### Phase 0 — Provisioning & infra serveur
1. ✅ **Instance créée + accès SSH stable/reboot-proof** (via `instance_keys`) + ✅ **durcissement** : `apt upgrade` (78 MAJ), **swap 4Gi** (swappiness=10), **ufw actif** (22/80/443, deny incoming par défaut), **fail2ban actif** (jail sshd, bantime 1h/maxretry 5), sshd key-only + ✅ **Docker installé** (CE 29.6.0 + Compose v5.1.4, smoke `hello-world` OK, service enabled). ⚠️ **Gotcha à gérer Phase 2** : Docker bypass ufw (ports `-p` contournent ufw) → publier en `127.0.0.1:` derrière Caddy, ou `ufw-docker`.
2. ⏭️ **DNS + Caddy/TLS — DIFFÉRÉ à Phase 2** (requis seulement à l'expo publique de la webui). Domaine décidé : **sous-domaine `bienpreter.com`** (ex. `jarvis.bienpreter.com`) → A record vers `51.15.106.239` le moment venu. Accès d'ici là = SSH. ✅ **Phase 0 considérée CLOSE.**

### Phase 1 — Moteur Hermes + Honcho (dép. Phase 0)
3. ✅ **Fork + remote** (PR #1) + ✅ **Dockerfile : bake Honcho** (PR #2, `--extra honcho`). **Build validé on-box** (item 4) → PR #2 mergeable.
4. ✅ **Build image ON-BOX + premier boot** : `jarvis:latest` buildée (exit 0), boot s6 clean, smoke OK (honcho 2.0.1 importable, SHA baké, `config.yaml display:` plein). Volume `jarvis-data`.
5. ✅ **Honcho self-hosted DÉPLOYÉ + wired**. Kit porté (`docker/honcho/`, PR #3) + **stack live** sur jarvis-prod (ollama 768 + pgvector + redis + api healthy + deriver), clé Anthropic récupérée d'Alfred (pipe serveur→serveur). `config.yaml memory.provider: honcho`, `hermes honcho status` = OK. Clé LLM moteur jarvis stagée dans `/opt/data/.env` (réutilise clé Alfred) → effective au recreate (groupé avec Slack).

### Phase 2 — Canaux & UI (dép. Phase 1)
6. ✅ **Slack natif DÉPLOYÉ + validé** (workspace promup, app `@jarvis`, allowlist Benjamin+Michaël, Socket Mode). Smoke DM end-to-end OK (inbound → Honcho write → réponse anthropic). Tokens en `/opt/data/.env`.
7. **[High/Medium]** WhatsApp Baileys : Node + **numéro dédié** + QR pairing + volume session + allowlist. ⚠️ risque ban.
8. ✅ **hermes-webui DÉPLOYÉ** (two-container, `ghcr.io/nesquena/hermes-webui`). Partage `jarvis-data`, agent in-process, `hermes-net`, source via `hermes-agent-src`. Auth native, brandé Jarvis, `127.0.0.1:8787` (tunnel SSH). Runbook `docker/webui/README.md`. Chat UI à valider par owner via tunnel.

### Phase 3 — Perso CEO & contexte (dép. Phase 1+2)
9. **[High/Small]** 1ʳᵉ session = **interview d'onboarding** de Michael par Jarvis (`docs/onboarding-ceo.md`).
10. **[High/Medium]** Analyser le transcript hors-serveur (Claude Code) → `USER.md` Michael + liste priorisée skills/MCP/tools + garde-fous autonomie → install délibérée.

### Phase 4 — Observabilité & runbooks
11. **[Medium/Medium]** Script de sync serveur→repo (adapter `sync-from-box.sh` d'Alfred).
12. **[Medium/Small]** Runbooks recreate/rollback + smoke tests + vérif MCP.

## 🎯 Kickoff prochaine session — Phase 3 (collaboratif) ; WhatsApp en pause

**Pré-requis OK** : `ssh jarvis-prod` ; conteneurs live = `jarvis` (agent, `gateway run`, réseaux honcho-net+hermes-net), `honcho-stack-*`, `hermes-webui`. Honcho + Slack + webui opérationnels. ✅ **Tuning fait** (aux providers coupés ; opus conservé). Secrets `/opt/data/.env` + `/opt/webui.env`.

**⏸️ Item 7 — WhatsApp : EN PAUSE** — à **confirmer avec Michael** avant tout dev (risque ban + numéro dédié jetable = engagement). Ne PAS démarrer sans son accord.

**🔒 Phase 3 — onboarding/perso CEO : COLLABORATIF STRICT** — préparé **ensemble** avec l'owner. L'agent **ne fait rien en solo** : pas d'interview lancée, pas de `USER.md`/`SOUL.md` rédigé seul, pas d'install skills autonome. Voie B renforcée. Attendre déclenchement explicite owner.
  - ✅ **Étape 1-2 outillée** : `docs/phase3-import.md` = prompt d'export mémoire (Source A, Michael lance dans SON Claude) + récup connecteurs Desktop (Source B) → croisement → liste MCP priorisée gated. **En attente** : Michael colle son export + liste connecteurs.
  - ⏭️ Étape 3 = interview ciblée (`docs/onboarding-ceo.md`) sur les trous. Étape 4 = curation `USER.md`/`SOUL.md` + install gated.

**Reliquats optionnels (au besoin, non prioritaires)** :
- `SOUL.md` custom (persona Jarvis) — à faire **dans** Phase 3 (dépend de l'usage CEO).
- `/hermes sethome` Slack (delivery cron/cross-platform).
- API agent 8642 (`API_SERVER_ENABLED`+`API_SERVER_KEY` ; recreate) → pastille statut gateway webui.

**Recreate webui** : `bash /opt/webui-run.sh`. **Recreate jarvis** : CMD `gateway run` + `--env-file <vol>/.env` (chemin HÔTE) + réattacher `honcho-net` ET `hermes-net`. ⚠️ Après rebuild image → repeupler `hermes-agent-src` (cf. `docker/webui/README.md`). **`config.yaml` édité DIRECT** (backups `config.yaml.bak-*` dans le volume).

**Rappels durs** : `authorized_keys` jamais à la main (scw-fetch wipe → mémoire `scaleway-ssh-instance-keys`) ; Docker bypass ufw → `127.0.0.1:` ; `config.yaml` édité **direct** (pas `hermes config set`), `display:` jamais `null` ; secrets en `/opt/data/.env` (600), jamais commit ; commits PR-based, jamais sur `main`. **Recreate jarvis = CMD `gateway run` + `--env-file <vol>/.env` (chemin HÔTE) + réattacher `honcho-net`.**

## Décisions tranchées (2026-06-23)
- ✅ **Auth UI** : **native hermes-webui** (`HERMES_WEBUI_PASSWORD` ou WebAuthn/passkeys). Pas de magic-link (sur-ingénierie pour 1 user). Durcir si exposition publique.
- ✅ **Repo** : **fork neuf de `nousresearch/hermes-agent` → `benjaminberes-bp/jarvis-agent`** (PAS clone d'Alfred — marketing baggage). **Remote créé, PR #1 ouverte.** Porter d'Alfred la *technique* : sed rebrand + bake Honcho du Dockerfile, runbooks `deploy/`, `scripts/sync-from-box.sh`.
- ✅ **Onboarding** : **voie B (human-in-the-loop)** en v1 — Jarvis interviewe Michael → transcript analysé hors-serveur avec Claude Code → install délibérée. Auto-install autonome (voie A) écartée en v1, réévaluable après la 1ʳᵉ session. Question set prêt : `docs/onboarding-ceo.md`.

## Décisions à trancher (ouvertes)
- **Usage CEO précis** : en attente — sera cerné par l'**interview d'onboarding** (`docs/onboarding-ceo.md`) lors de la 1ʳᵉ session de Michael. Cadre les skills Phase 3.
- **Voie A vs B (définitif)** : confirmer après la 1ʳᵉ session si Jarvis pourra un jour auto-rechercher/installer (gaté) ou si on reste en analyse hors-serveur.

## Setup repo — ✅ FAIT (2026-06-23)

Fork + clone + greffe docs + commit + push + PR #1 exécutés. Détail dans « Dernière session » ci-dessus. **L'ancien dossier `jarvis/` (vide) reste à supprimer** depuis une session lancée hors de lui (cwd verrouillé) :
```bash
rmdir /c/Users/bbere/claude-projects/jarvis 2>/dev/null
```

## Reprise — checklist
1. **Travailler depuis `claude-projects/jarvis-agent/`** (PAS `jarvis/` vide, PAS Alfred).
2. Lire `DECISIONS.md` (haut) + `CLAUDE.md` + ce fichier.
3. Référence infra éprouvée = `DECISIONS.md` d'Alfred (`../hermes-agent/alfred-agent/`).
4. Workflow PR : branche par chantier, pas de commit direct `main`. `upstream` = re-merge Hermes.

## Prompt de reprise prêt à coller

```
Reprise projet jarvis (instance Hermes perso pour Michael Martin, CEO Bienprêter).
Lance depuis claude-projects/jarvis-agent/. Lire DECISIONS.md + CLAUDE.md + REPRISE.md.

ÉTAT : repo créé le 2026-06-23 (fork nousresearch/hermes-agent → benjaminberes-bp/jarvis-agent,
PR #1 docs de suivi). origin + upstream câblés. Pas encore de code applicatif ni de serveur.
Choix actés : Scaleway dédié, single-user, rebrand au build, UI=hermes-webui (purpose-built Hermes),
WhatsApp=Baileys (numéro dédié jetable), Slack natif, Honcho self-hosted, onboarding voie B.
Estimation ~4,5–8 j-ingé, ~30–80 €/mo.

ÉTAT : ✅ PHASE 0 CLOSE + ✅ PHASE 1 item 4 (build on-box). Instance jarvis-prod live (51.15.106.239),
accès `ssh jarvis-prod` stable/reboot-proof (clé dans /root/.ssh/instance_keys — NE PAS éditer
authorized_keys, scw-fetch le wipe), OS durci (ufw + fail2ban + swap 4Gi), Docker CE 29.6.0.
Image `jarvis:latest` buildée on-box (exit 0), boot s6 clean, smoke OK (honcho 2.0.1 importable,
SHA baké, config.yaml display: plein). Volume jarvis-data. Repo on-box /opt/jarvis-agent. PR #2 mergée.

PROCHAINE ÉTAPE : Phase 1 item 5 — Honcho self-hosted (docker compose pgvector+redis+ollama
embeddings+haiku) sur jarvis-prod, réf = stack Honcho d'Alfred (../hermes-agent/alfred-agent/ +
/opt/honcho-stack box Alfred). Puis wire config.yaml memory.provider=honcho (édit DIRECT, pas
hermes config set). Honcho en 127.0.0.1: (Docker bypass ufw). PUIS Phase 2 (Slack/WhatsApp/webui).
Décisions ouvertes : usage CEO précis (→ interview onboarding Phase 3), voie A vs B définitif.
Savoir-faire infra éprouvé = DECISIONS.md d'Alfred (../hermes-agent/alfred-agent/).
```

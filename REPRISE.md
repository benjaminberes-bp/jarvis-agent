# Prompt de reprise — jarvis

> Créé le 2026-06-23. Suivi : DECISIONS.md + CHANGELOG.md + CLAUDE.md (orientation Claude Code).

## Contexte rapide

**jarvis** = nouvelle instance Hermes dédiée à l'**usage personnel de Michael Martin (CEO Bienprêter)**. Distinct d'Alfred (agent marketing collectif). Single-user. **Repo créé** le 2026-06-23 (fork upstream) ; docs de suivi greffés ; pas encore de code applicatif ni de serveur — phase de planning.

> ⚠️ **Dir projet = `claude-projects/jarvis-agent/`** (l'ancien `jarvis/` est vide). Toute session de travail se lance depuis `jarvis-agent/`. Remotes : `origin`=`benjaminberes-bp/jarvis-agent`, `upstream`=`nousresearch/hermes-agent` (re-merge).

Composants visés (estimation ~4,5–8 j-ingé) : serveur **Scaleway** + moteur **Hermes** (build on-box, rebrand au build) + **Honcho** self-hosted (mémoire) + **Slack** (natif) + **WhatsApp** (bridge Baileys natif) + UI **hermes-webui** (nesquena, purpose-built pour ce moteur).

## 🆕 Dernière session (2026-06-23) — PROVISIONING Scaleway (Phase 0 entamée)

### 🖥️ Instance `jarvis-prod` CRÉÉE (Scaleway, payante, live)
- **IPv4 `51.15.106.239`** · IPv6 `2001:bc8:1640:79f8:dc00:ff:fe6b:ab8f` · DNS `a9ed5871-a979-4cc2-be88-1d04d00b4f90.pub.instances.scw.cloud`
- ID `a9ed5871-a979-4cc2-be88-1d04d00b4f90` · type **STANDARD3-X4C-16G** (4 vCPU dédiés, 16 Go, = Alfred) · zone **nl-ams-1 (Amsterdam)** ⚠️ (PAR/MIL en rupture de stock) · OS **Ubuntu 24.04 LTS** · disque **Block Storage 5K 100 Go** · IPv4+IPv6 publiques · ~**131 €/mo HT** (€0,1797/h, horaire → arrêtable).
- Projet Scaleway `bienpreter-ai` (org After Infinity), même projet qu'Alfred. ⚠️ **Données en NL** (UE/RGPD OK), pas en France.
- Reste Phase 0 : durcissement OS de base + install Docker + DNS/Caddy/TLS (item 2).

### 🔑 Clé SSH dédiée `jarvis_prod`
- Locale (AKUMABen) : `~/.ssh/jarvis_prod` (ed25519, **sans passphrase**, fp `SHA256:gTUbtyjXMjf5f18ArryQH8ZxiDYOMl7/e5m7vql4soc`, perms 600). Alias `~/.ssh/config` → **`ssh jarvis-prod`**.
- Pubkey ajoutée aux **clés projet Scaleway** (`jarvis-prod`, ID `6becf07d-a498-497d-8cf2-0783db7bffa7`) → injectée dans toute FUTURE instance.
- 🚨 **À FAIRE EN PREMIER — injecter la clé sur l'instance existante** : `jarvis-prod` a été créée AVANT l'ajout de la clé projet → son `authorized_keys` n'a que les 3 clés alfred, **pas** `jarvis_prod`. Étape **interactive unique** (passphrase, donc terminal interactif — pas le Bash non-interactif) :
  ```bash
  ssh -i ~/.ssh/alfred_par1 root@51.15.106.239 \
    "install -d -m700 ~/.ssh && echo '$(cat ~/.ssh/jarvis_prod.pub)' >> ~/.ssh/authorized_keys && sort -u ~/.ssh/authorized_keys -o ~/.ssh/authorized_keys"
  ```
  `alfred_par1` = clé trusted `alfred-par1-akumaben`. Après ça : `ssh jarvis-prod` marche sans passphrase. ⚠️ `chmod 600 ~/.ssh/alfred_par1` si « UNPROTECTED PRIVATE KEY ».
  > 🔎 Probe 2026-06-23 (BatchMode, depuis AKUMABen) : `jarvis_prod` → `Permission denied` (pas encore injectée, attendu) ; `alfred_par1` → `Permission denied` en BatchMode = **clé à passphrase**, donc injection **non automatisable** (terminal interactif requis pour saisir la passphrase). À exécuter par l'owner.

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
1. ✅ **Instance `jarvis-prod` créée** (STANDARD3-X4C-16G, 100 Go, Ubuntu 24.04, nl-ams-1, IPv4 `51.15.106.239`) + clé SSH dédiée `jarvis_prod`. 🚨 **Reste : injecter la clé sur l'instance** (cf. bloc « Clé SSH dédiée » ci-dessus) + durcissement OS + install Docker.
2. **[Critique/Small]** DNS + domaine (sous-domaine type `jarvis.…`) + Caddy/TLS.

### Phase 1 — Moteur Hermes + Honcho (dép. Phase 0)
3. ✅ **Fork + remote** (PR #1) + ✅ **Dockerfile : bake Honcho** (PR #2, `--extra honcho` ; rebrand sed écarté → identité par `SOUL.md`). **Build non testé** (pas de serveur) → à valider au 1er build on-box (item 4).
4. **[Critique/Medium]** Build image **SUR LE SERVEUR** (gotchas : CRLF/s6, prune disque) + premier boot + `config.yaml` (`display: {}`).
5. **[High/Large]** Honcho self-hosted (pgvector+redis+ollama embeddings + haiku) + wire `memory.provider`.

### Phase 2 — Canaux & UI (dép. Phase 1)
6. **[High/Small]** Slack : app + tokens + connecteur natif.
7. **[High/Medium]** WhatsApp Baileys : Node + **numéro dédié** + QR pairing + volume session + allowlist. ⚠️ risque ban.
8. **[High/Medium]** hermes-webui : clone + `bootstrap.py` + wire state.db/config (layout volume) + **auth native** (`HERMES_WEBUI_PASSWORD`/WebAuthn) + route Caddy.

### Phase 3 — Perso CEO & contexte (dép. Phase 1+2)
9. **[High/Small]** 1ʳᵉ session = **interview d'onboarding** de Michael par Jarvis (`docs/onboarding-ceo.md`).
10. **[High/Medium]** Analyser le transcript hors-serveur (Claude Code) → `USER.md` Michael + liste priorisée skills/MCP/tools + garde-fous autonomie → install délibérée.

### Phase 4 — Observabilité & runbooks
11. **[Medium/Medium]** Script de sync serveur→repo (adapter `sync-from-box.sh` d'Alfred).
12. **[Medium/Small]** Runbooks recreate/rollback + smoke tests + vérif MCP.

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

PROCHAINE ÉTAPE : Dockerfile porté (PR #2 : honcho + rebrand). RESTE = Phase 0 provisionner
Scaleway, PUIS build on-box (valide le Dockerfile non testé), PUIS adapter sync-from-box.sh +
runbooks deploy/ (différés — besoin des params serveur).
Décisions ouvertes : usage CEO précis (→ interview onboarding), voie A vs B définitif.
Savoir-faire infra éprouvé = DECISIONS.md d'Alfred (../hermes-agent/alfred-agent/).
```

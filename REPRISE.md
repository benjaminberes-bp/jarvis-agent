# Prompt de reprise — jarvis

> Créé le 2026-06-23. Suivi : DECISIONS.md + CHANGELOG.md + CLAUDE.md (orientation Claude Code).

## Contexte rapide

**jarvis** = nouvelle instance Hermes dédiée à l'**usage personnel de Michael Martin (CEO Bienprêter)**. Distinct d'Alfred (agent marketing collectif). Single-user. Projet **scaffoldé le 2026-06-23 depuis la session Alfred** ; pas encore de code ni de serveur — phase de planning.

Composants visés (estimation ~4,5–8 j-ingé) : serveur **Scaleway** + moteur **Hermes** (build on-box, rebrand au build) + **Honcho** self-hosted (mémoire) + **Slack** (natif) + **WhatsApp** (bridge Baileys natif) + UI **hermes-webui** (nesquena, purpose-built pour ce moteur).

## 🆕 Dernière session (2026-06-23) — bootstrap du projet

- 📁 Dossier `claude-projects/jarvis/` créé + 5 docs de suivi (CLAUDE, DECISIONS, REPRISE, CHANGELOG, `.claude/tech-pm-ia.config.md`).
- ✅ Choix d'archi actés (cf. `DECISIONS.md` 2026-06-23) : Scaleway dédié (pas le box Alfred saturé), single-user, rebrand au build, **UI=hermes-webui** (vérifié = UI pour Nous Research Hermes, lit state.db/config en direct, zéro glue), **WhatsApp=Baileys** (risque ban assumé → numéro dédié jetable obligatoire), Slack natif, Honcho self-hosted.
- 📊 Estimation : ~4,5–8 j-ingé + ~30–80 €/mo infra. Dépendances externes : compte Scaleway, numéro tel dédié WhatsApp, app Slack, DNS/domaine.

## Prochaines actions (roadmap locale — Notion non câblé)

### Phase 0 — Provisioning & infra serveur
1. **[Critique/Medium]** Créer l'instance Scaleway (≥16 Go RAM / ≥80 Go disque), OS + Docker, durcissement de base.
2. **[Critique/Small]** DNS + domaine (sous-domaine type `jarvis.…`) + Caddy/TLS.

### Phase 1 — Moteur Hermes + Honcho (dép. Phase 0)
3. **[Critique/Medium]** **Fork `nousresearch/hermes-agent` → `benjaminberes-bp/jarvis-agent`** + remote git. Adapter le `Dockerfile` (sed rebrand Hermes→Jarvis + bake Honcho, porté d'Alfred).
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
- ✅ **Repo** : **fork neuf de `nousresearch/hermes-agent` → `benjaminberes-bp/jarvis-agent`** (PAS clone d'Alfred — marketing baggage). Porter d'Alfred la *technique* : sed rebrand + bake Honcho du Dockerfile, runbooks `deploy/`, `scripts/sync-from-box.sh`. **Remote git à créer.**
- ✅ **Onboarding** : **voie B (human-in-the-loop)** en v1 — Jarvis interviewe Michael → transcript analysé hors-serveur avec Claude Code → install délibérée. Auto-install autonome (voie A) écartée en v1, réévaluable après la 1ʳᵉ session. Question set prêt : `docs/onboarding-ceo.md`.

## Décisions à trancher (ouvertes)
- **Usage CEO précis** : en attente — sera cerné par l'**interview d'onboarding** (`docs/onboarding-ceo.md`) lors de la 1ʳᵉ session de Michael. Cadre les skills Phase 3.
- **Voie A vs B (définitif)** : confirmer après la 1ʳᵉ session si Jarvis pourra un jour auto-rechercher/installer (gaté) ou si on reste en analyse hors-serveur.

## Setup repo — turnkey (à exécuter en session jarvis/, PAS depuis Alfred)

> ⚠️ État bâtard à éviter : ne pas créer le remote sans greffer les docs. Le clone du fork doit **contenir** les docs de suivi à sa racine (comme Alfred). Séquence atomique :

```bash
cd /c/Users/bbere/claude-projects
# 1. Fork + clone upstream (repo GitHub nommé jarvis-agent, parallèle à alfred-agent)
gh repo fork nousresearch/hermes-agent --fork-name jarvis-agent --clone
# → crée claude-projects/jarvis-agent (source Hermes + .git, remote upstream auto)

# 2. Greffer les docs de suivi déjà rédigés
mv jarvis/CLAUDE.md jarvis/DECISIONS.md jarvis/REPRISE.md jarvis/CHANGELOG.md jarvis-agent/
mkdir -p jarvis-agent/.claude jarvis-agent/docs
mv jarvis/.claude/tech-pm-ia.config.md jarvis-agent/.claude/
mv jarvis/docs/onboarding-ceo.md jarvis-agent/docs/
rmdir jarvis/docs jarvis/.claude jarvis 2>/dev/null

# 3. Commit scaffolding sur une branche (workflow PR — pas de commit direct main)
cd jarvis-agent
git checkout -b chore/scaffolding-suivi
git add CLAUDE.md DECISIONS.md REPRISE.md CHANGELOG.md .claude/tech-pm-ia.config.md docs/onboarding-ceo.md
git commit -m "docs: scaffolding suivi projet Jarvis (CLAUDE/DECISIONS/REPRISE/CHANGELOG/config/onboarding)"
```

- **Projet dir devient `claude-projects/jarvis-agent/`** (le bare `jarvis/` disparaît). Mettre à jour les chemins mentaux.
- GitHub repo = `benjaminberes-bp/jarvis-agent`, `upstream` = `nousresearch/hermes-agent` (re-merge).
- Ensuite : porter d'Alfred la technique (sed rebrand + bake Honcho du Dockerfile, runbooks `deploy/`, `scripts/sync-from-box.sh`).

## Reprise — checklist
1. **Travailler depuis `claude-projects/jarvis/`** (pas depuis Alfred — bootstrap fini). ⚠️ Après le setup repo ci-dessus, le dir devient `claude-projects/jarvis-agent/`.
2. Lire `DECISIONS.md` (haut) + `CLAUDE.md` + ce fichier.
3. Référence infra éprouvée = `DECISIONS.md` d'Alfred (`../hermes-agent/alfred-agent/`).

## Prompt de reprise prêt à coller

```
Reprise projet jarvis (instance Hermes perso pour Michael Martin, CEO Bienprêter).
Lance depuis claude-projects/jarvis/ (ou jarvis-agent/ une fois le fork greffé).
Lire DECISIONS.md + CLAUDE.md + REPRISE.md.

1ʳᵉ ÉTAPE = setup repo turnkey (section « Setup repo » de REPRISE) : fork
nousresearch/hermes-agent → benjaminberes-bp/jarvis-agent, greffer les docs, commit branche.

ÉTAT : projet scaffoldé le 2026-06-23 (docs de suivi only). Pas encore de code/serveur.
Choix actés : Scaleway dédié (pas le box Alfred saturé), single-user, rebrand au build,
UI=hermes-webui (nesquena, purpose-built Hermes), WhatsApp=Baileys (numéro dédié jetable),
Slack natif, Honcho self-hosted. Estimation ~4,5–8 j-ingé, ~30–80 €/mo.

PROCHAINE ÉTAPE : Phase 0 — provisionner l'instance Scaleway. Voir roadmap locale
dans REPRISE.md. Décisions ouvertes : auth UI (native vs magic-link), repo (fork vs clone),
usage CEO précis. Savoir-faire infra éprouvé = DECISIONS.md d'Alfred.
```

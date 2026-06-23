# Decisions Log — jarvis

> Journal des choix de développement structurants. Maintenu par `/tech-pm-ia`.
> Lire ce fichier en début de chaque conversation pour contexte.

## Format

## YYYY-MM-DD — [titre court]
**Contexte** : pourquoi la question s'est posée
**Décision** : ce qui a été choisi
**Alternatives écartées** : ce qui n'a pas été retenu et pourquoi
**Impact** : conséquences concrètes
**Statut** : actif / révisé / obsolète

---

## 2026-06-23 — Instance Scaleway `jarvis-prod` provisionnée (Phase 0)

**Contexte** : exécution du provisioning serveur (Phase 0, décision « Scaleway dédié » du même jour). Choix de type, zone, stockage à figer.

**Décision**
- Instance **`jarvis-prod`** créée (live, payante) : type **STANDARD3-X4C-16G** (4 vCPU dédiés, 16 Go — identique au box Alfred, dimensionnement validé pour ollama embeddings Honcho), **Block Storage 5K 100 Go**, **Ubuntu 24.04 LTS**, IPv4 `51.15.106.239` + IPv6. Projet Scaleway `bienpreter-ai` (org After Infinity), même projet qu'Alfred. Facturation horaire (~131 €/mo HT si 24/7, arrêtable).
- **Zone = `nl-ams-1` (Amsterdam)** car **PAR (Paris) et MIL en rupture de stock** sur ce type. ⚠️ **Données hébergées aux Pays-Bas** — UE/RGPD OK, mais **pas en France**. À réévaluer (migration PAR) si une contrainte de localisation FR apparaît pour un CEO en secteur régulé.
- **Clé SSH dédiée `jarvis_prod`** (ed25519, sans passphrase) ajoutée aux clés projet Scaleway → injectée dans toute instance future. Séparée des clés Alfred (isolation des accès).

**Alternatives écartées**
- Zones PAR/MIL : indisponibles à la création (rupture stock) — d'où le repli NL.
- Réutiliser une clé SSH Alfred : écarté pour isoler les accès Jarvis (clé dédiée révocable indépendamment).

**Impact** : Phase 0 entamée. **Blocage immédiat** : `jarvis-prod` créée AVANT l'ajout de la clé projet → son `authorized_keys` n'a que les clés Alfred, pas `jarvis_prod`. Injection = **étape interactive** (passphrase sur `alfred_par1`, key trusted) — non réalisable via tooling non-interactif (probe BatchMode confirmé : `Permission denied`). À faire par l'owner en terminal. Ensuite : durcissement OS + Docker + DNS/Caddy/TLS, puis Phase 1 (build on-box).

**Statut** : actif

---

## 2026-06-23 — Port technique Alfred : Dockerfile maintenant, sync/runbooks différés

**Contexte** : après création du fork (PR #1 mergée), exécuter le port de la *technique* d'Alfred (cf. décision « fork upstream » du même jour). Quoi porter, quand.

**Décision**
- **Dockerfile porté immédiatement** (branche `feat/port-alfred-technique`, PR #2) : **bake Honcho uniquement** = `--extra honcho` ajouté à la ligne `uv sync` (sinon perdu à chaque recreate — le venv vit dans l'image, pas le volume `/opt/data`). **PAS porté** : CLIs/MCP marketing d'Alfred (Higgsfield, Meta Ads, Meta social, Notion MCP), backdrop custom.
- **Rebrand au build Hermes→Jarvis : ÉCARTÉ** (≠ Alfred). Le rebrand d'Alfred (sed `*.py` + `web_dist`) se justifiait car **Alfred utilise le dashboard baked comme UI**. Jarvis sert l'UI via **hermes-webui** (`:8787`) → le sed `web_dist` viserait la mauvaise UI (mort). Les libellés `.py` restants = CLI/banner/docstring/service-desc que l'utilisateur (Slack/WhatsApp/webui) ne voit jamais. La seule string chat-facing — `default_soul.py` « You are Hermes Agent… » — n'est qu'un **DÉFAUT** seedé dans `$HERMES_HOME/SOUL.md` *s'il est absent* (`config.py:824`) → **écrasé par un `SOUL.md` custom Jarvis** (défini à l'onboarding). **Identité = SOUL.md + titre hermes-webui, pas sed.**
- **`scripts/sync-from-box.sh` + `deploy/` (runbooks recreate/rollback) = DIFFÉRÉS** à Phase 0/4. Raison : fortement paramétrés serveur (volume `alfred-data`, alias SSH `alfred-deploy`, nom conteneur) — ces valeurs Jarvis **n'existent pas encore** (Scaleway non provisionné). À adapter quand le serveur existe.

**Alternatives écartées**
- **Rebrand sed (l'approche Alfred)** : écarté — surface mauvaise UI (web_dist) + libellés CLI non vus + fragile (no-op silencieux si upstream change strings/re-minifie) + **+60 lignes de divergence Dockerfile** = conflits de merge récurrents pour un payoff nul. Identité gérée proprement par SOUL.md.
- Porter `deploy/` + sync wholesale maintenant : amène le contenu marketing + params serveur inexistants.
- Lean extras (drop `messaging`/`matrix`/`hindsight`) : écarté pour minimiser le diff vs upstream non-testable hors-serveur ; `messaging` inclut déjà slack+qrcode.

**Impact** : `Dockerfile` Jarvis diverge d'upstream sur **1 seule ligne** (`--extra honcho`) → `git merge upstream/main` quasi-trivial. À FAIRE en aval : rédiger un `SOUL.md` Jarvis (persona CEO) lors de l'onboarding voie B. Build **non testé** (pas de serveur) → validé au 1er build on-box (Phase 1 item 4).

**Statut** : actif

---

## 2026-06-23 — Auth UI native + repo (fork upstream) + approche onboarding (voie B)

**Contexte** : 3 décisions ouvertes du scaffolding tranchées/cadrées avec l'owner.

**Décisions**
- **Auth hermes-webui = native** (`HERMES_WEBUI_PASSWORD` env, ou WebAuthn/passkeys). **PAS** de front magic-link/Caddy comme Alfred. Cohérent single-user : pas besoin de l'allow-list multi-email ni du flux Brevo d'Alfred. ⚠️ Si exposition publique : durcir (passkeys + TLS + éventuel filtrage IP), sinon garder en localhost/derrière VPN.
- **Repo = fork neuf de `nousresearch/hermes-agent` → `benjaminberes-bp/jarvis-agent`.** PAS un clone d'Alfred. Raison : Alfred porte toute la machinerie marketing (clients X/TikTok, crons emprunteurs BP DB, profil graphiste, skills AMF, MCP Meta/Brevo/Metabase) = poids mort + surface secrets pour un assistant perso CEO single-user. Fork upstream = base saine + re-merge `upstream/main` propre. **Porter sélectivement d'Alfred** la *technique* (pas le contenu) : méthode sed rebrand + bake Honcho du Dockerfile, runbooks `deploy/` recreate/rollback, `scripts/sync-from-box.sh`.
- **Onboarding = voie B (human-in-the-loop) en v1.** Jarvis mène l'**interview d'onboarding** (questions à Michael, capture du besoin) ; le transcript est ensuite **analysé hors-serveur avec Claude Code** → décision + installation délibérée des skills/MCP/tools → revue. **L'auto-recherche + auto-install autonome par Jarvis (voie A) est écartée en v1** (risque sécu/supply-chain/runaway pour un CEO en secteur régulé) ; palier ultérieur gaté. Question set de démarrage rédigé dans `docs/onboarding-ceo.md`.

**Alternatives écartées**
- Front magic-link pour l'UI : sur-ingénierie pour 1 utilisateur.
- Clone d'Alfred : héritage marketing non pertinent + dette.
- Voie A (Jarvis auto-installe) dès v1 : trop risqué sans garde-fous.

**Statut** : actif (auth + repo + approche onboarding tranchés ; choix voie A vs B définitif réévaluable après la 1ʳᵉ session)

---

## 2026-06-23 — Création du projet Jarvis (instance Hermes perso CEO) + choix d'architecture initiaux

**Contexte** : Michael Martin (CEO Bienprêter) veut un assistant personnel dédié, distinct d'Alfred (agent marketing collectif). Estimation faite depuis la session Alfred (savoir-faire build-on-box, Honcho, connecteurs réutilisés). Projet baptisé **Jarvis**, scaffoldé dans `claude-projects/jarvis/`.

**Décisions**
- **Nouvelle instance from-scratch sur Scaleway** (serveur dédié) — PAS un 2e conteneur sur le box Alfred (`163.172.181.112`, déjà ~99 % de disque). Dimensionnement visé : ≥16 Go RAM (ollama embeddings Honcho) / ≥80 Go disque.
- **Single-user** : Jarvis sert Michael Martin uniquement. Pas de multi-utilisateur, pas de délégation orchestrateur→graphiste (spécifique Alfred).
- **Rebrand au BUILD** (convention héritée d'Alfred) : strings d'affichage Hermes→Jarvis via `sed` dans le `Dockerfile`, jamais dans la source → merges upstream sans conflit.
- **UI = `hermes-webui` (nesquena/hermes-webui)** en remplacement du dashboard SPA Hermes de base. ✅ Vérifié : c'est une UI **conçue pour ce moteur** (Nous Research Hermes Agent), lit `state.db` SQLite + config en direct, vanilla JS + serveur HTTP stdlib, tourne sur le même serveur (`:8787`), auth optionnelle (password env `HERMES_WEBUI_PASSWORD` ou WebAuthn/passkeys). **Aucun glue protocole nécessaire** → poste UI peu coûteux (~0,5–1,5 j). ⚠️ Lit la state.db/config en direct → aligner sur le layout volume du conteneur. Auth à arbitrer (native webui vs front Caddy/magic-link comme Alfred).
- **WhatsApp = bridge Baileys natif Hermes** (`gateway/platforms/whatsapp.py`). Choix owner assumé malgré le risque. ⚠️ **API non-officielle** (émule WhatsApp Web) → risque de ban + casse aux updates protocole WA. Garde-fous obligatoires : **numéro dédié jetable** (jamais un numéro corp), pas d'outbound de masse, allowlist stricte, session montée en volume persistant.
- **Slack = connecteur natif Hermes** (`gateway/platforms/slack.py`), déjà rodé sur Alfred.
- **Honcho self-hosted** (mémoire) : stack pgvector+redis+ollama embeddings + haiku text-gen, comme Alfred (`/opt/honcho-stack`).

**Alternatives écartées**
- 2e conteneur sur le box Alfred : rejeté (disque saturé, isolation prod, blast radius).
- UI opensource générique (Open WebUI / LibreChat) : écartée au profit de hermes-webui, purpose-built pour ce moteur (zéro glue protocole).
- WhatsApp Cloud API officielle Meta : écartée pour le MVP (pas le connecteur Hermes → dev custom +2–4 j). Reste l'option de repli si bans répétés.

**Estimation initiale** : ~4,5–8 j-ingé (provisioning 0,5–1 + Hermes build 0,5–1 + Honcho 1–2 + Slack 0,5 + WhatsApp 0,5–1 + hermes-webui 0,5–1,5 + glue/tests/runbook 1). Infra Scaleway ~30–80 €/mo. Dépendances externes : compte Scaleway, **numéro tel dédié** WhatsApp, app Slack, DNS/domaine.

**Impact** : projet `jarvis/` créé avec docs de suivi (CLAUDE, DECISIONS, REPRISE, CHANGELOG, tech-pm-ia.config). Repo git + remote pas encore créés. Roadmap en local (REPRISE) tant que Notion non câblé.

**Statut** : actif

---

## 2026-06-23 — Bootstrap depuis la session Alfred (pas depuis le dossier Jarvis vide)

**Contexte** : où lancer la création des docs de suivi — depuis le projet Alfred (contexte chargé) ou depuis le dossier Jarvis neuf ?

**Décision** : **scaffold depuis Alfred**, car tout le savoir dur (gotchas build-on-box, stack Honcho, connecteurs natifs, estimation, choix UI/WhatsApp) vivait dans le contexte de cette session. Un dossier vide repart de zéro. Modèle = **bootstrap ici → run depuis `claude-projects/jarvis/`** ensuite (son propre CLAUDE.md oriente l'agent).

**Impact** : les futures sessions de travail Jarvis se lancent **depuis `claude-projects/jarvis/`**.

**Statut** : actif

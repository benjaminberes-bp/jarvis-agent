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

## 2026-06-24 — Phase 2 item 8 : hermes-webui DÉPLOYÉ (two-container, auth native, brandé Jarvis)

**Contexte** : UI web pour Jarvis (`nesquena/hermes-webui`, purpose-built Hermes). Comment la brancher sur l'agent `jarvis` existant ?

**Décision / résultat** :
- **Modèle two-container** (le bon fit) : `jarvis` reste l'agent (gateway Slack) ; `hermes-webui` = conteneur séparé (`ghcr.io/nesquena/hermes-webui:latest`) qui **partage le volume `jarvis-data`** (même config/sessions/mémoire) et lance l'agent **in-process** pour le chat UI. Pas de fork dans le repo (image publique).
- **Pas de recreate jarvis imposé par la webui** : volume `hermes-agent-src` peuplé depuis l'image (`docker run --rm -v hermes-agent-src:/opt/hermes jarvis:latest true`) pour que la webui installe les deps agent ; réseau `hermes-net` créé + attaché à `jarvis` **à chaud** (`docker network connect`).
- **Auth native** `HERMES_WEBUI_PASSWORD` (décision auth tranchée) — pas de magic-link. Brandé **« Jarvis »** (`HERMES_WEBUI_BOT_NAME`) → page login affiche Jarvis. Secret dans `/opt/webui.env` (600), run reproductible via `/opt/webui-run.sh` (non commités). UID/GID=10000 (match `hermes`).
- **Accès loopback uniquement** (`127.0.0.1:8787`, Docker bypass ufw) → **tunnel SSH** (`ssh -N -L 8787:127.0.0.1:8787 jarvis-prod`). DNS/TLS public différé.
- **API agent 8642 ÉCARTÉE en v1** : `API_SERVER_ENABLED=true` exige `API_SERVER_KEY` (même loopback) + la webui ne l'utilise que pour une **pastille de statut** (chat = in-process, sans API). Pas worth un recreate + wiring de clé pour une pastille → testé puis retiré.
- **Validé** : `/health` ok, `/api/auth/status` → `auth_enabled:true, password_auth_enabled:true`, `/` → 302 (login), page brandée Jarvis, container healthy. Chat round-trip UI = à faire par l'owner via tunnel (browser).

**Alternatives écartées** : single-container webui (lance SON propre agent, doublon du gateway) ; webui dans le conteneur jarvis (deps séparées + divergence upstream, maintenance lourde).

**Statut** : actif — **item 8 CLOS** (UI opérationnelle, accès tunnel).

---

## 2026-06-23 — Phase 2 item 6 : Slack natif DÉPLOYÉ + validé end-to-end

**Contexte** : 1er canal. Owner a fourni les tokens Slack (app jarvis, workspace promup.slack.com / team T01AXTJLEN9) + tranché « réutiliser la clé Alfred » pour le moteur LLM.

**Décision / résultat** :
- **Tokens en env conteneur** (mécanisme Alfred : gateway lit `os.getenv`) via `--env-file /opt/data/.env` (600 hermes, jamais commit) : `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`, `SLACK_ALLOWED_USERS`, `ANTHROPIC_API_KEY`. ⚠️ `--env-file` lit côté **hôte** → chemin = mountpoint volume (`/var/lib/docker/volumes/jarvis-data/_data/.env`), pas `/opt/data`.
- **Allowlist stricte** : `SLACK_ALLOWED_USERS=U08JAMRR1T3,U01AU8P3BT8` (Benjamin BERES + Michaël MARTIN ; IDs résolus via Slack). Slack = Socket Mode (`connections:write`) → **aucun port inbound**.
- **Recreate conteneur `jarvis`** (UN seul, groupe Anthropic + Slack) calqué sur Alfred : `docker run -d --name jarvis --restart unless-stopped --env-file <vol>/.env -v jarvis-data:/opt/data jarvis:latest **gateway run**` puis **`docker network connect honcho-net jarvis`** (réseau perdu au recreate — réattacher). Le `sleep infinity` du smoke item 4 est abandonné → boot prod réel (CMD `gateway run`).
- **Plateforme auto-activée** depuis la présence de `SLACK_BOT_TOKEN` (`gateway/config.py _apply_env_overrides`) — pas de config d'activation manuelle. Log : `Authenticated as @jarvis`, `Socket Mode connected`, `✓ slack connected`.
- **Smoke end-to-end validé** : DM de Benjamin (allowlisté) → inbound reçu → session Honcho créée (write mémoire OK) → turn `provider=anthropic model=claude-opus-4-6` → réponse livrée sur Slack (« déploiement validé ✅ »). Valide en une passe : Socket Mode + allowlist + clé LLM + write Honcho + delivery.

**Points ouverts (tuning, non bloquant)** :
- **Persona = défaut « Hermes Agent »** (SOUL.md pas encore custom) → rebrand identité Jarvis à l'onboarding Phase 3.
- **Modèle défaut `claude-opus-4-6`** (cher) — à arbitrer (haiku/sonnet pour le quotidien ?).
- **Providers auxiliaires `openrouter`/`nous`** sans crédit → warnings dans les logs (fallback anthropic OK) ; à désactiver dans `config.yaml` pour nettoyer.
- **Home channel Slack** non défini (`/hermes sethome`) — optionnel (cron/cross-platform delivery).

**Statut** : actif — **item 6 CLOS** (Slack opérationnel).

---

## 2026-06-23 — Phase 1 item 5 : Honcho self-hosted DÉPLOYÉ + wired (mémoire opérationnelle)

**MAJ deploy (même jour)** : le kit (ci-dessous) a été **déployé de bout en bout** sur `jarvis-prod` avec aval owner.
- `/opt/honcho-stack` (clone honcho officiel) + artefacts montés. `docker network honcho-net` créé.
- Services up : `ollama` (modèle `nomic-embed-text` 274 Mo pull), `database` (pgvector, healthy), `redis` (healthy), `api` (healthy, `/health`=ok), `deriver` (boucle OK).
- **Clé Anthropic récupérée du serveur Alfred** (`alfred-auto:/opt/honcho-stack/.env`), transférée **en pipe serveur→serveur** (jamais en clair) dans `jarvis-prod:/opt/honcho-stack/.env`.
- **Embeddings 768** : `alembic upgrade head` puis `configure_embeddings.py --yes` (ALTER vector→768, HNSW recréés) AVANT up api (sinon dimension mismatch 1536).
- **Wire jarvis** : conteneur `jarvis` attaché à `honcho-net` (joint `honcho-api:8000` OK), skill `autonomous-ai-agents/honcho` installé, `honcho.json` (`baseUrl=http://honcho-api:8000`, `workspace=jarvis`, AUTH off), `config.yaml memory.provider: honcho` (édition DIRECTE ligne 422, backup pris — PAS `hermes config set`), reload `s6-svc -r main-hermes`.
- **Smoke** : `hermes honcho status` → **OK** (connection établie, workspace jarvis, "no peer data yet" = normal, 0 conversation).
- ⚠️ **Clé LLM du moteur jarvis lui-même** (≠ Honcho) : owner a tranché **réutiliser la clé Alfred** → stagée dans `jarvis-prod:/opt/data/.env` (`ANTHROPIC_API_KEY`, 600 hermes). **Effective seulement au recreate du conteneur** (Alfred lit les secrets en env conteneur via `os.getenv`, source = `--env-file /opt/data/.env`). Recreate différé pour le grouper avec le wire Slack (un seul recreate).

**Statut** : actif — **item 5 CLOS** (mémoire opérationnelle).

---

## 2026-06-23 — Phase 1 item 5 (a) : kit de déploiement Honcho porté d'Alfred

**Contexte** : item 5 = Honcho self-hosted. Alfred porte une recette **éprouvée en prod** (`docker/honcho/` : `config.toml` + `docker-compose.override.yml` + `honcho.env.example` + `README.md`) — absente d'upstream et de jarvis. « Porter la technique d'Alfred » (CLAUDE.md) = rapatrier ce kit.

**Décision** :
- **Kit porté dans `docker/honcho/`** (branche `feat/honcho-self-host`, PR #3) adapté Jarvis : `alfred-gw`→`jarvis`, `alfred-deploy`→`jarvis-prod`, workspace `bienpreter`→`jarvis`, chemins `/opt/jarvis-agent`. Stack = Honcho officiel + **Ollama embeddings local** (`nomic-embed-text`, 768 dims, aucune donnée ne sort) + text-gen **Anthropic `claude-haiku-4-5`** (faible coût). AUTH off (réseau docker interne `honcho-net`).
- **Ajustement Jarvis vs runbook Alfred** : l'étape runtime `uv pip install honcho-ai` du README Alfred est **supprimée** — le SDK est **déjà baké + validé** dans l'image Jarvis (honcho 2.0.1, smoke item 4). Alfred l'avait câblé avant son bake ; Jarvis part propre.
- **Activation provider = édition directe de `config.yaml`** (`memory.provider: honcho`), PAS `hermes config set` (lossy, règle CLAUDE.md) — diverge du README Alfred qui utilisait `config set`.
- **Deploy on-box NON exécuté cette session (gaté)** : nécessite (1) **clé Anthropic** (secret, owner) à coller dans `/opt/honcho-stack/.env`, (2) **go owner** sur la charge RAM (Ollama+PG+Redis sur 16 Go+swap), (3) le `docker compose up` + reconfigure embeddings 768 + wire = opération contiguë à mener avec la clé. Porter le kit (secret-free) maintenant ; déployer ensuite.

**Alternatives écartées** : déployer la stack autonome sans clé → laisserait un demi-état (api refuse de démarrer sans embeddings 768 / sans clé text-gen) confus. Réutiliser la clé Alfred → décision owner, pas auto.

**Impact** : kit prêt et reviewable. Reste à faire (gaté) : exécuter la séquence du `docker/honcho/README.md` sur `jarvis-prod` avec la clé. ⚠️ Caveat 768 (migration crée 1536 par défaut → `configure_embeddings.py --yes`).

**Statut** : actif

---

## 2026-06-23 — Phase 1 item 4 : build image ON-BOX validé + premier boot OK (PR #2 validée)

**Contexte** : la PR #2 (bake Honcho) avait été ouverte mais **jamais testée** (build impossible sous Windows — gotcha Docker Desktop/BuildKit/CRLF). Le serveur `jarvis-prod` étant désormais prêt (Phase 0 close), exécuter le 1er build natif on-box pour valider le Dockerfile.

**Décision / résultat** :
- Repo cloné sur la box via **HTTPS** (`https://github.com/benjaminberes-bp/jarvis-agent.git`, repo **public** → pas d'auth), branche `feat/port-alfred-technique`, dans **`/opt/jarvis-agent`**. CRLF non problématique : checkout Linux natif = LF (+ `.gitattributes` force `eol=lf` sur `*.sh`/`Dockerfile`). `file` confirme scripts s6 en LF.
- **`docker build` natif réussi** (exit 0) → image **`jarvis:latest`** (5,28 Go disque / 1,27 Go content). Build-arg `HERMES_GIT_SHA=$(git rev-parse HEAD)` passé → SHA baké.
- **Premier boot** (`docker run -d --restart unless-stopped -v jarvis-data:/opt/data jarvis:latest sleep infinity`) **clean sous s6-overlay** : stage2-hook exit 0, 71 skills bundlés, services `main-hermes` + `dashboard` démarrés.
- **Smoke tests verts** : `honcho 2.0.1` importable dans le venv (✅ **valide le bake** — objet de la PR #2 ; le SDK vit dans l'image immuable, survit au recreate) ; `hermes --version` → `Hermes Agent v0.17.0 (2026.6.19) · upstream 759ae605` (✅ build SHA baké lisible) ; `config.yaml` seedé (16 Ko) avec bloc **`display:` plein** (ni `null` ni `{}` — gotcha respecté) ; `SOUL.md` seedé (template défaut, persona vide).
- **`memory.provider: ''`** (vide) laissé tel quel → défaut local, **wiring Honcho = item 5** (stack pas encore déployée).
- **SOUL.md custom Jarvis NON rédigé maintenant** : le persona dépend de l'usage CEO (décision ouverte, cernée à l'onboarding voie B Phase 3) → ne pas inventer. Reste le défaut « Hermes Agent » jusqu'à l'onboarding.

**Impact** : Phase 1 item 4 **clos**. PR #2 validée par le build → **mergeable**. Volume `jarvis-data` créé (data persistante). Conteneur `jarvis` tourne en `sleep infinity` (smoke) — les services s6 supervisés tournent indépendamment du CMD ; le boot « prod » réel (sans override) viendra avec le wiring des canaux Phase 2. Prochain : **Honcho self-hosted** (item 5).

**Statut** : actif

---

## 2026-06-23 — Accès SSH `jarvis_prod` : la clé doit vivre dans `/root/.ssh/instance_keys` (gotcha Scaleway scw-fetch)

**Contexte** : après injection manuelle de la pubkey `jarvis_prod` dans `~/.ssh/authorized_keys`, l'accès a **sauté en pleine session** (durcissement étape A). Diagnostic : `Permission denied (publickey)` alors que perms OK et fail2ban clean.

**Cause** : sur les instances Scaleway, `authorized_keys` est **généré par `scw-fetch-ssh-keys`** (`/usr/sbin/scw-fetch-ssh-keys`), service **oneshot** déclenché au boot **et** par d'autres évènements (ex. `scw-net-reconfig.path`, postinst de paquets pendant `apt upgrade`). Le script **réécrit entièrement** `authorized_keys` depuis : (1) `scw-metadata --cached` (clés SSH **figées dans les métadonnées d'instance à la CRÉATION** — ajouter une clé au **projet** Scaleway *après* création **ne se propage PAS** à l'instance existante, ni au cache ni au metadata live), + (2) `/root/.ssh/instance_keys` (fichier **user-managed**, concaténé tel quel). Toute ligne ajoutée à la main dans `authorized_keys` est donc **effacée** au prochain fetch.

**Décision** : la clé `jarvis_prod` est ajoutée à **`/root/.ssh/instance_keys`** (échappatoire prévue par le script). Vérifié reboot-proof : après `scw-fetch`, jarvis ré-apparaît dans `authorized_keys` sans intervention. La clé a aussi été ajoutée aux **clés projet Scaleway** (ID `007493cd-e5db-471d-9056-80bc227bacf5`) — sans effet sur cette instance, mais utile pour toute instance **re-créée** ensuite.

**Alternatives écartées**
- **Injection manuelle dans `authorized_keys`** : éphémère (wipée au prochain fetch). C'est ce qui a causé le lockout.
- **Ajout aux clés projet seules** : ne se propage pas à une instance existante (metadata figé à la création) — vérifié : `scw-fetch` laisse jarvis à 0 même après ajout projet.
- **`systemctl mask scw-fetch-ssh-keys.service`** : gèlerait `authorized_keys` mais casse la gestion de clés native + non-standard ; `instance_keys` fait le job proprement.

**Impact** : accès `ssh jarvis-prod` stable et reboot-proof. **Recovery de secours** = `alfred_par1` (clé alfred, dans le metadata figé → toujours présente après fetch) + ré-ajout à `instance_keys`. **Règle** : toute nouvelle clé d'accès à cette instance → l'ajouter à `/root/.ssh/instance_keys`, jamais directement à `authorized_keys`.

**Statut** : actif

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

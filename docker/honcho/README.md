# Déploiement Honcho self-hosted (jarvis-prod)

Artefacts versionnés pour déployer Honcho à côté du conteneur `jarvis`. Mémoire AI-native,
text-gen sur Anthropic `claude-haiku-4-5` (faible coût), embeddings **Ollama local**
(`nomic-embed-text`, 768 dims) → aucune donnée utilisateur ne sort.

> Porté du fork sœur **Alfred** (`docker/honcho/`, recette éprouvée en prod). Voir le
> `DECISIONS.md` d'Alfred (`../hermes-agent/alfred-agent/`) pour le contexte d'origine.

## Fichiers

- `config.toml` — config Honcho (tout text-gen → anthropic/haiku, embeddings Ollama 768).
  Committable (pas de secret).
- `docker-compose.override.yml` — ajoute Ollama, monte `config.toml`, réseau `honcho-net`.
- `honcho.env.example` — secrets/env (clé Anthropic à coller sur le serveur).

## ⚠️ Pré-requis Jarvis (gates)

1. **Clé Anthropic** (secret) à coller dans `/opt/honcho-stack/.env` → owner.
2. **RAM** : la stack ajoute Ollama + Postgres + Redis. `jarvis-prod` = 16 Go + swap 4 Gi.
   Surveiller (`free -h`, `docker stats`) — Ollama embeddings est le poste le plus gourmand.
3. **Docker bypass ufw** : ne JAMAIS publier l'api Honcho en public. Tout reste sur le
   réseau docker interne (`honcho-net`) ; le `curl /health` se fait depuis `127.0.0.1`.

## Séquence

```bash
ssh jarvis-prod
# 1. Cloner Honcho (hors volume jarvis-data)
cd /opt && git clone https://github.com/plastic-labs/honcho.git honcho-stack
cd honcho-stack
cp docker-compose.yml.example docker-compose.yml

# 2. Déposer les artefacts versionnés (repo déjà cloné on-box : /opt/jarvis-agent)
cp /opt/jarvis-agent/docker/honcho/config.toml .
cp /opt/jarvis-agent/docker/honcho/docker-compose.override.yml .
cp /opt/jarvis-agent/docker/honcho/honcho.env.example .env   # puis ÉDITER (secret)

# 3. Réseau partagé + Ollama + DB/redis
docker network create honcho-net
docker compose up -d ollama database redis
docker compose exec ollama ollama pull nomic-embed-text   # ~270 Mo, 768 dims

# 4. SECRET : éditer /opt/honcho-stack/.env → LLM_ANTHROPIC_API_KEY=<clé Anthropic>

# 5. API + deriver
docker compose up -d --build api deriver
curl -s http://127.0.0.1:8000/health        # OK attendu
```

### ⚠️ Reconfigurer les embeddings en 768 (OBLIGATOIRE avec Ollama)

La migration crée le schéma pgvector en **1536** par défaut (avant que `config.toml`
ne s'applique) → l'api refuse de démarrer ("dimension mismatch"). DB neuve → reconfigurer :

```bash
docker compose run --rm --entrypoint "" api \
  /app/.venv/bin/python scripts/configure_embeddings.py --yes   # ALTER -> vector(768)
docker compose up -d api deriver       # api+deriver redémarrent OK
```

## Câblage côté Hermes/Jarvis

> **SDK honcho-ai : DÉJÀ BAKÉ** dans l'image Jarvis (`Dockerfile` → `uv sync … --extra honcho`,
> honcho-ai==2.0.1, validé au build du 2026-06-23 : `import honcho` OK dans le venv).
> → **Sauter** l'étape `uv pip install honcho-ai` du runbook Alfred (elle n'existe que parce
> qu'Alfred a câblé Honcho AVANT son bake). Sur Jarvis le SDK est présent nativement.

```bash
# 1. Skill honcho (non-interactif)
docker exec -u hermes -e HERMES_HOME=/opt/data jarvis /opt/hermes/.venv/bin/hermes \
  skills install official/autonomous-ai-agents/honcho --yes

# 2. honcho.json (self-host, AUTH off → pas d'apiKey)
docker exec -u hermes jarvis /opt/hermes/.venv/bin/python -c \
  'import json;open("/opt/data/honcho.json","w").write(json.dumps({"baseUrl":"http://honcho-api:8000","workspace":"jarvis"}))'

# 3. Activer le provider : éditer config.yaml DIRECTEMENT (PAS `hermes config set` — lossy,
#    cf. CLAUDE.md). Mettre `memory.provider: honcho` (actuellement '').
#    docker exec jarvis sed -i 's/^  provider:.*/  provider: honcho/' /opt/data/config.yaml
#    (vérifier le bloc memory: au préalable — ne pas casser l'indentation)

# 4. Brancher Jarvis sur le réseau Honcho
docker network connect honcho-net jarvis

# 5. Restart gateway pour charger le provider — voir CAVEAT ci-dessous.
```

> **CAVEAT restart** : sur Alfred un restart brut a déjà cassé le gateway Slack (fix
> `s6-svc -u`). Quand des canaux Jarvis seront actifs (Phase 2), recharger le service
> visé plutôt qu'un `docker restart` brutal :
> `docker exec jarvis /command/s6-svc -r /run/service/main-hermes`.

## Vérifs

- `docker compose ps` — api/database/redis (healthy) + deriver + ollama up.
- `curl -s http://127.0.0.1:8000/health` → `{"status":"ok"}`.
- `docker exec jarvis curl -s http://honcho-api:8000/health` → OK (réseau honcho-net).
- `docker compose logs deriver` — boucle OK, pas d'erreur clé.
- `docker exec -u hermes -e HERMES_HOME=/opt/data jarvis /opt/hermes/.venv/bin/hermes honcho status`
  → `Enabled: True`.

## Single-user (Jarvis)

Jarvis est single-user (Michael) → pas de propagation d'identité multi-peer (le lot B
d'Alfred ne s'applique pas). La mémoire fonctionne par session (`sessionStrategy`) ou par
peer unique = Michael. Workspace Honcho = `jarvis`.

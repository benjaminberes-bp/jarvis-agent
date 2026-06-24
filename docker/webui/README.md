# Déploiement hermes-webui (jarvis-prod)

UI web (nesquena/hermes-webui) pour Jarvis. Modèle **two-container** : le conteneur
`jarvis` reste l'agent (gateway Slack), `hermes-webui` est un conteneur séparé qui
**partage le volume `jarvis-data`** (même config / sessions / mémoire) et lance l'agent
**in-process** pour le chat UI. Vanilla JS + Python stdlib, aucun build.

> Image publique `ghcr.io/nesquena/hermes-webui:latest` (pas de fork dans ce repo).
> Artefacts de déploiement serveur (non commités) : `/opt/webui-run.sh` + `/opt/webui.env` (600).

## Architecture retenue

- **`jarvis`** (agent) : `gateway run`, volume `jarvis-data:/opt/data`, réseaux `honcho-net` + `hermes-net`.
- **`hermes-webui`** : partage `jarvis-data` monté en `/home/hermeswebui/.hermes` (même home, chemin différent — supporté upstream), lit la **source agent** via le volume `hermes-agent-src` (deps installées au boot par `uv pip install`), rejoint l'agent sur `hermes-net`.
- **Auth native** : `HERMES_WEBUI_PASSWORD` (pas de magic-link). Brandé **Jarvis** (`HERMES_WEBUI_BOT_NAME`).
- **Accès** : `127.0.0.1:8787` **uniquement** (Docker bypass ufw → loopback). Pas de DNS/TLS public (différé). Accès via **tunnel SSH**.

## Pré-requis

- Conteneur `jarvis` tournant (volume `jarvis-data`), réseau `hermes-net` créé et attaché à `jarvis`.
- Volume `hermes-agent-src` peuplé depuis l'image agent (source `/opt/hermes`).

## Séquence (première fois)

```bash
ssh jarvis-prod

# 1. Réseau partagé + attacher l'agent (à chaud, pas de recreate)
docker network create hermes-net 2>/dev/null || true
docker network connect hermes-net jarvis 2>/dev/null || true

# 2. Peupler le volume source agent depuis l'image jarvis (sans recreate)
docker volume create hermes-agent-src
docker run --rm -v hermes-agent-src:/opt/hermes jarvis:latest /bin/true   # Docker copie /opt/hermes -> volume

# 3. Image webui
docker pull ghcr.io/nesquena/hermes-webui:latest

# 4. Secret + run (cf. /opt/webui.env + /opt/webui-run.sh)
#    /opt/webui.env (600) : HERMES_WEBUI_PASSWORD=<mot de passe>
bash /opt/webui-run.sh

# 5. Vérifs
curl -s http://127.0.0.1:8787/health            # {"status":"ok",...}
curl -s http://127.0.0.1:8787/api/auth/status   # auth_enabled:true, password_auth_enabled:true
```

## Accès owner (tunnel SSH)

```bash
ssh -N -L 8787:127.0.0.1:8787 jarvis-prod
# puis ouvrir http://localhost:8787  → login (mot de passe HERMES_WEBUI_PASSWORD)
```

## Recreate

```bash
bash /opt/webui-run.sh        # rm -f + run, lit le password depuis /opt/webui.env
```

## ⚠️ Gotchas

- **UID/GID = 10000** (`WANTED_UID`/`WANTED_GID`) pour matcher l'utilisateur `hermes` de l'agent → lecture/écriture du volume partagé `jarvis-data`.
- **Mise à jour de l'image agent** : le volume `hermes-agent-src` est figé à sa création. Après un rebuild de `jarvis:latest`, le repeupler :
  ```bash
  docker rm -f hermes-webui
  docker volume rm hermes-agent-src
  docker volume create hermes-agent-src
  docker run --rm -v hermes-agent-src:/opt/hermes jarvis:latest /bin/true
  bash /opt/webui-run.sh
  ```
- **API agent 8642 NON activée** : `API_SERVER_ENABLED=true` exige `API_SERVER_KEY` (même en loopback) et la webui ne s'en sert que pour une **pastille de statut gateway** (le chat tourne in-process, sans l'API). Non câblée en v1 — `HERMES_API_URL` pointe sur `http://jarvis:8642` mais la pastille reste dégradée tant que l'API n'est pas activée. À activer plus tard si besoin (clé partagée webui↔agent).
- **Deux process agent** (gateway `jarvis` + in-process webui) partagent le même `HERMES_HOME` — modèle multi-process Hermes normal (comme CLI + gateway).
- Secrets (`HERMES_WEBUI_PASSWORD`) dans `/opt/webui.env` (600), **jamais commités**.

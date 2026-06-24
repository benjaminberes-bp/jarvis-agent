# Phase 3 — Import du contexte existant (récup mémoire LLM + connecteurs)

> Outillage de **récupération** pour amorcer la perso de Jarvis (Michael Martin, CEO).
> Étape 1-2 du flux voie B. Le **question-set d'interview** (étape 3) vit séparément dans
> `onboarding-ceo.md`. **Phase 3 = collaboratif strict** : aucune ingestion ni install en solo.

## Pourquoi

Michael a déjà un assistant (Claude Desktop) avec un store mémoire + des connecteurs MCP.
On récupère cette **matière brute** pour accélérer la rédaction de `USER.md` / `SOUL.md` et
la sélection des skills/MCP de Jarvis — sans repartir de zéro. L'export **ne remplace pas**
l'interview ciblée : il la nourrit.

## Source A — Export mémoire (déclaratif, via prompt)

Michael lance ce prompt **dans SON compte Claude** (ce sont ses mémoires, pas les nôtres) :

```
Export everything you have stored about me in your memory / personalization store
(NOT recalled from past conversation history — only what is actually saved there).
Preserve my words verbatim, in their original language, especially for instructions
and preferences.

CRITICAL RULES:
- Include ONLY genuinely stored entries. Do NOT infer, guess, paraphrase, or fill gaps.
- If a category has no stored entries, write "(none stored)".
- Tag every line with its source: (stored) for a real memory entry, (inferred) if
  you are not certain it is actually stored.

## Categories (output in this exact order):

1. Instructions — Rules I've explicitly asked you to follow going forward: tone,
   format, style, "always do X", "never do Y", and corrections to your behavior.
   Stored memories only, not conversation-derived.

2. Identity — Name, age, location, education, family, relationships, languages,
   personal interests.

3. Career — Current and past roles, companies, general skill areas.

4. Projects — Projects I meaningfully built or committed to. ONE entry per project.
   Include what it does, current status, key decisions. Project name first.

5. Preferences — Opinions, tastes, working-style preferences that apply broadly.

6. Tools & Integrations — External tools, services, MCP connectors, or integrations
   I have mentioned using, asked you to connect to, or rely on for work — from stored
   memory only. Verbatim names. Do NOT list tools you merely have access to; only what
   I actually use.

## Format:
- One section header per category.
- One entry per line, sorted oldest date first:
  [YYYY-MM-DD] - content - (stored|inferred)
  Use [unknown] if no date is known.

## Output:
- Wrap the entire export in a single code block.
- After the code block, state whether this is the complete set or if more remain
  (I will reply "continue" to get the rest).
```

**Garde-fous intégrés** : cible le store mémoire (pas le rappel conversationnel, non fiable) ;
interdit l'inférence/comblement de trous ; tag `(stored|inferred)` pour séparer le sûr du
supposé ; verbatim en langue d'origine.

## Source B — Connecteurs réels (vérité terrain, hors prompt)

Un LLM **ne peut pas introspecter** quels MCP/tools ont *tourné* dans ses conversations →
la liste réelle se prend directement dans Claude Desktop (zéro hallucination) :

- **UI** : Réglages → **Connecteurs** → copier les **noms** des MCP actifs.
- **Fichier config** (exhaustif) :
  - Windows : `%APPDATA%\Claude\claude_desktop_config.json`
  - macOS : `~/Library/Application Support/Claude/claude_desktop_config.json`

⚠️ **Sécurité** : ne copier que les **noms** sous `mcpServers` — **retirer tout
token/clé/secret** des blocs `env` avant de partager. On veut la liste, pas les creds.

## Croisement → liste MCP priorisée (gated)

1. **Source A cat. 6** (déclaratif : ce que Michael dit utiliser) **+ Source B** (réel : ce qui est branché) → recoupement.
2. **Tri pertinence usage perso CEO** : on écarte le bruit marketing/corp (Jarvis ≠ Alfred).
3. **Garde-fous sécurité** : Jarvis = single-user, secteur régulé → chaque MCP = surface d'accès/secret. Install **délibérée**, jamais auto (voie B actée, cf. `DECISIONS.md`).
4. Sortie = liste skills/MCP priorisée, validée avec l'owner avant toute install.

## Flux complet Phase 3 (rappel)

1. **Récup** : Michael lance A + fournit B → colle ici.
2. **Tri ensemble** : ce qui va dans `USER.md` (identité/contexte CEO) vs `SOUL.md` (persona/ton de Jarvis) vs préférences de travail vs liste MCP.
3. **Interview ciblée** (`onboarding-ceo.md`) sur les trous que l'export ne couvre pas : canaux, tâches récurrentes, niveau d'autonomie, accès (mail/agenda/CRM ?), garde-fous régulé.
4. **Curation délibérée** → `USER.md` + `SOUL.md` + install MCP/skills gated.

> Rien de tout ceci n'est exécuté en solo par l'agent — chaque étape se fait avec l'owner.

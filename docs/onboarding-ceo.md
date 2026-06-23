# Onboarding CEO — première session de Michael Martin avec Jarvis

> But : cerner les besoins, le contexte et les préférences de Michael pour décider quels skills / MCP / tools installer. Rédigé le 2026-06-23. Approche **voie B** (human-in-the-loop) : Jarvis pose les questions → le transcript est analysé **hors-serveur avec Claude Code** → installation délibérée + revue. Cf. `DECISIONS.md` 2026-06-23.

## Comment ça marche (v1)

1. Jarvis mène l'interview ci-dessous lors de la 1ʳᵉ session (Slack ou WhatsApp).
2. Le transcript est récupéré hors-serveur.
3. Claude Code analyse → propose une liste de skills/MCP/tools + un `USER.md` Michael.
4. Revue humaine → installation délibérée sur le serveur (pas d'auto-install par Jarvis en v1).

⚠️ Ne PAS demander à Jarvis d'installer lui-même des outils en v1 (voie A écartée — sécu/supply-chain/runaway). Réévaluable après cette session.

## Posture de Jarvis pendant l'interview

- Conversationnel, une question à la fois, rebondir sur les réponses (pas un questionnaire récité).
- Reformuler pour confirmer la compréhension.
- Ne rien promettre qu'on n'a pas encore installé.
- Clôturer par un récap de ce qu'il a compris.

## Série de questions (starter set)

### A. Rôle & quotidien
1. Décris une journée type : qu'est-ce qui mange le plus ton temps ?
2. Quelles 3 tâches récurrentes aimerais-tu déléguer en premier ?
3. Sur quoi perds-tu le plus de temps que tu juges « sous ton niveau » ?

### B. Outils & écosystème
4. Quels outils utilises-tu chaque jour ? (email, agenda, Slack, Notion/Docs, CRM, BI, banque…)
5. Lesquels veux-tu que Jarvis puisse lire ? écrire/agir dedans ?
6. Y a-t-il des données sensibles/confidentielles à exclure totalement ?

### C. Ce que Jarvis doit faire
7. Briefings : veux-tu un point quotidien/hebdo ? Sur quoi (agenda, mails clés, chiffres, veille) ? À quelle heure ?
8. Rédaction : mails, notes, posts, synthèses de réunions — lesquels ?
9. Recherche/veille : sujets, concurrents, marché, réglementaire à suivre ?
10. Données : questions chiffrées récurrentes (KPIs, collecte, perf) à interroger en langage naturel ?
11. Rappels/relances : Jarvis doit-il te pousser des rappels proactifs ?

### D. Canaux & disponibilité
12. Canal principal : Slack ou WhatsApp ? (les deux sont prévus)
13. Quand veux-tu être sollicité / jamais dérangé ?
14. Réponses : courtes et directes, ou détaillées ?

### E. Style & préférences
15. Ton : direct/familier ou formel ? Tutoiement ?
16. Langue de travail (français par défaut).
17. Un assistant que tu as aimé/détesté ? Pourquoi ?

### F. Limites & confiance
18. Qu'est-ce que Jarvis ne doit JAMAIS faire sans ta validation ? (envoyer un mail, poster, dépenser…)
19. Niveau d'autonomie souhaité au départ : proposer puis attendre ton OK, ou agir et te rendre compte ?
20. Comment juges-tu que Jarvis t'est utile dans 1 mois ? (critère de succès concret)

## Sortie attendue de l'analyse (hors-serveur)

- `memories/users/michael.martin@bienpreter.com/USER.md` : rôle, prefs (ton, langue, canal, horaires), limites.
- Liste priorisée de skills/MCP/tools à installer (avec justification + niveau d'accès lecture/écriture).
- Garde-fous d'autonomie (ce qui exige validation).
- Backlog `JAR-xxx` des intégrations à câbler.

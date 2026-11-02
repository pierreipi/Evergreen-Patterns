# Analyse du repo Evergreen-Patterns

*Analyse réalisée le 2 novembre 2026 sur un fork de [michelin/Evergreen-Patterns](https://github.com/michelin/Evergreen-Patterns).*

## 1. Faire tourner le site

Le repo est un site **Hugo** avec le thème Ananke (sous-module git). En l'état, il ne compilait plus avec un Hugo récent (v0.165).

| Problème | Correction |
|---|---|
| Thème Ananke figé en 2023, il utilise `resources.ToCSS`, supprimé de Hugo | Sous-module mis à jour vers Ananke v2.19.1 |
| `hugo.toml` sans `theme`, `languageCode` déprécié, `baseURL` d'exemple | `theme = 'ananke'`, `locale`, `baseURL` local |
| Pas de page d'accueil, de pages de section ni de menu | `content/_index.md`, un `_index.md` par famille, menu dans `hugo.toml` |
| 24 pages pointent vers des illustrations jamais générées | Liens retirés (seules 15 illustrations existent) |

Deux versions du site sont disponibles :

```bash
git submodule update --init --recursive
hugo server -D                            # contenu 2024 (généré par IA), dossier content/
hugo server -D --config hugo-2022.toml    # contenu 2022 (écrit par l'équipe), dossier old-content/
```

Le site est ensuite accessible sur http://localhost:1313.

## 2. Historique du projet

| Année | Commits | Qui |
|---|---|---|
| 2022 | 469 | ~7 personnes (une partie des commits vient de la CI « Updating JSON file ») |
| 2023 | 2 | — |
| 2024 | 73 | quasiment une seule personne, de février à mai |

- **2022 :** une vraie équipe écrit le contenu à la main.
- **2023 :** le projet est déjà à l'arrêt, avant toute utilisation d'IA.
- **2024 :** une seule personne relance le projet en régénérant tout avec des LLM. Le texte d'origine est déplacé dans `old-content/`. Plus rien ne se passe après mai 2024.

Les autres branches (`addPattern`, `llm`, `nathan`) datent de 2022-2023 et sont plus anciennes que `main`.

## 3. Le contenu actuel (`content/`) est généré par IA

- **`generate-content.py` :** il produit les pages avec llama3 (Ollama) ou GPT-4-turbo, à partir de prompts Jinja et de `patterns.yaml`.
- **Champ `model:` :** 34 pages l'ont encore dans leurs métadonnées.
- **`mitigation-claude-3.txt` :** c'est une sortie de LLM.
- **Illustrations :** elles sont générées par Stable Diffusion ou ComfyUI (`stablediff.py`, `comfy-workflow.json.j2`).
- **Script cassé :** `generate-content.py` ne s'exécute même plus (erreur de syntaxe ligne 46 : `def generate_pattern(prompt, template_name, action model_name):`).

**Interprétation (hypothèse) :** l'IA n'a pas tué le projet, puisqu'il était déjà à l'arrêt en 2023. Elle a servi à faire vivre à moindre coût un projet sans équipe. Sans accès à la doc interne, les LLM n'ont produit que des généralités : le résultat a l'air propre mais n'a plus de substance. C'est un schéma classique pour l'open source d'entreprise : on publie le projet pendant un projet ou un stage, puis personne n'a de temps prévu pour le maintenir.

## 4. Le contenu d'origine (`old-content/`, 2022)

58 pages, avec un anglais un peu francisé typique d'une doc interne. Le contenu est nettement meilleur que la version générée.

### Ce qui vient des livres (~60 %)

- **La famille Architecting et la moitié des anti-patterns** reprennent *Release It!* de Michael Nygard, souvent avec les mêmes noms : Integration Points, Cascading Failures, Users, SLA Inversion, Unbalanced Capacities, Steady State, Fail Fast, Handshaking, Bulkheads, Timeouts, Decoupling Middleware. Le livre n'est jamais cité.
- **SLO, error budget, golden signals et toil** viennent du livre SRE de Google.
- **Immutability et Snowflake** viennent de Martin Fowler, et **Andon** du système de production Toyota.

Le comble, c'est que l'IA de 2024 a régénéré la partie qui était déjà générique, et a laissé tomber celle qui ne l'était pas.

### Ce qui reste difficile à trouver ailleurs (~25 %)

Ce sont les passages issus du terrain. Ils viennent surtout du monde de l'informatique d'entreprise (traitements batch, ETL, MQ, WebSphere), très peu présent en ligne face au contenu « startup cloud-native ».

- **`decoupling-middleware` (section « At Michelin ») :** MQ a été mal utilisé pour rendre asynchrones des échanges synchrones par nature. Ça ajoute de la complexité d'exploitation, rend l'équipe dépendante de l'infra pour gérer les messages en erreur, et lui fait perdre son autonomie.
- **`accounting-for-internal-failures` :**
  - interrompre brutalement chaque traitement long pendant les tests d'exploitabilité ;
  - fixer un objectif de temps de rétablissement pour chaque batch critique, et vérifier qu'on l'atteint avec les seuls logs ;
  - mettre les fichiers d'entrée en lieu sûr avant de les traiter ;
  - pouvoir relancer la partie restante sans toucher aux fichiers.
- **`overkill-response` :** tuer le processus plutôt que redémarrer le serveur, activer le mode debug sur un seul job ETL et pas sur tout le serveur, ne pas relancer tout le plan de batch.
- **`rushing-to-solution` et `collecting-evidence` :** faire un dump mémoire avant de redémarrer. Il y a aussi l'astuce du flag d'arrêt propre : au démarrage, si le flag montre que l'arrêt précédent ne s'est pas bien passé, l'application sauvegarde automatiquement ses derniers logs et transactions.
- **`everything-works-all-the-time` :** un plan de batch entier reste bloqué parce qu'un seul fichier n'est pas arrivé, d'où l'idée de découper le plan.
- **`error-categorization` :** le niveau de log y est vu comme un contrat entre ceux qui développent et ceux qui exploitent (« FATAL = quelqu'un est réveillé la nuit »).
- **`using-timeouts` :**
  - réfléchir à la chaîne complète des timeouts, de bout en bout ;
  - versionner les valeurs dans git pour les relier à l'état de santé de l'application ;
  - ne pas remplacer Hystrix juste parce qu'il n'est plus maintenu.
- **`error-budget` :** une coupure planifiée compte autant qu'une coupure imprévue, et on mesure sur 28 jours parce que ça contient toujours 4 week-ends.

### Ce qui est vide (~15 %)

- **Pages squelettiques :** `stateless`, `cascading-failures`, `unmastered-systems`, `continuous-integration`…
- **Sections « REAL MICHELIN WORLD: Ex: » :** elles sont vides sur 5 pages. La partie la plus précieuse, le contexte interne, n'a jamais été publiée.

### Ce qui est obsolète

- **Outils :** Hystrix est arrêté depuis 2018, et les pages citent encore Java 7, Log4J (avant la faille Log4Shell), Spring Boot Admin, JavaMelody, Delphix et MQ Series.
- **Logs :** le format `yyyy-mm-dd HH:mm:ss` n'a pas de fuseau horaire, alors qu'aujourd'hui on fait des logs structurés en JSON avec des horodatages ISO 8601 en UTC.
- **Health check :** la page recommande de tester la connexion à chaque dépendance. Avec Kubernetes, c'est devenu un piège : si la sonde qui décide du redémarrage (liveness) teste les dépendances, une panne en aval fait redémarrer tous les pods en cascade. Le bon conseil est inversé.
- **`smart-assistance` (chatbots, callbots) :** la page est entièrement dépassée par les LLM.
- **Backpressure par pools de threads ajustés dynamiquement :** c'est daté, avec les virtual threads de Java et les systèmes managés.

## 5. Les LLM rendent-ils tout ça inutile ?

Non. Pour la partie solide, c'est même l'inverse.

1. **Un LLM sait déjà ce qu'il y a dans les livres**, et la version de 2024 le prouve. Cette couche n'a plus de valeur en tant que documentation.
2. **Les décisions ne s'écrivent pas en code.** Choisir le niveau de service visé, le comportement de secours, ou la gravité qui justifie de réveiller quelqu'un : la doc dit déjà « avec le métier ». Un LLM peut coder un circuit breaker, mais il ne sait pas quoi afficher à l'utilisateur quand il s'ouvre.
3. **Le code généré par IA reproduit exactement ces anti-patterns par défaut.** Il ne traite que le cas où tout se passe bien (« everything works all the time »), ajoute des retries naïfs, avale les erreurs et produit des logs bruyants. La checklist d'origine gagne donc en valeur comme critère de relecture.
4. **Les agents IA qui exploitent la production reproduisent ces erreurs encore plus vite.** Un agent qui redémarre un serveur au moindre souci, c'est « overkill response » et « rushing to solution » à la vitesse machine. « Collecter les preuves avant de corriger » devient une règle de sécurité pour ces agents.
5. **Les patterns d'organisation ne bougent pas.** Le héros indispensable, le toil, l'Andon, les post-mortems sans recherche de coupable : rien de tout ça n'est remplacé par l'IA.

## 6. Conclusion

Comme encyclopédie, ce repo ne sert plus à grand-chose. Comme vocabulaire commun et checklist d'exploitabilité pour l'informatique d'entreprise, il garde de la valeur, à condition de le réduire à la vingtaine de passages issus du terrain. On pourrait aussi le transformer en règles de relecture, pour des humains comme pour des agents IA.

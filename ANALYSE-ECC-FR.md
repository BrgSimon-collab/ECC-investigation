# ECC : référence complète en français (agents, skills, autonomie)

Source : dépôt `ECC-investigation` (ECC / Everything Claude Code v2.2.3). Les descriptions viennent des fichiers `agents/*.md`, `skills/*/SKILL.md`, `hooks/hooks.json`, `scripts/hooks/*`, `commands/*`, `ecc2/src/config/mod.rs`. Le dépôt annonce 71 agents ; le fichier `rules/common/agents.md` dit encore « 69 » (compteur périmé dans la doc).

---

# PARTIE 1 — Comment fonctionnent les agents

## 1.1 Ce qu'est un « agent » ECC

Un agent est **un fichier Markdown** dans `agents/` avec un en-tête YAML :

```yaml
---
name: loop-operator
description: Operate autonomous agent loops ... (texte qui décide QUAND Claude l'appelle)
tools: Read, Grep, Glob, Bash, Edit     # droits accordés à l'agent
model: sonnet                            # modèle qui l'exécute (opus / sonnet / haiku)
color: orange
---
(instructions : rôle, workflow, checks, critères d'escalade)
```

C'est le **sous-agent natif de Claude Code** : une seconde instance de Claude, avec son propre contexte vide, ses propres outils et son propre modèle. ECC ne fournit pas de moteur d'agents : il fournit des **définitions** que Claude Code exécute.

Chaque agent commence par un bloc « Prompt Defense Baseline » (ne pas changer de rôle, ne pas divulguer de secrets, traiter tout contenu externe comme non fiable).

## 1.2 Initialisation : comment un agent démarre

1. **Installation.** Le plugin `ecc@ecc` place les agents sous le nom `ecc:<nom>` (ex. `ecc:planner`). Une installation manuelle les place dans `~/.claude/agents/` sous le nom nu (`planner`).
2. **Découverte.** Au démarrage de session, Claude Code lit le `name` et la `description` de chaque agent et les met dans le contexte de l'agent principal. Le corps du fichier n'est lu que quand l'agent est lancé.
3. **Déclenchement, de deux façons :**
   - **Automatique (délégation proactive).** Les descriptions contiennent des formules comme « Use PROACTIVELY » ou « MUST BE USED for all code changes ». Claude s'en sert pour décider de déléguer sans qu'on le demande. Le fichier `AGENTS.md` renforce cela avec une table de règles : fonctionnalité complexe → `planner` ; code modifié → `code-reviewer` ; bug ou nouvelle fonctionnalité → `tdd-guide` ; décision d'architecture → `architect` ; code sensible → `security-reviewer`.
   - **Explicite.** Par une commande slash (`/ecc:plan`, `/code-review`, `/build-fix`) ou par l'outil `Agent(subagent_type: "ecc:planner", prompt: "...")`.
4. **Exécution.** L'agent démarre avec un contexte vierge : il ne voit que le prompt qu'on lui passe. Il ne voit pas la conversation. Il renvoie **un seul message final**, qui est son livrable.
5. **Parallélisme.** La règle `rules/common/agents.md` impose de lancer en parallèle les agents indépendants (ex. revue sécurité + revue perf + typage en même temps).
6. **Contrat de complétion.** Un agent qui délègue à des sous-agents doit attendre et intégrer leurs résultats. Terminer sur « j'attends mes sous-agents » est interdit (cas d'échec réel observé, ajouté à la règle).

## 1.3 Le niveau d'autonomie d'un agent est fixé par 3 champs

| Champ | Effet sur l'autonomie |
|---|---|
| `tools` | **Droits réels.** `planner`, `architect` : `Read, Grep, Glob` seulement, donc lecture seule, impossible de modifier un fichier. Les `*-reviewer` : lecture + `Bash` (lancer des commandes d'analyse), pas d'écriture. Les `*-build-resolver`, `tdd-guide`, `refactor-cleaner`, `e2e-runner` : `Write/Edit/Bash`, donc ils modifient le code. Les `network-*` : `Read, Grep` (+ `Bash` pour le dépannage), jamais d'édition. |
| `model` | `opus` pour la réflexion lourde (planner, architect, healthcare-reviewer, spec-miner). `sonnet` par défaut. `haiku` pour les tâches légères et bon marché (doc-updater, comment-analyzer, docs-lookup, agents du pipeline open-source). |
| `description` | Détermine la fréquence de déclenchement automatique (« PROACTIVELY » = souvent ; « MUST BE USED » = systématique). |

**Conséquence pratique :** la séparation des rôles est une barrière de sécurité. Les agents qui jugent (reviewers) ne peuvent pas modifier ce qu'ils jugent ; ceux qui corrigent (resolvers) sont bornés par « corrections minimales, pas de changement d'architecture ».

## 1.4 Automatisation : les hooks (le « système nerveux »)

Les hooks sont des scripts Node.js que **Claude Code exécute lui-même** à des moments précis, que l'agent le veuille ou non. Ils sont déclarés dans `hooks/hooks.json` et passent tous par `scripts/hooks/run-with-flags.js`, qui applique les profils.

| Événement | Ce qui tourne | Effet |
|---|---|---|
| **SessionStart** | `session-start-bootstrap`, `plan-canvas-sessions` | Recharge le contexte, la mémoire de session précédente, les instincts |
| **UserPromptSubmit** | `hookify-runtime` | Applique vos règles personnalisées sur chaque prompt |
| **PreToolUse** (avant chaque outil) | `pre-bash-dispatcher` (bloque `npm run dev` hors tmux, rappel avant `git push`, contrôle qualité avant `git commit`), **`gateguard-fact-force`**, `config-protection`, `governance-capture`, `mcp-health-check`, `doc-file-warning`, `suggest-compact`, `observe` (async), `hookify-runtime` | Peut **bloquer** (code de sortie 2) ou avertir |
| **PostToolUse** | `posttooluse-dispatcher` (formatage, typecheck, avertissement `console.log`, détection de build terminé / PR créée) | Retour immédiat à l'agent |
| **PostToolUseFailure** | `mcp-health-check`, `skill-run-tracker` | Suit les pannes de serveurs MCP et de skills |
| **PreCompact** | `pre-compact` | Sauvegarde l'état avant que le contexte soit compacté |
| **Stop** (fin de chaque réponse) | `stop-format-typecheck` (timeout 300 s), `check-console-log`, `session-end`, `evaluate-session`, **`cost-tracker`**, `desktop-notify` | Vérifie, comptabilise le coût, notifie |
| **SessionEnd** | `session-end-marker` | Marque la fin de session |

**GateGuard** (le garde-fou le plus original) : au lieu de demander « êtes-vous sûr ? » (à quoi un LLM répond toujours oui), il **refuse la première édition ou commande** tant que l'agent n'a pas présenté des faits concrets : qui importe ce fichier, quelle est l'API publique, quel est le schéma de données, quelles cibles une commande destructrice touche, quel est le plan de retour arrière. Il est « fail-closed » : si l'entrée du hook est tronquée, il bloque plutôt que de laisser passer.

## 1.5 Contrôle : les profils et interrupteurs

Variables d'environnement lues par `scripts/lib/hook-flags.js` :

| Variable | Valeurs | Effet |
|---|---|---|
| `ECC_HOOKS_ENABLED` | `true` (défaut) / `false` | Coupe tous les hooks |
| `ECC_HOOK_PROFILE` | `minimal` / `standard` (défaut) / `strict` | `minimal` : socle réduit (`hookify-runtime` et le dispatcher `pre-bash`, qui bloque notamment une entrée tronquée). `standard` : tous les hooks marqués `standard,strict`. `strict` : idem, avec les contrôles les plus serrés |
| `ECC_DISABLED_HOOKS` | liste d'identifiants séparés par des virgules | Désactive des hooks précis (ex. `pre:edit-write:gateguard-fact-force`) |
| `ECC_GATEGUARD` | on/off | Interrupteur propre à GateGuard |
| `ECC_DISABLED_MCPS` | liste | Filtre les serveurs MCP à l'installation |
| `ECC_DRY_RUN` | 1/0 | Simulation |
| `ECC_HOOK_INPUT_MAX_BYTES` | entier | Taille max de l'entrée lue par un hook |

Règles de conception des hooks (`.claude/rules/node.md`) : tout hook sort avec le code 0 sur erreur non critique (il ne doit jamais bloquer l'outil par accident) ; les hooks bloquants doivent répondre en moins de 200 ms et sans réseau ; les hooks asynchrones ont un timeout de 30 s maximum.

Autres freins : le skill `safety-guard` (modes **Careful** : confirme `rm -rf`, `git push --force`, `DROP TABLE`, `kubectl delete`, `--no-verify`, etc. ; **Freeze** : verrouille l'écriture à un dossier ; **Guard** : les deux) et `/hookify` (créer vos propres règles « avertir » ou « bloquer » en Markdown).

## 1.6 Les 4 niveaux d'autonomie

| Niveau | Mécanisme | Qui décide | Garde-fous |
|---|---|---|---|
| **1. Assisté** | Vous tapez `/ecc:plan`, `/code-review`, etc. | Vous | Hooks + permissions Claude Code |
| **2. Délégation automatique** | Claude lance lui-même `planner`, `code-reviewer`, `tdd-guide`… selon `AGENTS.md` | Claude, dans un cadre fixé | Outils limités par agent, hooks |
| **3. Pipelines à portes** (`orch-*`) | Recherche → Plan → TDD → Revue → Commit, chaque phase confiée à l'agent adapté | Claude, avec 2 validations humaines | **Porte 1 : validation du plan** (pas de code avant). **Porte 2 : confirmation du commit**. Taille de la tâche classée en trivial / small / standard / large pour dimensionner le processus. Le document dit explicitement : « gated, not autonomous » |
| **4. Boucles autonomes** | `/loop-start`, `continuous-agent-loop`, `gan-style-harness`, `ralphinho-rfc-pipeline`, `claude -p` chaînés | La boucle, supervisée par `loop-operator` | Voir 1.7 |

## 1.7 Les boucles autonomes en détail

**`/loop-start [pattern] [--mode safe|fast]`** prépare une boucle, ne la lance pas aveuglément. Patterns : `sequential`, `continuous-pr`, `rfc-dag`, `infinite`. Mode `safe` (défaut) : portes de qualité strictes et points de contrôle ; mode `fast` : portes réduites. Il vérifie que les tests passent avant la première itération, que `ECC_HOOK_PROFILE` n'est pas coupé et qu'une **condition d'arrêt explicite** existe, puis écrit un runbook dans `.claude/plans/` et affiche les commandes de lancement et de suivi.

**`loop-operator`** (l'agent superviseur). Vérifie avant de partir : portes de qualité actives, référence d'évaluation existante, chemin de retour arrière, isolation par branche ou worktree. **Il escalade** (donc s'arrête et vous prévient) si : aucun progrès sur deux points de contrôle consécutifs ; échecs répétés avec la même trace ; dérive de coût hors budget ; conflits de fusion qui bloquent la file.

**Harnais GAN (`/gan-build`)** : trois agents. `gan-planner` transforme une phrase en spécification et en grille d'évaluation. `gan-generator` code et corrige selon le retour. `gan-evaluator` teste l'application **vivante** via Playwright, note selon la grille et écrit son retour. Boucle bornée : **15 itérations max** (`--max-iterations`) et **seuil de réussite 7,0** (`--pass-threshold`) par défaut ; modes d'évaluation `playwright`, `screenshot`, `code-only`.

**Autres motifs** (skill `autonomous-loops`, remplacé par `continuous-agent-loop`) : pipeline séquentiel de `claude -p` (un contexte neuf par étape, `set -e` pour arrêter à la première erreur, routage de modèle opus/sonnet par étape) ; REPL NanoClaw ; boucle infinie agentique ; boucle PR continue avec CI ; motif **De-Sloppify** (passe de nettoyage distincte après chaque implémentation) ; DAG piloté par RFC avec file de fusion.

**Autres contrôles de sécurité des boucles :** `loop-design-check` (détecte une boucle qui tourne en rond, qui triche sur son vérificateur ou qui mène une mauvaise réponse à terme), `santa-method` (deux relecteurs indépendants doivent tous deux valider), `delivery-gate` (hook d'arrêt qui empêche Claude de conclure tant que les contrôles ne passent pas), `operator-approval-loop` (tout message sortant rédigé par un agent passe par une approbation de l'opérateur).

## 1.8 Mémoire et apprentissage automatiques

- **Mémoire de session** : les hooks `session-start`, `pre-compact`, `session-end` sauvegardent et rechargent l'état entre sessions (`/save-session`, `/resume-session`).
- **Apprentissage continu v2** : le hook `observe` enregistre vos actions ; un agent observateur en arrière-plan (modèle Haiku) les analyse et crée des **« instincts »** (petits comportements appris, confiance de 0,3 à 0,9, portée projet ou globale). Un instinct fréquent dans plusieurs projets peut être promu en global. `/evolve` regroupe les instincts en skills. **Réglages par défaut : `observer.enabled: false`** (désactivé), analyse toutes les 5 min, minimum 20 observations. L'observateur n'est pas supporté sous Windows natif.
- **Coût** : `cost-tracker` écrit un journal local à la fin de chaque réponse ; `/cost-report` et le skill `cost-tracking` le lisent.

## 1.9 Orchestration multi-agents à grande échelle (ECC 2.0 « alpha »)

Le dossier `ecc2/` (Rust, SQLite, tableau de bord terminal) est un **prototype**, décrit comme « alpha, pas le produit final ». Ses réglages par défaut (`ecc2/src/config/mod.rs`) montrent la philosophie d'autonomie :

| Réglage | Défaut |
|---|---|
| `max_parallel_sessions` | 8 |
| `max_parallel_worktrees` | 6 |
| `auto_create_worktrees` | **true** (une branche isolée par agent) |
| `auto_merge_ready_worktrees` | **false** (jamais de fusion automatique) |
| `auto_dispatch_unread_handoffs` | **false** (pas de passage de relais automatique) |
| `auto_terminate_stale_sessions` | **false** |
| `cost_budget_usd` | 10,0 |
| `token_budget` | 500 000 |
| `risk_thresholds` | seuils de score de risque avec alertes de budget |

Lecture : par défaut, **isolation oui, fusion et relais automatiques non**. L'humain garde la décision d'intégrer.

Côté skills, `team-agent-orchestration` modélise chaque agent comme un coéquipier avec un propriétaire, un périmètre, un état (backlog → ready → running → review → blocked → merged → archived), des preuves et une **porte de fusion**. `claude-devfleet` et `dmux-workflows` lancent des agents parallèles dans des worktrees isolés.

## 1.10 Ce qu'ECC ne fait pas (limites à connaître)

- **ECC ne gère pas les permissions de Claude Code.** Les demandes d'autorisation (modifier un fichier, lancer une commande) restent celles de votre configuration Claude Code. ECC ajoute des hooks par-dessus ; il ne remplace pas ce cadre. Un mode « tout autoriser » annulerait une partie des garde-fous.
- Les hooks sont du **code exécuté sur votre machine** avec vos droits : à relire avant installation.
- La délégation automatique dépend du jugement de Claude : elle est fréquente mais non garantie. Le déterminisme vient des hooks et des commandes explicites, pas des descriptions.
- Un agent ne voit pas la conversation : si le prompt de délégation est pauvre, le résultat l'est aussi.
- Hors Claude Code, le support est partiel (matrice de compatibilité dans le README).

---

# PARTIE 2 — Les 71 agents

Légende outils : **L** = lecture seule (Read/Grep/Glob) ; **L+B** = lecture + Bash (exécute des analyses, ne modifie pas) ; **É** = peut écrire/éditer le code. Modèle entre parenthèses.

## Planification, architecture, analyse de code

| Agent | Ce qu'il fait | Droits |
|---|---|---|
| planner (opus) | Plan d'implémentation détaillé : étapes, fichiers, dépendances, risques, ordre. Auto-activé pour les fonctionnalités complexes | L |
| architect (opus) | Conception de systèmes, scalabilité, décisions techniques | L |
| code-architect (sonnet) | Plan de fonctionnalité fondé sur les conventions du code existant : fichiers, interfaces, flux de données, ordre de build | L+B |
| code-explorer (sonnet) | Retrace les chemins d'exécution et cartographie couches et dépendances d'une fonctionnalité existante | L |
| spec-miner (opus) | Extrait des spécifications comportementales (exigences, invariants) d'un code existant pour OpenSpec | L+B + écrit les specs |
| a11y-architect (sonnet) | Accessibilité WCAG 2.2 (web et natif) : conception de composants et audit | É |
| performance-optimizer (sonnet) | Goulots d'étranglement, fuites mémoire, taille de bundle, rendu | É |
| homelab-architect (sonnet) | Plan de réseau domestique ou petit labo, avec étapes sûres et retour arrière | L |
| network-architect (sonnet) | Architecture réseau d'entreprise multi-sites | L |

## Revue de code (lecture seule)

| Agent | Ce qu'il fait |
|---|---|
| code-reviewer (sonnet) | Revue générale qualité, sécurité, maintenabilité. « MUST BE USED for all code changes » |
| security-reviewer (sonnet) | Vulnérabilités : secrets, SSRF, injections, crypto, OWASP Top 10 |
| silent-failure-hunter (sonnet) | Erreurs avalées, replis dangereux, erreurs non propagées |
| type-design-analyzer (sonnet) | Qualité de conception des types : encapsulation, invariants |
| pr-test-analyzer (sonnet) | Qualité de couverture de tests d'une PR (comportements réels) |
| comment-analyzer (haiku) | Exactitude des commentaires et risque de « pourrissement » |
| commit-reviewer (sonnet) | Relit le diff indexé et rédige un message Conventional Commits fidèle à l'historique |
| database-reviewer (sonnet) | PostgreSQL / Supabase : requêtes, schéma, migrations, sécurité |
| typescript-reviewer, python-reviewer, go-reviewer, rust-reviewer, java-reviewer, kotlin-reviewer, swift-reviewer, csharp-reviewer, cpp-reviewer, php-reviewer, ruby-reviewer, fsharp-reviewer, flutter-reviewer (sonnet) | Revue experte par langage : idiomes, concurrence, sécurité mémoire, async, typage, performances. « MUST BE USED » sur les projets du langage |
| react-reviewer, vue-reviewer, django-reviewer, fastapi-reviewer (sonnet) | Revue par framework : hooks et frontière serveur/client (React), Composition API et réactivité (Vue), ORM/DRF/migrations (Django), async/Pydantic/OpenAPI (FastAPI) |
| mle-reviewer (sonnet) | Machine learning en production : contrats de données, reproductibilité, évaluation, monitoring, retour arrière |
| rag-pipeline-reviewer (sonnet) | Pipelines RAG : découpage, embeddings, qualité de récupération, évaluation |
| healthcare-reviewer (opus) | Sécurité clinique, exactitude des aides à la décision, conformité PHI |
| bengali-reviewer (sonnet) | Texte bengali : Unicode, normalisation, internationalisation |
| network-config-reviewer (sonnet) | Configs de routeurs/commutateurs : sécurité, commandes risquées, références périmées |
| agent-evaluator (sonnet) | Note le travail d'un agent sur 5 axes (exactitude, complétude, clarté, actionnabilité, concision) |

## Correction de build et d'erreurs (peuvent modifier le code, corrections minimales)

| Agent | Ce qu'il fait |
|---|---|
| build-error-resolver (sonnet) | Erreurs de build et de types TypeScript, diffs minimaux, « build vert vite » |
| cpp-build-resolver | C++ / CMake / éditeur de liens / templates |
| go-build-resolver | Go : build, vet, linter |
| rust-build-resolver | Cargo, emprunts (borrow checker), Cargo.toml |
| java-build-resolver | Maven / Gradle, Spring Boot ou Quarkus auto-détecté |
| kotlin-build-resolver | Kotlin / Gradle |
| swift-build-resolver | Swift / Xcode / SPM / signature de code |
| dart-build-resolver | `dart analyze`, Flutter, `pub`, `build_runner` |
| django-build-resolver | pip/Poetry, migrations, `collectstatic` |
| react-build-resolver | Vite, webpack, Next.js, hydratation, frontières serveur/client |
| pytorch-build-resolver | PyTorch : formes de tenseurs, CUDA, gradients, DataLoader, précision mixte |
| harmonyos-app-resolver | HarmonyOS : ArkTS / ArkUI, gestion d'état V2, API |

## Tests, nettoyage, documentation

| Agent | Ce qu'il fait | Droits |
|---|---|---|
| tdd-guide (sonnet) | Impose « test d'abord », vise 80 %+ de couverture | É |
| e2e-runner (sonnet) | Tests de bout en bout (Vercel Agent Browser, repli Playwright), met les tests instables en quarantaine | É |
| refactor-cleaner (sonnet) | Supprime le code mort (knip, depcheck, ts-prune) et les doublons | É |
| code-simplifier (sonnet) | Simplifie le code récent sans changer le comportement | É |
| doc-updater (haiku) | Met à jour documentation et « codemaps » | É |
| docs-lookup (haiku) | Documentation de bibliothèques à jour via Context7 (MCP) | L |

## Autonomie et pilotage

| Agent | Ce qu'il fait |
|---|---|
| loop-operator (sonnet) | Supervise les boucles autonomes, détecte blocages et dérives, escalade (voir 1.7) |
| harness-optimizer (sonnet) | Améliore fiabilité et coût de la configuration d'agents par évaluation (pass@k) |
| gan-planner / gan-generator / gan-evaluator (sonnet) | Les trois rôles du harnais GAN (voir 1.7) |
| conversation-analyzer (haiku) | Analyse des transcriptions pour proposer des hooks (`/hookify` sans argument) |

## Pipeline open source (3 étapes enchaînées)

| Agent | Ce qu'il fait |
|---|---|
| opensource-forker (haiku) | Copie le projet, retire secrets (20+ motifs) et références internes, nettoie l'historique git |
| opensource-sanitizer (sonnet) | Vérifie qu'il ne reste ni secrets ni données personnelles ; rapport PASS / FAIL / PASS-WITH-WARNINGS |
| opensource-packager (haiku) | Génère README, LICENSE, CONTRIBUTING, CLAUDE.md, modèles d'issues |

## Réseau, communication, marketing

| Agent | Ce qu'il fait |
|---|---|
| network-troubleshooter (sonnet) | Diagnostic réseau en lecture seule par couches OSI |
| chief-of-staff (sonnet) | Trie e-mails, Slack, LINE, Messenger en 4 niveaux (ignorer / info / réunion / action requise) et rédige les brouillons de réponse |
| marketing-agent (sonnet) | Stratégie et rédaction : campagnes, positionnement, pages, e-mails, réseaux |
| seo-specialist (sonnet) | Audits SEO techniques, balises, données structurées, Core Web Vitals |

---

# PARTIE 3 — Les 311 skills

Un skill est un **dossier avec un `SKILL.md`** : des instructions et du savoir chargés **à la demande** quand la description correspond à votre tâche (contrairement aux règles, chargées en permanence). Il est invoqué par Claude automatiquement ou par `/nom`.

## 3.1 Pilotage d'agents, boucles et orchestration

| Skill | Rôle |
|---|---|
| agent-architecture-audit | Diagnostic de la pile d'une application d'agents (12 couches) : régressions de wrapper, pollution mémoire, discipline des outils |
| agent-eval | Comparaison de Claude Code, Aider, Codex… sur vos tâches (réussite, coût, temps) |
| agent-harness-construction | Conception des outils et des observations d'un agent pour qu'il réussisse mieux |
| agent-introspection-debugging | Débogage structuré d'un échec d'agent : capture, diagnostic, reprise |
| agent-payment-x402 | Paiements x402 pour agents, avec budgets par tâche et portefeuilles non custodiaux |
| agent-runtime-gateway-smoke-test | Test de fumée d'une API d'agent locale, d'un tunnel et d'un rappel de bac à sable |
| agent-security-hardening | Durcissement des frameworks d'agents qui traitent du contenu non fiable |
| agent-self-evaluation | Auto-notation d'un travail sur 5 axes avec preuves |
| agent-sort | Plan d'installation ECC sur mesure : trie skills/commandes/hooks en « quotidien » et « bibliothèque » |
| agentic-engineering | Façon de travailler d'un ingénieur d'agents : évals d'abord, décomposition, routage de modèles par coût |
| agentic-os | Systèmes multi-agents persistants sur Claude Code (noyau, agents, mémoire fichiers, tâches planifiées) |
| ai-first-engineering | Modèle d'équipe quand les agents écrivent une grande part du code : revue, responsabilités |
| autonomous-agent-harness | Fait de Claude Code un système autonome : mémoire persistante, opérations planifiées, file de tâches |
| autonomous-loops | Motifs de boucles autonomes (conservé pour compatibilité) |
| continuous-agent-loop | Boucles autonomes avec portes de qualité, évaluations et reprise |
| loop-design-check | Concevoir/auditer une boucle : tourne en rond, triche, mauvaise réponse menée à terme |
| claude-devfleet | Orchestration d'agents parallèles dans des worktrees isolés, avec rapports |
| dmux-workflows | Orchestration multi-agents avec dmux (panneaux tmux), multi-harnais |
| dynamic-workflow-mode | Harnais locaux à une tâche, portes d'évaluation et extraction de skills |
| enterprise-agent-ops | Cycle de vie, observabilité, moindre privilège, arrêt d'urgence pour agents longue durée |
| gan-style-harness | Boucle générateur/évaluateur pour construire des applications (voir 1.7) |
| nanoclaw-repl | REPL de session persistante de la maison, commandes slash, changement de modèle |
| nasiko-control-plane | Cycle de vie de la CLI expérimentale Nasiko, lecture seule + installation avec consentement |
| orch-pipeline | Moteur commun des `orch-*` : recherche → plan → TDD → revue → commit, deux portes humaines |
| orch-add-feature | Nouvelle fonctionnalité de bout en bout, via les agents adaptés |
| orch-build-mvp | MVP depuis un document de spécification : tranches verticales fines |
| orch-change-feature | Modifier un comportement existant : tests d'abord, puis code |
| orch-fix-defect | Corriger un bug : test de régression qui échoue, puis correctif |
| orch-refine-code | Refactorisation sans changer le comportement |
| plan-orchestrate | Lit un plan, classe chaque étape et émet la commande ECC prête à coller |
| plan-canvas | Plans et pages HTML dans un canevas local où l'humain annote, discute et approuve |
| ralphinho-rfc-pipeline | Découpe une RFC en DAG de tâches multi-agents avec file de fusion |
| blueprint | Une phrase d'objectif → plan de construction multi-sessions, une étape = une PR |
| recursive-decision-ledger | Essais répétés avec registre de décisions append-only et portes de promotion |
| santa-method | Vérification adversariale : deux relecteurs indépendants doivent valider |
| team-agent-orchestration | Équipes d'agents : propriétaire, périmètre, Kanban, branches isolées, portes de fusion |
| team-builder | Sélecteur interactif de personas d'agents (jusqu'à cinq) |
| dev-team | Simule une équipe (PM, architecte, dev, QA) répondant à un même problème |
| council | Quatre voix pour trancher une décision ambiguë |
| council-multi-model | Ajoute une critique externe (Codex) après le conseil |
| parallel-execution-optimizer | Transforme une tâche en graphe de voies parallèles aux surfaces d'écriture isolées |
| iterative-retrieval | Affine la récupération de contexte pour les sous-agents qui manquent d'informations |
| operator-approval-loop | Contrat d'approbation humaine pour les messages sortants rédigés par un agent |
| counterparty-channel-discipline | Règles par canal pour agents dans des canaux partagés avec des tiers externes |
| hermes-imports | Convertit des workflows d'opérateur Hermes en skills ECC assainis |
| openclaw-persona-forge | Forge une personnalité complète (SOUL.md) pour un agent OpenClaw |
| deepseek-harness-setup | Installer ECC sur le harnais DeepSeek |
| terminal-opener | Ouvre un exécutable dans un terminal visible via un plan de lancement sans shell |
| terminal-ops | Exécution de commandes dépôt avec preuves : CI, correctifs ciblés |

## 3.2 Mémoire, contexte, apprentissage, coûts, configuration d'ECC

| Skill | Rôle |
|---|---|
| ck | Mémoire par projet (Context Keeper) : init, save, resume, forget |
| unified-memory | Contexte et relais partagés entre Claude, Codex, Cursor… via un coffre local |
| continuous-learning-v2 | Apprentissage par « instincts » avec score de confiance (voir 1.8) |
| continuous-learning | Ancienne version v1, dépréciée |
| context-budget | Contrôle de la pression de contexte et audit statique des composants |
| strategic-compact | Propose un `/compact` manuel à des moments logiques |
| token-budget-advisor | Propose des profondeurs de réponse (25/50/75/100 %) avec estimation de tokens |
| token-card | Carte de consommation de tokens commitée dans le dépôt |
| cost-tracking | Rapports de coût et d'usage depuis le journal local |
| cost-aware-llm-pipeline | Routage de modèles par complexité, budgets, cache de prompts |
| prompt-caching-strategy | Optimise le taux de succès du cache de prompts |
| prompt-optimizer | Améliore un brouillon de prompt et propose les commandes/skills ECC adaptés |
| config-gc | Nettoyage de `~/.claude` : skills, mémoire, hooks, permissions obsolètes |
| configure-ecc | Assistant de configuration ECC (portée, profil de hooks) |
| ecc-guide | Répond aux questions sur ECC en lisant le dépôt vivant |
| ecc-recipes | Associe un workflow décrit au bon groupe de commandes ECC |
| rules-distill | Extrait des principes transverses des skills pour en faire des règles |
| skill-comply | Mesure si skills, règles et agents sont réellement suivis |
| skill-host-compat | Lint de compatibilité des skills pour Codex/Cursor |
| skill-scout | Cherche un skill existant avant d'en créer un |
| skill-stocktake | Audit de qualité des skills et commandes |
| growth-log | Entrées de journal de progression : cause racine, règle transférable |
| knowledge-ops | Gestion et synchronisation d'une base de connaissances multi-stockage |
| workspace-surface-audit | Audit du dépôt, des MCP, plugins, connecteurs pour recommander les bons skills |
| automation-audit-ops | Inventaire des tâches, hooks, connecteurs et MCP actifs, recoupements |
| mcp-dependency-review | Revue statique des configs MCP : paquets non épinglés |
| mcp-server-patterns | Construire des serveurs MCP (SDK Node/TypeScript) |

## 3.3 Garde-fous, vérification, qualité du processus

| Skill | Rôle |
|---|---|
| gateguard | Porte « forcer les faits » avant toute première édition ou commande (voir 1.4) |
| safety-guard | Modes Careful / Freeze / Guard contre les opérations destructrices |
| hookify-rules | Créer des règles hook en Markdown (avertir ou bloquer) |
| delivery-gate | Hook d'arrêt : Claude ne conclut pas tant que les contrôles ne passent pas |
| verification-loop | Vérification en 6 phases : build, types, lint, tests, grep sécurité, revue du diff |
| plankton-code-quality | Formatage, lint et corrections à chaque édition via hooks |
| codehealth-mcp | Score de santé du code (CodeScene) avant/après édition |
| change-impact | Analyse d'impact avant changement, comparée au diff réel |
| module-regression | Registre de régression inter-modules dans de gros dépôts |
| test-collaboration | Inventaire des actifs de test relié aux exigences et risques |
| ai-regression-testing | Tests de régression propres au code généré par IA |
| coding-standards | Conventions de base inter-projets : nommage, lisibilité, immutabilité |
| eval-harness | Développement piloté par les évaluations (EDD) |
| production-audit | Audit de préparation à la production, preuves locales |
| security-scan | Analyse de `.claude/` avec AgentShield (secrets, permissions, injections) |
| security-review | Revue de sécurité (auth, entrées, secrets, API, paiement) |
| security-bounty-hunter | Chasse aux vulnérabilités exploitables à distance |
| conventional-commit-review | Rédaction et contrôle de messages de commit |
| click-path-audit | Retrace chaque bouton jusqu'aux changements d'état pour trouver les actions qui s'annulent |
| canary-watch | Surveille une URL déployée après une release |
| browser-qa | Vérification d'interface après déploiement via automatisation de navigateur |

## 3.4 Méthode, produit, planification, documentation

| Skill | Rôle |
|---|---|
| search-first | Chercher un outil existant avant d'écrire du code |
| tdd-workflow | Cycle test d'abord, vert minimal, refactor, 80 %+ de couverture |
| intent-driven-development | Critères d'acceptation vérifiables avant d'implémenter |
| product-capability | PRD → plan de capacités avec contraintes et invariants |
| product-lens | Diagnostic « pourquoi construire » avec décision go/no-go |
| contract-first | Un contrat machine (OpenAPI, Protobuf…) fait foi entre services |
| architecture-decision-records | Décisions d'architecture en fichiers ADR numérotés |
| context-and-decisions | Contexte de domaine stable (CONTEXT.md) et décisions difficiles à inverser |
| brownfield-retrospective | Rétro-application du processus ECC sur un projet existant |
| inherit-legacy-style | Évite la dérive de style sur un projet ancien |
| codebase-onboarding | Guide d'intégration d'un code inconnu + CLAUDE.md de départ |
| code-tour | Visites guidées CodeTour avec ancrages fichier/ligne |
| docs-governance | Audit en lecture seule des rôles de documentation |
| living-docs-governance | Empêche la documentation de pourrir sur un projet long |
| documentation-lookup | Documentation à jour via Context7 |
| story-lifecycle | Boucle épique → story → sprint par fichiers |
| project-flow-ops | Flux d'exécution GitHub + Linear |
| git-workflow | Branches, commits, fusion vs rebase |
| github-ops | Issues, PR, CI, releases avec `gh` |
| jira-integration | Tickets Jira via MCP |
| unified-notifications-ops | Alertes unifiées GitHub, Linear, bureau, hooks |

## 3.5 Langages (patterns, tests, standards)

| Langage | Skills |
|---|---|
| C++ | cpp-coding-standards (C++ Core Guidelines), cpp-testing (GoogleTest, CTest, sanitizers) |
| C# / .NET | csharp-testing (xUnit), dotnet-patterns (DI, async) |
| Dart / Flutter | dart-flutter-patterns, flutter-dart-code-review |
| F# | fsharp-testing |
| Go | golang-patterns, golang-testing |
| Java | java-coding-standards, jpa-patterns, mybatis-patterns, tinystruct-patterns |
| Kotlin / Android | kotlin-patterns, kotlin-testing, kotlin-coroutines-flows, kotlin-exposed-patterns, kotlin-ktor-patterns, android-clean-architecture, compose-multiplatform-patterns |
| Perl | perl-patterns, perl-security, perl-testing |
| Python | python-patterns, python-testing, pytorch-patterns |
| Ruby | rails-patterns |
| Rust | rust-patterns, rust-testing |
| Swift / iOS | swiftui-patterns, swift-concurrency-6-2, swift-actor-persistence, swift-protocol-di-testing, foundation-models-on-device, liquid-glass-design, ios-icon-gen |
| TypeScript / JS | bun-runtime, nodejs-keccak256 (piège Keccak vs SHA3), error-handling |

## 3.6 Frameworks web et backend

| Skill | Rôle |
|---|---|
| angular-developer | Code et architecture Angular |
| django-patterns / django-security / django-tdd / django-verification / django-celery | Django : architecture et DRF, sécurité, tests, boucle de vérification, tâches asynchrones |
| fastapi-patterns | FastAPI, Pydantic v2, injection de dépendances |
| laravel-patterns / laravel-security / laravel-tdd / laravel-verification / laravel-plugin-discovery | Laravel : architecture, sécurité, tests, vérification, recherche de paquets |
| nestjs-patterns | NestJS |
| springboot-patterns / springboot-security / springboot-tdd / springboot-verification | Spring Boot |
| quarkus-patterns / quarkus-security / quarkus-tdd / quarkus-verification | Quarkus 3.x |
| backend-patterns | Patterns backend Node/Express/Next.js |
| api-design | Conception d'API REST |
| api-connector-builder | Ajouter un connecteur en respectant le motif existant |
| hexagonal-architecture | Ports et adaptateurs |
| nextjs-turbopack | Next.js 16+ et Turbopack |
| nuxt4-patterns | Nuxt 4 |
| vue-patterns | Vue 3 / Pinia / Router |
| vite-patterns | Vite |
| react-patterns / react-performance / react-testing / react-native-patterns | React : hooks, performance (règles Vercel), tests, React Native/Expo |
| frontend-patterns | Patterns front React/Next.js |
| ui-to-vue | Convertit des maquettes en composants Vue 3 |

## 3.7 Données, bases, infrastructure

| Skill | Rôle |
|---|---|
| postgres-patterns | PostgreSQL : requêtes, schéma, index |
| mysql-patterns | MySQL/MariaDB |
| redis-patterns | Redis : cache, verrous, limitation de débit |
| clickhouse-io | ClickHouse, analytique |
| prisma-patterns | Prisma ORM et pièges |
| database-migrations | Migrations sûres et réversibles, zéro interruption |
| data-throughput-accelerator | Accélère les mouvements massifs de données |
| content-hash-cache-pattern | Cache par hash de contenu |
| latency-critical-systems | Systèmes sensibles à la latence (p50/p95/p99) |
| benchmark / benchmark-optimization-loop | Mesure de performance et boucle d'optimisation mesurée |
| docker-patterns | Docker et Compose |
| kubernetes-patterns | Kubernetes |
| uncloud | Gestion d'un cluster Uncloud |
| flox-environments | Environnements reproductibles Flox/Nix |
| deployment-patterns | CI/CD, conteneurs, retour arrière |
| dashboard-builder | Tableaux de bord Grafana/SigNoz qui répondent à de vraies questions |
| generating-python-installer | Installeurs Windows Python compacts (Nuitka, Inno Setup) |
| windows-desktop-e2e | Tests E2E d'applications Windows natives |
| e2e-testing | Tests E2E Playwright |
| data-scraper-agent | Agent de collecte de données planifié et enrichi par IA |
| regex-vs-llm-structured-text | Analyser du texte structuré : regex d'abord, LLM en renfort |

## 3.8 Interface, design, vidéo et médias

| Skill | Rôle |
|---|---|
| accessibility / frontend-a11y | Accessibilité WCAG 2.2 (général / React) |
| design-system | Extraire/auditer un système de design (tokens) |
| frontend-design-direction | Direction de design pour l'UI de production |
| make-interfaces-feel-better | Détails d'interface (espacements, mouvement, zones de clic) |
| motion-foundations / motion-patterns / motion-advanced | Animation React : bases, patterns, avancé |
| frontend-slides | Présentations HTML animées |
| ui-demo | Vidéos de démo d'interface avec Playwright |
| manim-video | Explications animées avec Manim |
| remotion-video-creation | Vidéo en React avec Remotion |
| video-editing | Montage vidéo assisté par IA |
| videodb | Indexation et montage vidéo/audio avec VideoDB |
| fal-ai-media | Génération image/vidéo/audio via fal.ai |
| taste / taste-distillation / taste-application / tasteforge-video | Direction créative vidéo : vocabulaire esthétique, extraction d'un « style pack », application, workflow complet |
| blender-motion-state-inspection | Inspection de rigs et d'animations Blender |
| esign-field-placement | Placement de champs de signature électronique |
| nutrient-document-processing | Conversion, OCR, extraction de documents (API Nutrient) |
| visa-doc-translate | Traduction de pièces de visa en PDF bilingue |
| i18n-sync | Synchronisation des fichiers de traduction |
| bengali-nlp | Traitement de texte bengali |

## 3.9 Réseau et homelab

| Skill | Rôle |
|---|---|
| cisco-ios-patterns | Revue de configs Cisco IOS |
| netmiko-ssh-automation | Automatisation SSH réseau (Netmiko) |
| network-bgp-diagnostics | Diagnostic BGP |
| network-config-validation | Contrôles avant déploiement de configs réseau |
| network-interface-health | Santé des interfaces (erreurs, CRC, duplex) |
| homelab-network-setup | Planification de réseau domestique |
| homelab-network-readiness | Check-list VLAN, DNS filtrant, VPN |
| homelab-vlan-segmentation | VLAN IoT/invité/serveurs |
| homelab-pihole-dns | Pi-hole |
| homelab-wireguard-vpn | VPN WireGuard |

## 3.10 Santé

| Skill | Rôle |
|---|---|
| healthcare-cdss-patterns | Aide à la décision clinique : interactions, doses, scores |
| healthcare-emr-patterns | Dossiers patients (EMR/EHR) |
| healthcare-eval-harness | Évaluation de sécurité patient |
| healthcare-phi-compliance | Données de santé protégées (PHI) |
| hipaa-compliance | Entrée HIPAA |

## 3.11 Recherche, ML, science

| Skill | Rôle |
|---|---|
| deep-research | Rapports cités multi-sources (firecrawl, exa) |
| research-ops | Recherche d'état courant avec preuves |
| exa-search | Recherche neuronale Exa |
| market-research | Étude de marché et diligence |
| osint-investigation | Méthode d'enquête OSINT |
| mle-workflow | ML en production |
| ml-adoption-playbook | Ajouter du ML à un code non-ML |
| recsys-pipeline-architect | Pipelines de recommandation en six étapes |
| scientific-db-pubmed-database | PubMed / NCBI |
| scientific-db-uspto-database | Brevets et marques USPTO |
| scientific-pkg-gget | Requêtes génomiques (gget) |
| scientific-thinking-literature-review | Revue de littérature systématique |
| scientific-thinking-scholar-evaluation | Évaluation de travaux académiques |

## 3.12 Contenu, marketing, concurrence, investisseurs, réseaux sociaux

| Skill | Rôle |
|---|---|
| article-writing | Articles et guides dans une voix distinctive |
| brand-discovery | Entretien d'identité de marque en 8 modules |
| brand-voice | Profil de style d'écriture dérivé de vos textes |
| content-engine | Contenus natifs par plateforme |
| crosspost | Diffusion X, LinkedIn, Threads, Bluesky |
| social-publisher | Publication planifiée sur 13 plateformes |
| x-api | API X/Twitter |
| connections-optimizer | Réorganisation de réseau X/LinkedIn |
| social-graph-ranker | Classement de graphe social, introductions chaudes |
| lead-intelligence | Prospection et scoring de signaux |
| marketing-campaign | Campagne marketing de bout en bout |
| seo | Audit et mise en œuvre SEO |
| benchmark-methodology / competitive-platform-analysis / competitive-report-structure | Analyse concurrentielle : méthode, périmètre, rapport |
| investor-materials / investor-outreach | Pitch decks, notes, e-mails aux investisseurs |
| master-agreement-generator | Brouillons de contrats-cadres à partir d'un gabarit |

## 3.13 Opérations métier et bureautique

| Skill | Rôle |
|---|---|
| email-ops | Tri et envoi d'e-mails avec preuves |
| messages-ops | Lecture de messages, codes à usage unique |
| google-workspace-ops | Drive, Docs, Sheets, Slides |
| mailtrap-email-integration | Envoi d'e-mails transactionnels (Mailtrap) |
| customer-billing-ops | Facturation clients (Stripe) |
| finance-billing-ops | Revenus, prix, remboursements, vérité du modèle de facturation |
| ecc-tools-cost-audit | Audit des dépenses d'ECC Tools : PR en rafale, contournement de quota, fuite de modèles premium |
| carrier-relationship-management | Transporteurs : sélection, tarifs, appels d'offres |
| customs-trade-compliance | Douanes et conformité commerciale |
| energy-procurement | Achat d'électricité et de gaz |
| inventory-demand-planning | Prévision de la demande et stocks |
| logistics-exception-management | Exceptions de fret |
| production-scheduling | Ordonnancement de production |
| quality-nonconformance | Non-conformités qualité (FDA, IATF, AS9100) |
| returns-reverse-logistics | Retours et logistique inverse |

## 3.14 Écosystème Itô, marchés prédictifs, web3

| Skill | Rôle |
|---|---|
| ito-baskets | Paniers et marchés prédictifs (lecture seule) |
| ito-compute | Inventaire GPU, appels d'offres de calcul |
| ito-inference | Mise en service de modèles sur calcul réservé |
| ito-training | Entraînement ML sur calcul réservé |
| prediction-market-oracle-research | Marchés prédictifs comme source de signaux |
| prediction-market-risk-review | Revue de risques des flux de paris/paniers |
| llm-trading-agent-security | Sécurité des agents de trading avec portefeuille |
| defi-amm-security | Sécurité des contrats AMM Solidity |
| evm-token-decimals | Erreurs de décimales entre chaînes EVM |
| repo-scan | Installe un analyseur de dépôt externe depuis un commit épinglé |
| opensource-pipeline | Pipeline fork → nettoyage → emballage pour publication |

## 3.15 Skills de scaffold de workflow du dépôt

Les trois entrées `add-language-rules`, `database-migration` et `feature-development` ne sont pas dans `skills/` : ce sont des commandes de `.claude/commands/` (« workflow command scaffold »), listées comme skills par la session. Le fichier `.claude/rules/ecc-guardrails.md` indique qu'elles ont été générées par ECC Tools à partir de l'historique du dépôt. Elles ne comptent pas dans les 311 skills.

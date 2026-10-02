<div align="center">

# Pandi

**Plateforme d'apprentissage adaptative — développement & cybersécurité.**
Un moteur pédagogique qui suit la maîtrise réelle de l'étudiant, un mentor IA qui guide sans jamais donner la solution, et des labs de cybersécurité isolés pour pratiquer sur de vraies vulnérabilités, sans risque.

![Status](https://img.shields.io/badge/status-en%20développement-orange)
![Stack](https://img.shields.io/badge/stack-React%20%2F%20Node%20%2F%20Docker-informational)
![License](https://img.shields.io/badge/license-propriétaire-lightgrey)

</div>

---

> **À propos de ce dépôt.** Pandi est un projet propriétaire en développement ; le code source n'est pas publié ici. Ce dépôt documente l'architecture, les choix techniques et le fonctionnement de la plateforme, à des fins de portfolio.

## Le principe

La plupart des plateformes d'apprentissage proposent le même parcours à tout le monde. Pandi construit un **profil de compétences réel** pour chaque étudiant (ce qu'il maîtrise vraiment, pas seulement ce qu'il a validé) et choisit dynamiquement le prochain exercice en fonction de ses erreurs récentes et de ses points faibles. Un mentor IA local donne des indices progressifs — jamais la réponse directe — contextualisés sur le niveau réel de l'étudiant.

```
React (dashboard, cours, exercices)
          │
          ▼
API Express ──┬── Skill Graph        (graphe de compétences et dépendances)
              ├── Student Model      (maîtrise par compétence, erreurs, XP)
              ├── Exercise Engine    (sélection adaptative + exécution sandboxée)
              ├── AI Gateway         (mentor IA, indices progressifs)
              └── Cyber Labs         (environnements vulnérables isolés)
```

## Ce qui existe aujourd'hui

- **Comptes & progression réelle.** Authentification JWT, profil de compétences par utilisateur, XP et streak calculés depuis l'activité réelle (pas de chiffres fictifs).
- **Sélection adaptative d'exercices.** Le prochain exercice proposé dépend de la maîtrise réelle et des erreurs récentes de l'étudiant, parmi un pool couvrant plusieurs niveaux et formats.
- **Exécution de code sandboxée.** Chaque soumission tourne dans un conteneur Docker éphémère et isolé : pas d'accès réseau, ressources plafonnées, système de fichiers en lecture seule, utilisateur non privilégié.
- **Mentor IA local.** Un modèle de langage (Qwen, via Ollama, 100 % local) donne des indices progressifs contextualisés sur le profil réel de l'étudiant — jamais la solution en premier.
- **Labs de cybersécurité isolés.** Injection SQL, IDOR, XSS stocké : des applications volontairement vulnérables, chacune dans son conteneur, sans accès Internet sortant, pour s'entraîner sur de vraies failles sans risque.
- **Tableau de bord.** Activité récente, progression par compétence, niveau — branché sur les données réelles de l'étudiant.

## Ce qui n'est pas encore là

Génération d'exercices par IA à plus grande échelle, missions combinant plusieurs labs, et un pool de conteneurs pré-chauffés pour réduire la latence de démarrage. La feuille de route complète est suivie en interne.

## Stack

`React 18` · `Vite` · `TypeScript` · `Tailwind CSS` · `Node.js` · `Express` · `PostgreSQL` · `Prisma` · `Docker` · `Ollama (Qwen)`

---

## Architecture technique

<details>
<summary><strong>Détails pour les développeurs</strong> (sandbox, IA, labs, modèle de données)</summary>

### Monorepo

```
apps/
  web/        React + Vite — dashboard, cours, exercices
  api/         Express — routes, moteur pédagogique
packages/
  shared/      types TypeScript partagés (Course, Exercise, StudentModel, SkillGraph)
```

npm workspaces. `npm run dev` lance le front (`:5173`) et l'API (`:4000`) en parallèle, avec proxy `/api` en dev.

### Modèle de données (Prisma / PostgreSQL)

`User`, `StudentModel`, `SkillMastery`, `Misconception`, `Skill`, `Course`, `Lesson`, `Exercise`, `ExerciseSubmission`, `Lab`, `LabSession`. Le `StudentModel` distingue la maîtrise déclarée de la maîtrise réellement démontrée par les soumissions, et conserve les erreurs récurrentes (`Misconception`) pour nourrir la sélection d'exercices et les indices du mentor IA.

### Exécution de code — `DockerCodeRunner`

Chaque soumission d'exercice est évaluée dans un conteneur Docker **éphémère et isolé**, lancé avec :

- `--network none` — aucun accès réseau
- `--memory 128m --memory-swap 128m --cpus 0.5 --pids-limit 64` — ressources plafonnées, fork-bombs bloquées
- `--read-only` — système de fichiers racine immuable
- `--cap-drop ALL --security-opt no-new-privileges` — aucune capability Linux
- utilisateur non-root, `--rm`, timeout côté hôte en filet de sécurité

Le choix du JavaScript pour les exercices (plutôt que Python, envisagé initialement) garde le `CodeRunner` cohérent avec le reste de la stack : le code peut être évalué directement dans un `vm` Node avant sandboxing, sans orchestrer un second runtime.

### Mentor IA — `AI Gateway`

Une abstraction `AIProvider` découple la plateforme du modèle utilisé. Le provider par défaut (`OllamaProvider`) appelle un modèle Qwen local via Ollama (jamais exposé publiquement), avec un prompt contextualisé sur le rôle, le niveau réel de l'étudiant et ses compétences fortes/faibles issues du `StudentModel`. Règle pédagogique appliquée côté prompt : jamais la solution directe — des indices numérotés, progressifs, puis la réponse complète seulement en dernier recours.

### Cyber Labs — isolation réseau

Chaque lab (`sql-injection-login`, `idor-invoices`, `xss-guestbook`) est une petite application volontairement vulnérable, buildée en image Docker dédiée. Le lab XSS valide l'exploitation avec un vrai navigateur headless (Puppeteer) plutôt qu'une simulation.

Isolation réseau : les conteneurs de labs tournent sur un réseau Docker bridge dédié, publiés uniquement sur `127.0.0.1`. L'isolation **sortante** (empêcher un conteneur exploité d'atteindre Internet ou les autres services) est appliquée au niveau pare-feu via une règle `DOCKER-USER` qui autorise le retour vers les plages RFC1918 et bloque tout le reste depuis le subnet des labs — un réseau Docker `--internal` a été testé puis abandonné, car il bloquait aussi le port-forwarding entrant légitime depuis l'hôte.

### Sécurité

JWT + bcrypt pour l'authentification, sandbox Docker durcie pour toute exécution de code utilisateur, labs réseau-isolés avec pare-feu dédié, Ollama non exposé publiquement.

</details>

---

<sub>Documentation de portfolio — le code source reste privé.</sub>

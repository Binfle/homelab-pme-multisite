# Process — cadre méthodologique du projet

> Quand produit-on quoi, pourquoi, comment, et qui le fait. Doc de référence pour ne pas se noyer dans la doc.

---

## 1. Vocabulaire commun

| Terme | Définition courte |
|---|---|
| **Phase** | Grand ensemble du projet. Phase 1 = MVP, Phase 2 = bonus / approfondissement. |
| **Sprint** | Période de travail (1 à 2 semaines) consacrée à une grappe thématique cohérente. Produit un livrable. |
| **Grappe** | Thème cohérent regroupant plusieurs services liés (ex: réseau bas-niveau, identité, services applicatifs). Une grappe = un sprint. *(Terme interne, pas standard de l'industrie.)* |
| **ADR** | Architecture Decision Record. Trace écrite d'une décision structurante. |
| **Runbook** | Procédure opérationnelle de remédiation ou maintenance courante (ex: "redémarrer le tunnel WireGuard"). |
| **Procédure d'installation** | Documentation de mise en place initiale d'un composant technique. Se distingue du runbook par son déclencheur (volonté de mise en place vs problème). Cf. ADR-006. |
| **CR** | Compte-rendu de réunion. Daté, court, factuel. |
| **Livrable** | Artefact terminé selon les 5 critères de la section 4. |
| **MVP** | Minimum Viable Product : le périmètre minimum qui rend le projet "présentable". |

---

## 2. Quand produit-on quoi ?

### 2.1 ADR — Architecture Decision Record

**Pourquoi**
- Tracer une décision structurante : qu'a-t-on choisi, pourquoi, qu'a-t-on écarté.
- Permettre à un futur lecteur (l'autre membre, un recruteur, soi-même 6 mois plus tard) de comprendre l'historique des choix.
- Forcer la rigueur de la décision : si on doit la justifier par écrit, on y réfléchit mieux.

**Quand**
- Dès qu'on tranche entre **plusieurs options crédibles** sur un sujet impactant (techno, archi, méthode).
- **Avant** de commencer à implémenter, pas après.
- **Pas** pour chaque petit choix de config (port, version, paquet).

**Qui**
- Rédigé par celui qui propose la décision.
- **Reviewé et accepté par l'autre** via Pull Request.

**Format**
- Modèle dans `docs/adr/template.md`.
- 1 à 2 pages max.
- Numéroté séquentiellement : ADR-001, ADR-002, etc.

**Cycle de vie**
- `proposé` → `accepté` → éventuellement `déprécié` ou `remplacé par ADR-XXX`.
- **Un ADR accepté ne se modifie plus.** Si on change d'avis, on écrit un nouvel ADR qui remplace l'ancien.

**Exemples qui méritent un ADR**
- Choix d'hyperviseur (ADR-004)
- Choix d'une stack IaC (ADR-002)
- Décision d'architecture multi-sites (ADR-001)
- Plan d'adressage IP (ADR-003)

**Exemples qui ne méritent PAS un ADR**
- "On utilise Debian 13" → README ou note dans le rôle Ansible
- "Le port SSH sera 2222" → variable Ansible
- "On nomme la VM `dc01`" → convention de nommage dans `docs/conventions.md`

---

### 2.2 Runbook

**Pourquoi**
- Procédure reproductible pour une opération de remédiation ou de maintenance courante.
- Garantit que **l'autre membre puisse intervenir** si tu n'es pas là.
- En production réelle, un service sans runbook = un service non exploitable.

**Quand**
- **Après** avoir exécuté la procédure pour la première fois.
- **Pas avant** : un runbook spéculatif est faux 9 fois sur 10.
- **Pas trop tard** : passé 5 exécutions, on a oublié des étapes.

**Qui**
- Rédigé par celui qui a exécuté la procédure en premier.
- Relu et **testé** par l'autre membre, qui doit pouvoir l'exécuter sans aide.

**Format**
- Modèle dans `docs/runbooks/template.md`.
- Sections : symptômes, diagnostic, résolution, validation, post-mortem.

**Exemples obligatoires**
- "Restauration depuis Restic" → **dès** que Restic est en place. Une sauvegarde non testée = pas de sauvegarde.
- "Reconfigurer le tunnel WireGuard"
- "Redémarrer pfSense après plantage"

**Exemples non nécessaires**
- "Installer Debian" → tutoriel public déjà disponible

**À ne pas confondre avec une procédure d'installation** (cf. 2.3).

---

### 2.3 Procédure d'installation

**Pourquoi**
- Documenter la **mise en place initiale** d'un composant technique
  (pfSense, AD, BIND9, Docker, NetBox, etc.).
- Permettre à l'autre membre de reproduire l'installation à l'identique
  sur son site.
- Constituer la mémoire vive du projet : ce qui a été fait, dans quel
  ordre, avec quels paramètres.

**Quand**
- **Pendant** ou **immédiatement après** la première installation réussie.
- **Pas avant** : un brouillon spéculatif est faux.
- Idéalement issue d'un journal de bord brut transformé en doc reproductible.

**Qui**
- Rédigée par le membre ayant exécuté l'installation initiale.
- **Testée et validée** par l'autre membre, qui doit pouvoir reproduire
  l'installation chez lui sans aide. Tant que ce test croisé n'a pas
  eu lieu, la procédure est marquée `à valider`.

**Format**
- Modèle dans `docs/installations/template.md`.
- Sections : objet, pré-requis, variables (si paramétrée par site),
  étapes phasées, validation finale, erreurs connues, liens.

**Distinction avec un runbook**
- Une **procédure d'installation** est déclenchée par une *volonté de mise
  en place* (`docs/installations/`).
- Un **runbook** est déclenché par un *problème* ou une *maintenance
  courante* (`docs/runbooks/`).
- Décision tracée dans `docs/adr/006-procedures-vs-runbooks.md`.

**Exemples qui méritent une procédure d'installation**
- Installation initiale de pfSense sur VMware
- Déploiement initial d'AD DC1
- Mise en service de Traefik en DMZ

**Exemples qui ne méritent PAS une procédure d'installation**
- "Créer une VM Debian dans VMware" → tutoriel public déjà disponible
- "Installer un paquet sur Debian" → rôle Ansible, pas une doc à part

---

### 2.4 Compte-rendu de réunion (CR)

**Pourquoi**
- Trace les décisions structurantes prises hors-stream.
- Sert de rétrospective formelle en clôture de sprint.

**Quand** (cf section 6)
- **Clôture de sprint** : rétrospective écrite (ce qui a marché, ce qui a coincé, leçons).
- **Décision structurante** prise lors d'une session dédiée hors travail courant.
- **Pas pour** : le travail courant (les décisions de détail se prennent à voix haute en stream Discord et sont tracées dans les commits / ADR / runbooks selon la portée).

**Qui**
- Rédigé par l'un des deux à tour de rôle.
- Validé par l'autre dans la foulée.

**Format**
- Modèle dans `docs/meetings/template.md`.
- Sections : fait depuis la dernière fois, points bloquants, décisions, plan suivant.
- Stocké dans `docs/meetings/AAAA-MM-JJ-<sujet>.md`.

---

### 2.5 README et docs annexes

**README principal**
- Vu par tout le monde (recruteurs inclus). Doit être clair, accrocheur, à jour.
- Mis à jour à chaque fin de sprint pour refléter ce qui marche désormais.

**README de dossier** (ex: `docs/README.md`, `docs/adr/README.md`, `ansible/roles/<role>/README.md`)
- Explique ce qu'il y a dans le dossier et comment l'utiliser.
- Au moins un README par rôle Ansible.

**Documentation utilisateur** (ex: "comment se connecter à l'AD")
- Rédigée si un "utilisateur" fictif doit pouvoir s'en servir.
- Bonus pour le portfolio.

---

## 3. Workflow Git et reviews

(Rappel et compléments de `CONTRIBUTING.md`.)

- **Pas de commit direct sur `main`**. Tout passe par PR.
- **Une branche par sujet** : `feat/wireguard-site-to-site`, `docs/adr-004`, `fix/dns-secondary`.
- **Review obligatoire** par l'autre membre avant merge.
- **Squash & merge** à la fusion (un commit propre par PR sur `main`).
- **Conventional Commits** : `feat(scope): ...`, `fix(scope): ...`, `docs(scope): ...`.
- **Une PR = un sujet**. Si une PR mélange config réseau + doc + nouveau rôle, la découper.

---

## 4. Définition de "livrable"

Un livrable est livrable quand **les 5 conditions** sont réunies :

1. **Ça marche** : démontrable, reproductible. Pas "ça a marché une fois en juin".
2. **C'est versionné** : code/config dans le repo, mergé sur `main`.
3. **C'est documenté** :
   - Au minimum un README ou une section dans le README principal.
   - ADR si décision structurante.
   - Runbook ou procédure d'installation si pertinent.
4. **C'est testable par l'autre** :
   - L'autre membre peut le rejouer chez lui (ou comprendre pourquoi pas, ex: tunnel WG cross-sites).
   - Pour Ansible : le rôle s'applique sur une VM neuve sans intervention manuelle.
   - Pour pfSense (UI) : captures d'écran annotées + procédure compensent l'impossibilité d'automatiser.
5. **C'est mis en avant** : entrée dans le README principal ou dans la section "fonctionnalités".

**Pas de livrable = sprint pas terminé.** On ne passe pas à la grappe suivante avant que la précédente coche les 5 cases.

---

## 5. Workflow d'un sprint

### 5.1 Avant le sprint
- Session de cadrage en stream (~30 min) : identifier les 3-5 tâches principales et tracer leur découpage.
- Créer les issues GitHub correspondantes, milestone "Sprint N".
- Mettre à jour `docs/sprints/sprint-N-<theme>.md` (objectif, livrables, DoD).

### 5.2 Pendant le sprint
- Pas de point hebdo formel : la communication se fait en stream Discord permanent.
- Commits réguliers, PR au fil de l'eau.
- Reviews dans la journée (l'autre membre est généralement disponible en quelques minutes via Discord).
- Pas de CR par défaut. Si un sujet bloque ou nécessite une décision structurante, on l'acte en ADR ou dans un CR dédié.

### 5.3 Fin du sprint
- Vérifier les 5 critères de "livrable" sur chaque tâche.
- Mettre à jour le README principal avec les nouvelles fonctionnalités.
- Rédiger les ADR, runbooks et procédures d'installation manquants.
- Réunion de clôture : ce qui a marché, ce qui a coincé, leçons apprises (CR formel dans `docs/meetings/`).
- Tag Git `sprint-N` sur le commit final.

---

## 6. Cadence de travail et CR

Le binôme bosse **en stream permanent** quand il travaille sur le projet (~10h/jour, tous les jours de la semaine). Toutes les discussions, décisions de détail, débuggages se font en direct vocal. Cela rend caduques les rituels classiques d'équipes distribuées (daily standups, points hebdo, CR détaillés).

### Cérémonies retenues (le minimum utile)

**CR formel** (template `docs/meetings/template.md`) — uniquement pour :
- **Clôture de sprint** : rétrospective écrite (ce qui a marché, ce qui a coincé, leçons apprises, ajustements pour le sprint suivant).
- **Décisions structurantes** prises lors d'une session dédiée : si la décision aboutit à un ADR, le CR est facultatif (l'ADR fait office de trace). Sinon, un CR court justifie le choix.

**Pas de CR pour** :
- Le travail courant (les décisions de détail se prennent à voix haute en stream)
- Les daily ou les points intermédiaires
- Les sessions de pair-programming standard

### Trace des décisions

Vu le format stream, **le risque c'est que les décisions importantes se prennent oralement et soient oubliées**. Discipline à tenir :

- Toute décision qui influence l'archi ou la stack → **ADR** (cf section 2.1)
- Toute convention récurrente → notée dans `docs/conventions.md` ou un README de dossier
- Toute info technique qui se redécouvre → ajoutée au runbook ou à la procédure concernée

Si après 5 minutes de discussion on s'aperçoit qu'on retombera dessus plus tard, **on l'écrit**. Pas en CR formel — en commit message, en commentaire de PR, en issue, ou en ADR selon la portée.

### Workflow Git

Le binôme adapte son workflow Git selon le contexte :

- **Sujets parallélisables** (deux composants indépendants) → branches séparées + PR croisées avec review immédiate (l'autre est sur Discord, ça prend 2 minutes)
- **Sujets complexes nécessitant pair-programming** → branche commune, commits qui mentionnent les deux contributeurs via `Co-authored-by:` dans le message
- **Travail solo** (l'un avance pendant une absence de l'autre) → PR classique avec review différée

Le workflow PR (cf `CONTRIBUTING.md`) reste la **règle par défaut** sur `main`, indépendamment du contexte de travail. La branch protection l'impose techniquement.

---

## 7. Structure du repo (rappel)
├── README.md
├── CONTRIBUTING.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── architecture.md
│   ├── addressing-plan.md
│   ├── process.md                ← ce document
│   ├── adr/
│   │   ├── README.md             ← index des ADR
│   │   ├── template.md
│   │   ├── 001-architecture-deux-sites-vpn.md
│   │   ├── 002-stack-iac-ansible-seul-phase1.md
│   │   ├── 003-plan-adressage.md
│   │   ├── 004-choix-hyperviseur-phase-1.md
│   │   └── 006-procedures-vs-runbooks.md
│   ├── installations/            ← procédures d'install (cf ADR-006)
│   │   ├── README.md
│   │   ├── template.md
│   │   └── ...
│   ├── runbooks/                 ← remédiation et maintenance
│   │   ├── README.md
│   │   ├── template.md
│   │   └── ...
│   ├── meetings/
│   │   ├── template.md
│   │   └── AAAA-MM-JJ-<sujet>.md
│   └── sprints/
│       ├── sprint-0-cadrage.md
│       └── ...
├── diagrams/
│   └── network-overview.txt
├── ansible/
│   ├── inventory/
│   ├── roles/
│   └── playbooks/
└── .github/
├── pull_request_template.md
└── ISSUE_TEMPLATE/

---

## 8. Pièges à éviter

- **Documenter pour documenter** : si un ADR n'apporte rien à un futur lecteur, ne pas l'écrire.
- **Runbook ou procédure spéculatifs** : écrire avant d'avoir exécuté → erreurs garanties.
- **CR trop longs** : un CR > 1 page n'est pas relu. Synthétiser.
- **PR fourre-tout** : mélanger doc + config + nouveau rôle → review impossible.
- **Sauvegarde non testée** : Restic configuré ≠ sauvegardes valides. Test de restauration obligatoire avant de cocher la case.
- **"Ça marche chez moi"** : si ça ne se rejoue pas ailleurs, ce n'est pas un livrable.
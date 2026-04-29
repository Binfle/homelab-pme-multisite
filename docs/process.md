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
| **Runbook** | Procédure opérationnelle reproductible (ex: "redémarrer le tunnel WireGuard"). |
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
- Procédure reproductible pour une opération courante ou de remédiation.
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
- "Joindre un poste Linux à AD via SSSD"
- "Redémarrer pfSense après plantage"

**Exemples non nécessaires**
- "Créer une VM dans VirtualBox" → tutoriel public déjà disponible
- "Installer Debian" → idem

---

### 2.3 Compte-rendu de réunion (CR)

**Pourquoi**
- Trace les décisions, points bloquants, plan jusqu'à la prochaine réunion.
- Permet à un membre absent de rattraper.
- Évite de redécider 2 fois la même chose.

**Quand**
- À chaque réunion **synchrone** (visio, vocal, présentiel).
- **Pas** pour les chats Discord/SMS asynchrones (qui restent dans Discord).

**Qui**
- Rédigé par l'un des deux à tour de rôle.
- Validé par l'autre dans la foulée.

**Format**
- Modèle dans `docs/meetings/template.md`.
- 3 sections : fait depuis la dernière fois, points bloquants, plan jusqu'à la prochaine.
- Stocké dans `docs/meetings/AAAA-MM-JJ.md`.

---

### 2.4 README et docs annexes

**README principal**
- Vu par tout le monde (recruteurs inclus). Doit être clair, accrocheur, à jour.
- Mis à jour à chaque fin de sprint pour refléter ce qui marche désormais.

**README de dossier** (ex: `docs/README.md`, `ansible/roles/<role>/README.md`)
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
   - Runbook si procédure d'exploitation existe.
4. **C'est testable par l'autre** :
   - L'autre membre peut le rejouer chez lui (ou comprendre pourquoi pas, ex: tunnel WG cross-sites).
   - Pour Ansible : le rôle s'applique sur une VM neuve sans intervention manuelle.
   - Pour pfSense (UI) : captures d'écran annotées + runbook compensent l'impossibilité d'automatiser.
5. **C'est mis en avant** : entrée dans le README principal ou dans la section "fonctionnalités".

**Pas de livrable = sprint pas terminé.** On ne passe pas à la grappe suivante avant que la précédente coche les 5 cases.

---

## 5. Workflow d'un sprint

### 5.1 Avant le sprint
- Réunion de cadrage du sprint (~30 min).
- Identifier les 3-5 tâches principales.
- Créer les issues GitHub correspondantes, milestone "Sprint N".
- Mettre à jour `docs/sprints/sprint-N-<theme>.md` (objectif, livrables, tâches par membre, DoD).

### 5.2 Pendant le sprint
- Une à deux réunions de point hebdo (~20 min).
- Chaque CR dans `docs/meetings/`.
- Commits réguliers, PR au fil de l'eau.
- Reviews dans les 24-48h max après ouverture d'une PR.

### 5.3 Fin du sprint
- Vérifier les 5 critères de "livrable" sur chaque tâche.
- Mettre à jour le README principal avec les nouvelles fonctionnalités.
- Rédiger les ADR et runbooks manquants.
- Réunion de clôture : ce qui a marché, ce qui a coincé, leçons apprises.
- Tag Git `sprint-N` sur le commit final.

---

## 6. Rôle de Claude (l'IA) dans le process

⚠️ **Important** : Claude est un **scribe et un relecteur**, pas un décideur ni un membre de l'équipe au sens fort.

### 6.1 Ce que Claude peut faire (sur demande explicite)

- Rédiger un brouillon d'**ADR** à partir d'un contexte fourni (sujet, options, décision, alternatives).
- Rédiger un brouillon de **runbook** à partir d'une procédure réellement exécutée et décrite.
- Rédiger un **CR** à partir de notes brutes ou d'un audio fourni.
- Relire un texte et signaler incohérences, oublis, fautes.
- Expliquer un concept (Ansible, pfSense, AD, etc.) à la demande, en restant honnête sur ses limites de connaissance et l'âge de ses informations.
- Suggérer des améliorations méthodologiques.

### 6.2 Ce que Claude ne peut pas faire

- ❌ **Lire automatiquement** vos discussions Discord, SMS, mails ou messageries entre les conversations. Aucune intégration n'existe.
- ❌ **Écouter en arrière-plan** vos discussions vocales en temps réel.
- ❌ **Maintenir une mémoire** entre conversations sans que la mémoire utilisateur ait été activée dans les paramètres Claude.
- ❌ **Vous notifier** ou vous relancer.
- ❌ **Décider à votre place** sur les choix structurants : Claude suggère, vous tranchez.

### 6.3 Comment utiliser Claude efficacement

- **Donner du contexte** : pointer vers les ADR existants, le README, le sprint en cours.
- **Être explicite** sur ce qu'on demande : "rédige" vs "relis" vs "explique" vs "compare".
- **Vérifier les sorties** : Claude peut se tromper, surtout sur des informations récentes ou des détails techniques pointus. Croisez avec la doc officielle pour les points critiques.
- **Garder Claude au courant des décisions actées** : à chaque nouvelle conversation, lui donner le sprint en cours et les ADR mergés.

---

## 7. Workflow CR de réunion (à expérimenter)

Le binôme n'a pas encore choisi de workflow définitif. Trois options à tester :

### Option A — Notes brutes après réunion → mise en forme
1. Pendant la réunion : prendre des notes très brèves (5-10 lignes) à deux dans un doc partagé.
2. Après la réunion : envoyer les notes à Claude qui produit un CR structuré.
3. Relire et committer.

✅ Léger, rapide, peu intrusif
❌ Demande discipline pendant la réunion

### Option B — Audio enregistré → transcription
1. Enregistrer la réunion (smartphone, OBS, etc.).
2. Envoyer l'audio à Claude *(à tester : formats supportés à confirmer ; en cas d'échec, transcrire d'abord avec Whisper, Otter, ou équivalent, puis envoyer le texte)*.
3. Claude produit un CR structuré.

✅ Aucune prise de note nécessaire pendant la réunion
❌ Confidentialité à considérer si sujets sensibles
❌ Audios longs (>30 min) peuvent dépasser les limites

### Option C — Réunion live avec Claude
1. Pendant la réunion, l'un des deux discute avec Claude en parallèle.
2. À la fin, Claude résume.

✅ CR prêt en fin de réunion
❌ Très intrusif, distrait des échanges humains

**Recommandation** : tester l'**Option A pendant 2-3 réunions**, voir si c'est viable. Passer à l'Option B uniquement si l'Option A est trop lourde.

---

## 8. Structure du repo (rappel)

```
.
├── README.md
├── CONTRIBUTING.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── architecture.md
│   ├── addressing-plan.md
│   ├── threat-model.md
│   ├── process.md                ← ce document
│   ├── adr/
│   │   ├── template.md
│   │   ├── 001-architecture-deux-sites-vpn.md
│   │   ├── 002-stack-iac-ansible-seul-phase1.md
│   │   ├── 003-plan-adressage.md
│   │   └── 004-choix-hyperviseur-phase-1.md
│   ├── runbooks/
│   │   ├── template.md
│   │   └── ...
│   ├── meetings/
│   │   ├── template.md
│   │   └── AAAA-MM-JJ.md
│   └── sprints/
│       ├── sprint-0-cadrage.md
│       ├── sprint-1-fondations.md
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
```

---

## 9. Pièges à éviter

- **Documenter pour documenter** : si un ADR n'apporte rien à un futur lecteur, ne pas l'écrire.
- **Runbook spéculatif** : écrire la procédure avant de l'avoir exécutée → erreurs garanties.
- **CR trop longs** : un CR > 1 page n'est pas relu. Synthétiser.
- **PR fourre-tout** : mélanger doc + config + nouveau rôle → review impossible.
- **Sauvegarde non testée** : Restic configuré ≠ sauvegardes valides. Test de restauration obligatoire avant de cocher la case.
- **"Ça marche chez moi"** : si ça ne se rejoue pas ailleurs, ce n'est pas un livrable.

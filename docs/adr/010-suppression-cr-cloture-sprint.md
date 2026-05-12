# ADR-010 — Suppression des CR de clôture de sprint au profit des traces atomiques

**Statut** : proposé
**Date** : 12/05/2026
**Remplace** : néant
**Remplacé par** : néant

---

## Contexte

`process.md` § 2.4 et § 6 prévoient un **CR formel de clôture de sprint** (rétrospective écrite : ce qui a marché, ce qui a coincé, leçons apprises, ajustements pour le sprint suivant), à stocker dans `docs/meetings/AAAA-MM-JJ-cloture-sprint-N.md`.

En pratique, après le Sprint 0 (où un CR a été rédigé : `2026-04-29-cloture-sprint-0.md`), la décision implicite a été prise de ne plus rédiger ce CR. Le présent ADR formalise et trace cette décision.

Le binôme travaille en stream Discord permanent (~10h/jour, cf `process.md` § 6). Les décisions structurantes vont en ADR, les procédures opérationnelles vont en `installations/` et `runbooks/`, les détails du quotidien vont en commits avec message Conventional Commits. Toute information utile est déjà tracée atomiquement.

Le CR de clôture de sprint dans cette configuration **duplique** ces informations sans en ajouter de nouvelle.

---

## Décision

Supprimer l'obligation du CR de clôture de sprint. La trace de fin de sprint se fait désormais via les artefacts existants :

- **ADR** : décisions structurantes prises pendant le sprint (avec contexte, conséquences positives/négatives, alternatives écartées — déjà très proche d'une rétrospective sur le sujet)
- **Procédures d'installation** (`docs/installations/`) : ce qui a été installé, comment, avec les pièges connus rencontrés
- **Runbooks** (`docs/runbooks/`) : pour les opérations de remédiation et maintenance, avec section "symptômes / diagnostic / résolution / post-mortem"
- **Commits Git** : Conventional Commits qui décrivent atomiquement chaque modification
- **README principal** : mis à jour à chaque fin de sprint pour refléter ce qui marche désormais
- **Tag Git annoté de clôture de sprint** : marqueur de la fin du sprint avec un message court décrivant le périmètre

Le dossier `docs/meetings/` est conservé pour les CR exceptionnels (décisions structurantes prises lors d'une session dédiée qui n'aboutiraient pas à un ADR — cas rare mais possible). Le CR du Sprint 0 (`2026-04-29-cloture-sprint-0.md`) reste comme trace historique.

---

## Conséquences

### Positives

- **Pas de duplication** : chaque information est tracée une seule fois, là où elle est pertinente.
- **Process plus léger** : un livrable de fin de sprint en moins, sans perte d'information.
- **Cohérence avec la cadence stream** : tout est tracé au fil de l'eau, pas dans un format formel post-hoc.
- **Lecture facilitée** pour un lecteur externe : ADR pour le « pourquoi », procédures pour le « comment », commits pour la chronologie. Pas besoin d'un méta-document supplémentaire.

### Négatives

- **Perte de la vue d'ensemble rétrospective** en un seul endroit. Pour avoir une synthèse de sprint, il faut parcourir plusieurs documents.
- **Si la qualité des ADR baisse** (rédaction expéditive, sections « conséquences » mal remplies), la trace rétrospective se perd. La pratique des ADR doit rester rigoureuse.

### Risques

- **Glissement de la pratique** : si on commence aussi à zapper les ADR ou les runbooks, on perd toute trace. Cet ADR ne supprime que les CR, pas les autres types de trace. À surveiller.

---

## Alternatives écartées

### Maintenir le CR de clôture obligatoire

- Format prévu dans `process.md` actuel.
- Écarté car redondant avec les autres traces. Aucune information unique au CR de clôture (la rétrospective est déjà couverte par les sections « conséquences négatives » / « alternatives écartées » / « conditions de révision » des ADR du sprint).

### CR ultra-court (1 paragraphe par sprint)

- Variante allégée du CR.
- Écarté : si on doit faire un CR, autant en faire un complet ; sinon autant ne pas en faire du tout. La demi-mesure n'apporte rien.

### Section « Leçons » dans le README principal par sprint

- Centralisation visible.
- Écarté : grossit le README sprint après sprint, redondant avec les ADR, et incite à reformuler des choses déjà tracées.

### Fichier vivant `docs/lessons-learned.md`

- Vue centralisée des leçons par sprint.
- Écarté pour les mêmes raisons : redondance avec les ADR, fichier supplémentaire à maintenir.

---

## Conditions de révision

Cet ADR sera révisé si :

- La qualité des ADR rédigés au fil des sprints se dégrade au point de ne plus capturer la dimension rétrospective.
- Le binôme constate qu'il a besoin d'une vue synthétique de fin de sprint pour ses propres besoins (préparation du sprint suivant, communication externe, etc.).
- Un évaluateur externe (jury, recruteur, formateur) signale que l'absence de rétrospective formelle nuit à la lisibilité du projet.

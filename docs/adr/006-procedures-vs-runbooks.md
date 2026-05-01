# ADR-006 — Séparation procédures d'installation vs runbooks de remédiation

**Statut** : accepté
**Date** : 01/05/2026
**Remplace** : néant
**Remplacé par** : néant

---

## Contexte

`process.md` §2.2 définit le **runbook** comme une procédure opérationnelle
au format *symptômes → diagnostic → résolution → validation → post-mortem*.
Ce format correspond à la **remédiation** (incident, panne, état dégradé)
ou à la **maintenance courante**.

Pendant le Sprint 1, la rédaction d'une procédure d'**installation initiale**
(pfSense) a fait apparaître un mismatch : ce type de document a une
structure naturelle différente (*pré-requis → étapes → validation finale*),
sans symptôme ni post-mortem.

Trois options ont été considérées :

1. Détourner le format runbook pour les installs.
2. Créer un sous-dossier `runbooks/installation/`.
3. Créer une catégorie distincte `installations/` au même niveau que `runbooks/`.

Plusieurs procédures d'installation sont anticipées dans la suite du
projet (Active Directory, BIND9, Docker host, NetBox, etc.), donc trancher
maintenant est plus rentable que sprint après sprint.

---

## Décision

Créer la catégorie **`installations/`** au même niveau que `runbooks/`,
chacune avec son propre template.
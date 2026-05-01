# Procédures d'installation

Ce dossier regroupe les **procédures d'installation initiale** des composants
techniques du projet : pfSense, Active Directory, BIND9, Docker host, etc.

Une procédure d'installation décrit comment **mettre en place** un composant
de zéro, dans le cadre du projet `homelab-pme-multisite`. Elle suit le format
*pré-requis → étapes → validation finale*, avec une section "Erreurs connues
et solutions" pour les pièges typiques.

## Distinction avec les runbooks

Les **runbooks** (dans [`../runbooks/`](../runbooks/)) couvrent la
**remédiation** (panne, état dégradé) et la **maintenance courante**, avec
un format différent (*symptômes → diagnostic → résolution → validation →
post-mortem*).

Règle de tri rapide :

- Déclencheur = volonté de mise en place initiale → `installations/`
- Déclencheur = problème ou maintenance → `runbooks/`

Décision tracée dans
[ADR-006](../adr/006-procedures-vs-runbooks.md).

## Convention de nommage

Les fichiers prennent le nom du composant en kebab-case, sans préfixe
redondant :

-  `pfsense.md`

Le titre interne du document commence par **"Procédure d'installation —"**
pour lever l'ambiguïté en lecture isolée.

## Modèle

Voir [`template.md`](template.md) pour le squelette à utiliser lors de la
création d'une nouvelle procédure.

## Index

| Composant | Sprint | Statut |
|---|---|---|
| [pfSense](pfsense.md) | Sprint 1 | à valider — test croisé en attente |
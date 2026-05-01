# Runbooks

Ce dossier regroupe les **runbooks de remédiation et de maintenance** des
composants techniques du projet.

Un runbook décrit comment **réagir** à un problème (panne, état dégradé,
incident) ou **exécuter** une opération de maintenance courante. Il suit le
format *symptômes → diagnostic → résolution → validation → post-mortem*.

## Distinction avec les procédures d'installation

Les **procédures d'installation** (dans [`../installations/`](../installations/))
couvrent la **mise en place initiale** d'un composant, avec un format
différent (*pré-requis → étapes → validation finale*).

Règle de tri rapide :

- Déclencheur = problème ou maintenance → `runbooks/`
- Déclencheur = volonté de mise en place initiale → `installations/`

Décision tracée dans
[ADR-006](../adr/006-procedures-vs-runbooks.md).

## Convention de nommage

Les fichiers prennent un nom explicite décrivant l'**action** ou le
**symptôme** traité, en kebab-case :

- o `wireguard-restart.md`
- o `pfsense-config-reset.md`
- o `connectivity-troubleshooting.md`
- x `wireguard.md` (trop vague, on ne sait pas ce que le runbook traite)

Le titre interne du document commence par **"Runbook —"** pour lever
l'ambiguïté en lecture isolée.

## Modèle

Voir [`template.md`](template.md) pour le squelette à utiliser lors de la
création d'un nouveau runbook.

## Index

| Sujet | Sprint | Statut |
|---|---|---|
| *(aucun runbook pour l'instant — à venir au Sprint 1)* | – | – |
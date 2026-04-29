# ADR-002 — Stack IaC : Ansible seul en phase 1

**Statut** : accepté
**Date** : 28/04/2026

## Contexte

Délai souple, projet d'apprentissage assumé, niveau « bases » avec apprentissage en cours de route. Tentation de tout faire en Terraform + Ansible + Packer dès le départ.

## Décision

**Ansible seul** pour la phase 1. Terraform et Packer envisagés en bonus ou phase 2.

L'apprentissage Ansible se fait en parallèle de l'utilisation, pas en pré-requis : on commence par des rôles simples (paquets, users, fichiers de config) et on monte en complexité au fil des sprints.

## Conséquences

**Positives**
- Une seule techno IaC à maîtriser → plus de chance d'arriver au bout.
- Ansible couvre la majorité du périmètre (config, déploiement, orchestration).
- Provisioning manuel des VM acceptable au début (clic-clic Proxmox / VirtualBox).

**Négatives**
- Pas de provisioning entièrement automatisé en phase 1.
- À ajouter dans la phase 2 si on veut le côté « lab reproductible en une commande ».

## Alternatives écartées

- **Terraform + Ansible dès le début** : risque d'enlisement sur l'apprentissage Terraform.
- **Pure scripting bash** : pas idempotent, mauvais signal portfolio.

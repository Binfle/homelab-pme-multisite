# ADR-001 — Architecture deux sites avec VPN site-à-site

**Statut** : accepté
**Date** : 28/04/2026

## Contexte

Le binôme travaille à distance, chacun sur sa machine (32 Go RAM). Pas de serveur partagé. Le projet doit refléter une PME crédible et exploiter cette contrainte plutôt que la subir.

## Décision

Modéliser **deux sites distincts** (siège + agence) hébergés respectivement chez chaque membre, reliés par un **tunnel WireGuard site-à-site** entre les pare-feux pfSense.

## Conséquences

**Positives**
- Scénario réaliste de PME multi-sites.
- Apprentissage concret du VPN site-à-site, du routage inter-sites, des vues DNS.
- Exploite les deux machines plutôt qu'une seule.

**Négatives**
- Si l'un éteint sa machine, l'autre site devient isolé (à documenter dans le runbook).
- Configuration NAT/firewall des box Internet personnelles à gérer (ou Tailscale en repli).

## Alternatives écartées

- **Lab partagé sur un cloud dédié** : coût récurrent, écarté pour la phase 1.
- **Cluster Proxmox 2 nœuds** : nécessite un réseau LAN commun, non disponible.

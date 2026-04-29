# ADR-003 — Plan d'adressage IP

**Statut** : accepté
**Date** : 28/04/2026

## Contexte

Besoin d'un plan d'adressage cohérent, lisible, qui supporte la segmentation par VLAN et le routage inter-sites.

## Décision

- Site A (siège) : `10.10.0.0/16`
- Site B (agence) : `10.20.0.0/16`
- Transit VPN : `10.99.99.0/30`

VLANs en `10.<site>.<vlan>.0/24`. Détail dans [addressing-plan.md](../addressing-plan.md).

## Conséquences

**Positives**
- Lisible : 3e octet = numéro de VLAN, 2e octet = site.
- Espace généreux pour ajouter des sous-réseaux.
- Aucun chevauchement entre sites → routage simple via WireGuard.

**Négatives**
- Adressage privé classe A « luxe » pour 40 personnes (mais c'est pédagogique).

## Alternatives écartées

- `192.168.x.0/24` : trop limité pour la segmentation prévue.
- IPv6 dès le début : reporté en bonus pour ne pas alourdir la phase 1.

# Déploiement réseau d'une PME — From scratch

> Conception et déploiement complet de l'infrastructure réseau et système d'une PME fictive de 40 personnes répartie sur deux sites, en Infrastructure as Code.

## Contexte

ACME Corp (fictive), 40 personnes :
- **Siège** : direction, IT, commerce, R&D (~30 postes)
- **Agence** : antenne régionale (10 personnes)

Besoins : auth centralisée Win + Linux, partage de fichiers, télétravail, site web public, app métier interne, sauvegarde testée, supervision, continuité entre sites.

## Architecture

Deux sites reliés par un tunnel WireGuard site-à-site, chacun avec son propre pare-feu pfSense. Voir [docs/architecture.md](docs/architecture.md), [docs/addressing-plan.md](docs/addressing-plan.md), [diagrams/network-overview.txt](diagrams/network-overview.txt), [docs/adr/](docs/adr/).

## Stack

| Domaine | Choix |
|---|---|
| Hyperviseur (phase 1) | VirtualBox / VMware Workstation Pro |
| Hyperviseur (phase 2 bonus) | Proxmox VE |
| Pare-feu / VPN | pfSense + WireGuard |
| IaC | Ansible |
| Annuaire | Active Directory |
| DNS | AD-DNS (siège) + BIND9 (agence) |
| Auth Linux | SSSD joint à AD |
| Reverse proxy | Traefik |
| PKI interne | step-ca (à arbitrer après Sprint 2) |
| Conteneurs | Docker + Compose |
| Supervision | Prometheus + Grafana |
| Sauvegarde | Restic |
| IPAM | NetBox |

## Périmètre

**MVP** : pare-feux 2 sites, VPN site-à-site, AD + DNS, clients joints (Win + Linux), Traefik + certificats, 1 appli en Docker, monitoring, sauvegarde testée, doc complète.

**Bonus** : 2e DC HA, SIEM Wazuh, CI GitHub Actions, tests Goss, GPO durcissement, IDS Suricata, IPv6, PKI step-ca.

## Découpage en grappes (sprints)

| Sprint | Grappe | Livrable principal |
|---|---|---|
| 0 | Cadrage | Décisions structurantes, ADR fondateurs, prérequis validés |
| 1 | Réseau bas-niveau | pfSense × 2 + tunnel WireGuard site-à-site fonctionnel |
| 2 | Identité & postes | AD DC1, BIND9 split-horizon, clients Win + Linux joints |
| 3 | Services applicatifs | Traefik + 1 appli métier en Docker dans la DMZ |
| 4 | Observabilité & résilience | Prometheus + Grafana + Restic (avec test de restauration) |
| 5 | Source de vérité | NetBox renseigné, doc finale |

Détail dans [docs/sprints/](docs/sprints/).

## Équipe

| Membre | Site | Domaines |
|---|---|---|
| Augustin | Siège (A) | pfSense A, AD, file server, Traefik, monitoring |
| Martial | Agence (B) | pfSense B, BIND9, clients, sauvegarde, NetBox |

Le binôme bosse en pair-programming sur l'ensemble du projet : les deux 
membres maîtrisent toute la stack à la fin du projet.

Communs : tunnel WireGuard, rôles Ansible partagés, CI, doc, ADR.
**Règle** : aucun merge sur `main` sans review de l'autre.

## Workflow

Voir [CONTRIBUTING.md](CONTRIBUTING.md) et [docs/process.md](docs/process.md).

## Licence

MIT

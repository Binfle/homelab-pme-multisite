# Registre des IPs allouées

> Liste des IPs effectivement attribuées dans le projet, par site et par VLAN.
> Document **vivant** — à mettre à jour à chaque ajout/migration/suppression de VM ou d'équipement.

## Convention

- Classification fonctionnelle par dizaines : voir `conventions.md` § 3 et ADR-011.
- Plan global : voir `addressing-plan.md`.
- Stratégie d'attribution : voir ADR-011 § 2.

## États possibles

| État | Sens |
|---|---|
| `En service` | VM / équipement effectivement déployé et opérationnel |
| `Prévu Sprint X` | IP réservée pour un livrable d'un sprint à venir |
| `À migrer` | IP allouée mais à déplacer (dette technique) |

## Stratégie d'attribution possible

| Attribution | Sens |
|---|---|
| `Statique manuel` | IP configurée à la main sur la machine (jamais via DHCP) |
| `DHCP réservé` | IP attribuée par Kea via une réservation MAC (à configurer côté pfSense) |
| `DHCP libre` | IP attribuée dans la plage dynamique `.100-200` (LAN_USR uniquement) |

> ⚠️ **À terme (Sprint 5)** : ce registre sera **remplacé par NetBox** (cf `architecture.md` § 1 et README principal). Le maintien manuel ici est une mesure transitoire.

---

## Site `siege01` (10.10.0.0/16)

### LAN_USR — `10.10.10.0/24`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `siege01` | Statique manuel | En service | Gateway |
| `.100-200` | (plage DHCP libre) | DHCP libre | — | Postes utilisateurs |

### LAN_SRV — `10.10.20.0/24`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `siege01` | Statique manuel | En service | Gateway |
| `.10` | `dc01-siege01` | Statique manuel | Prévu Sprint 2 | AD DC1 |
| `.20` | `traefik-siege01` | DHCP réservé | Prévu Sprint 3 | Reverse proxy interne |
| `.30` | `docker-siege01` | DHCP réservé | Prévu Sprint 3 | Docker host |
| `.40` | `prometheus-siege01` | DHCP réservé | Prévu Sprint 4 | Monitoring |

### DMZ — `10.10.30.0/24`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `siege01` | Statique manuel | En service | Gateway |
| `.10` | `traefik-public-siege01` | DHCP réservé | Prévu Sprint 3 | Reverse proxy public |

### MGMT — `10.10.40.0/24`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `siege01` | Statique manuel | En service | Gateway |
| `.10` | `ansible-siege01` | Statique manuel | En service | Contrôleur Ansible (méthode actuelle, à confirmer côté mainteneur siege01 si bascule en DHCP réservé) |

---

## Site `agence01` (10.20.0.0/16)

### LAN_USR — `10.20.10.0/24`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `agence01` | Statique manuel | En service | Gateway |
| `.100` | `client-debian-agence01-01` | DHCP libre | En service | VM Debian de test (lease DHCP attribué automatiquement, IP non figée) |
| `.100-200` | (plage DHCP libre) | DHCP libre | — | Postes utilisateurs |

### LAN_SRV — `10.20.20.0/24`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `agence01` | Statique manuel | En service | Gateway |
| `.10` | `bind9-agence01` | Statique manuel | Prévu Sprint 2 | BIND9 secondaire |
| `.20` | `netbox-agence01` | DHCP réservé | Prévu Sprint 5 | IPAM |

### DMZ — `10.20.30.0/24`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `agence01` | Statique manuel | En service | Gateway |
| — | — | — | — | DMZ conservée pour pédagogie (`addressing-plan.md`), pas de services prévus |

### MGMT — `10.20.40.0/24`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `agence01` | Statique manuel | En service | Gateway |
| `.100` | `ansible-agence01` | Statique manuel | **À migrer vers `.10`** | Contrôleur Ansible — dette identifiée Sprint 1, à corriger après activation de Kea sur MGMT |

---

## Transit VPN — `10.99.99.0/30`

| IP | Hostname | Attribution | État | Notes |
|---|---|---|---|---|
| `.1` | pfSense `siege01` (côté tunnel) | Statique manuel | En service | Endpoint WireGuard siège |
| `.2` | pfSense `agence01` (côté tunnel) | Statique manuel | En service | Endpoint WireGuard agence |

---

## Dette technique en cours

| Dette | Site | Détail | Cible | Sprint cible |
|---|---|---|---|---|
| Migration contrôleur Ansible | agence01 | `10.20.40.100` → `10.20.40.10` | Aligner sur classification ADR-011 (outils admin = `.10-29`) | Sprint 2 (avant les premières VMs Sprint 2) |

---

## Comment ajouter une entrée

1. Identifier le VLAN du site concerné
2. Choisir une IP libre dans la sous-plage fonctionnelle adéquate (cf `conventions.md` § 3)
3. Ajouter la ligne dans le tableau correspondant ci-dessus
4. Si DHCP réservé : créer aussi la réservation côté pfSense Kea (récupérer la MAC de la VM, ajouter le static mapping)
5. Commit avec message : `docs(ip-registry): allocation <IP> pour <hostname>`
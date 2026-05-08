# Index des ADR

> Liste des Architecture Decision Records du projet, classés et reliés. Mis à jour à chaque ajout ou changement de statut d'un ADR.

## Convention

- Numérotation **séquentielle simple** : ADR-001, ADR-002, …, ADR-099, ADR-100.
- La numérotation reflète l'ordre d'**allocation** (planification incluse), pas strictement l'ordre d'acceptation chronologique. Un numéro peut être **réservé** dans un document de planification (sprint, roadmap) avant que l'ADR ne soit effectivement rédigé.
- Un ADR accepté **ne se modifie pas** sur le fond. Si la décision change, on écrit un nouvel ADR qui remplace ou amende le précédent.
- Statuts possibles : `proposé` | `accepté` | `déprécié` | `remplacé par ADR-YYY`.
- Modèle : [`template.md`](template.md).
- Logique générale : voir [`docs/process.md`](../process.md) section 2.1.

---

## Vue chronologique

| # | Titre | Sprint | Date | Statut |
|---|---|---|---|---|
| [001](001-architecture-deux-sites-vpn.md) | Architecture deux sites avec VPN site-à-site | Sprint 0 | 28/04/2026 | accepté |
| [002](002-stack-iac-ansible-seul-phase1.md) | Stack IaC : Ansible seul en phase 1 | Sprint 0 | 28/04/2026 | accepté |
| [003](003-plan-adressage.md) | Plan d'adressage IP | Sprint 0 | 28/04/2026 | accepté |
| [004](004-choix-hyperviseur-phase-1.md) | Hyperviseur de phase 1 : VMware Workstation Pro sur PC perso | Sprint 0 | 28/04/2026 | **remplacé par ADR-007** |
| [005](005-segmentation-vlan-et-flux.md) | Segmentation par VLANs taggés et matrice de flux inter-VLAN | Sprint 1 | 07/05/2026 | proposé |
| [006](006-procedures-vs-runbooks.md) | Séparation procédures d'installation vs runbooks de remédiation | Sprint 1 | 30/04/2026 | proposé |
| [007](007-passage-hyperv.md) | Passage à Hyper-V comme hyperviseur de phase 1 | Sprint 1 | 07/05/2026 | proposé |
| [008](008-backend-dhcp-kea.md) | Choix du backend DHCP : Kea (au lieu d'ISC) | Sprint 1 | 07/05/2026 | proposé |

---

## Vue thématique

### Architecture & topologie

- [ADR-001](001-architecture-deux-sites-vpn.md) — Architecture deux sites avec VPN site-à-site
- [ADR-005](005-segmentation-vlan-et-flux.md) — Segmentation par VLANs taggés et matrice de flux inter-VLAN

### Adressage & DNS

- [ADR-003](003-plan-adressage.md) — Plan d'adressage IP

### Outils & stack

- [ADR-002](002-stack-iac-ansible-seul-phase1.md) — Stack IaC : Ansible seul en phase 1
- [ADR-004](004-choix-hyperviseur-phase-1.md) — Hyperviseur de phase 1 : VMware Workstation Pro sur PC perso (**remplacé par ADR-007**)
- [ADR-007](007-passage-hyperv.md) — Passage à Hyper-V comme hyperviseur de phase 1 (remplace ADR-004)
- [ADR-008](008-backend-dhcp-kea.md) — Choix du backend DHCP : Kea (au lieu d'ISC)

### Sécurité & PKI

*(aucun ADR pour l'instant — décision step-ca repoussée après Sprint 2)*

### Sauvegarde & continuité

*(aucun ADR pour l'instant)*

### Supervision & observabilité

*(aucun ADR pour l'instant)*

### Process & gouvernance

- [ADR-006](006-procedures-vs-runbooks.md) — Séparation procédures d'installation vs runbooks de remédiation

---

## Graphe des liens entre ADR

> Représentation simplifiée des relations « remplace », « amende », « dépend de ».

```
ADR-001 ── structure tout le projet (architecture)
        │
        ├── influence ADR-003 (plan d'adressage adapté à 2 sites)
        ├── influence ADR-004 (hyperviseur capable de virtualiser pfSense)
        ├── influence ADR-005 (segmentation VLAN inter-sites)
        └── influence (futur) ADR sur WireGuard site-à-site

ADR-002 ── structure tout le travail IaC du projet
        │
        └── influence (futur) ADR sur Terraform / Packer si ajout en phase 2

ADR-003 ── détaille les sous-réseaux
        │
        └── dépend de ADR-001

ADR-004 ── (déprécié, remplacé par ADR-007)

ADR-005 ── matrice de flux inter-VLAN
        │
        ├── dépend de ADR-001 (architecture 2 sites)
        ├── dépend de ADR-003 (plan d'adressage)
        └── dépend de ADR-007 (Hyper-V comme hyperviseur)

ADR-006 ── structure la documentation opérationnelle
        │
        └── influence process.md (sections 2.2 et 2.3)

ADR-007 ── Hyper-V (remplace ADR-004)
        │
        └── influence ADR-005 (config trunk VLAN spécifique Hyper-V)

ADR-008 ── backend DHCP Kea
        │
        └── influence configuration DHCP par interface (Sprint 1+)
```

⚠️ Ce graphe est **maintenu manuellement** à chaque évolution. À actualiser quand un ADR remplace, amende, ou est référencé par un autre.

---

## ADR dépréciés ou remplacés

| # | Titre | Remplacé par | Date |
|---|---|---|---|
| [004](004-choix-hyperviseur-phase-1.md) | Hyperviseur de phase 1 : VMware Workstation Pro sur PC perso | [ADR-007](007-passage-hyperv.md) | 07/05/2026 |

---

## Comment ajouter un nouvel ADR

1. Copier [`template.md`](template.md) en `<NNN>-<titre-court-en-kebab-case>.md`.
2. Numéro suivant disponible, **ou** numéro précédemment réservé dans une planification (cf. note sur la convention en haut).
3. Remplir tous les champs du header (statut, date, sprint).
4. Statut initial : `proposé`. Devient `accepté` après review et merge de la PR.
5. **Mettre à jour ce fichier** :
   - Ligne dans la vue chronologique
   - Ligne dans la vue thématique (créer une catégorie si besoin)
   - Mise à jour du graphe si pertinent
6. Si l'ADR remplace ou amende un précédent : modifier le statut de l'ancien (`remplacé par ADR-NNN`) dans son fichier ET dans cet index.

---

## Outillage automatique éventuel

⚠️ Cet index est tenu **à la main** pour l'instant. Discussion en cours sur l'opportunité d'utiliser un outil ou un script pour générer automatiquement la vue chronologique et le graphe à partir des en-têtes des fichiers ADR. Si décision prise, elle fera l'objet d'un ADR dédié.

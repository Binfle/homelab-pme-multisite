# Index des ADR

> Liste des Architecture Decision Records du projet, classés et reliés. Mis à jour à chaque ajout ou changement de statut d'un ADR.

## Convention

- Numérotation **séquentielle simple** : ADR-001, ADR-002, …, ADR-099, ADR-100.
- Un ADR accepté **ne se modifie pas** sur le fond. Si la décision change, on écrit un nouvel ADR qui remplace ou amende le précédent.
- Statuts possibles : `proposé` | `accepté` | `déprécié` | `remplacé par ADR-YYY`.
- Modèle : [`template.md`](template.md).
- Logique générale : voir [`docs/process.md`](../process.md) section 2.1.

---

## Vue chronologique

| # | Titre | Sprint | Date | Statut |
|---|---|---|---|---|
| [001](001-architecture-deux-sites-vpn.md) | Architecture deux sites avec VPN site-à-site | Sprint 0 | <à remplir> | accepté |
| [002](002-stack-iac-ansible-seul-phase1.md) | Stack IaC : Ansible seul en phase 1 | Sprint 0 | <à remplir> | accepté |
| [003](003-plan-adressage.md) | Plan d'adressage IP | Sprint 0 | <à remplir> | accepté |
| [004](004-choix-hyperviseur-phase-1.md) | Hyperviseur de phase 1 : VirtualBox sur PC perso | Sprint 0 | <à remplir> | accepté |

---

## Vue thématique

### Architecture & topologie

- [ADR-001](001-architecture-deux-sites-vpn.md) — Architecture deux sites avec VPN site-à-site

### Adressage & DNS

- [ADR-003](003-plan-adressage.md) — Plan d'adressage IP

### Outils & stack

- [ADR-002](002-stack-iac-ansible-seul-phase1.md) — Stack IaC : Ansible seul en phase 1
- [ADR-004](004-choix-hyperviseur-phase-1.md) — Hyperviseur de phase 1 : VirtualBox sur PC perso

### Sécurité & PKI

*(aucun ADR pour l'instant — décision step-ca repoussée après Sprint 2)*

### Sauvegarde & continuité

*(aucun ADR pour l'instant)*

### Supervision & observabilité

*(aucun ADR pour l'instant)*

### Process & gouvernance

*(aucun ADR pour l'instant — la convention de nommage des ADR est définie ici-même, pas dans un ADR dédié)*

---

## Graphe des liens entre ADR

> Représentation simplifiée des relations « remplace », « amende », « dépend de ».

```
ADR-001 ── structure tout le projet (architecture)
   │
   ├── influence ADR-003 (plan d'adressage adapté à 2 sites)
   ├── influence ADR-004 (hyperviseur capable de virtualiser pfSense)
   └── influence (futur) ADR-005 (config WireGuard site-à-site)

ADR-002 ── structure tout le travail IaC du projet
   │
   └── influence (futur) ADR sur Terraform / Packer si ajout en phase 2

ADR-003 ── détaille les sous-réseaux
   │
   └── dépend de ADR-001

ADR-004 ── choix d'outil de virtualisation
   │
   └── à reconsidérer pour phase 2 (Proxmox sur mini PC envisagé)
```

⚠️ Ce graphe est **maintenu manuellement** à chaque évolution. À actualiser quand un ADR remplace, amende, ou est référencé par un autre.

---

## ADR dépréciés ou remplacés

*(aucun pour l'instant — section qui se remplira au fil du projet)*

| # | Titre | Remplacé par | Date |
|---|---|---|---|
| – | – | – | – |

---

## Comment ajouter un nouvel ADR

1. Copier [`template.md`](template.md) en `<NNN>-<titre-court-en-kebab-case>.md`.
2. Numéro suivant disponible (003 → 004 → 005 …).
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

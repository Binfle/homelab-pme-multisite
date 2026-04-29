# CR — Clôture Sprint 0 (Cadrage)

**Date** : 2026-04-29
**Participants** : Membre 1, Membre 2
**Format** : Stream Discord (visio + partage écran)
**Durée** : ~15 min

---

## Bilan du sprint

### Objectif initial du Sprint 0

Poser le **cadre méthodologique et technique** du projet `homelab-pme-multisite` : architecture cible, choix de stack, plan d'adressage, méthode de travail, structure du repo et premier schéma réseau.

### Livrables produits

#### Documentation fondatrice
- `README.md` — présentation du projet et fiction ACME Corp
- `CONTRIBUTING.md` — règles de contribution et conventions Git
- `LICENSE` — MIT
- `.gitignore` — exclusions standards
- `docs/architecture.md` — vue d'ensemble de l'archi
- `docs/addressing-plan.md` — plan d'adressage IP complet (siege01, agence01, transit VPN, asymétrie justifiée, IPs réservées Phase 1)
- `docs/process.md` — cadre méthodologique (vocabulaire, ADR, runbooks, CR, workflow Git, livrables, cadence)

#### ADR (Architecture Decision Records)
- ADR-001 : Architecture deux sites avec VPN site-à-site
- ADR-002 : Stack IaC — Ansible seul en Phase 1
- ADR-003 : Plan d'adressage IP
- ADR-004 : Choix d'hyperviseur — VMware Workstation Pro sur PC perso
- `docs/adr/README.md` — index des ADR (chronologique + thématique + relations)
- `docs/adr/template.md` — modèle pour les futurs ADR

#### Templates et infrastructure repo
- `docs/runbooks/template.md`
- `docs/meetings/template.md`
- `docs/sprints/sprint-0-cadrage.md` — feuille de route du sprint
- `.github/pull_request_template.md`
- `.github/ISSUE_TEMPLATE/{bug,doc,feature}.md`

#### Schéma réseau
- `diagrams/network-overview.txt` — version ASCII textuelle
- `diagrams/network-overview.png` — schéma Packet Tracer (vue d'ensemble, 2 sites, segmentation VLAN, tunnel WireGuard, légende)

#### Mise en place Git
- Repo initialisé localement
- Premier push sur GitHub effectué
- Branch protection activée sur `main` (PR obligatoire)

---

## Décisions structurantes actées

- **Architecture** : 2 sites distincts, hébergés chez chaque membre, tunnel WireGuard site-à-site via internet
- **Hyperviseur Phase 1** : VMware Workstation Pro sur PC perso (ADR-004)
- **Stack IaC Phase 1** : Ansible seul (ADR-002)
- **Plan d'adressage** : siege01 = 10.10.0.0/16, agence01 = 10.20.0.0/16, transit VPN = 10.99.99.0/30 (ADR-003)
- **Nomenclature** : siege01 / agence01 (numérotation simple, extension prévue)
- **Asymétrie segments** : pas de DMZ ni Wi-Fi guest sur agence01 (justifiée dans `addressing-plan.md`)
- **Segmentation** : VLANs taggés sur trunk 802.1Q (1 switch + sub-interfaces pfSense). ADR-005 à rédiger en début Sprint 1.
- **DNS** : BIND9 conservé sur agence01 (split-horizon avec AD-DNS du siège)
- **Découpage** : Phase 1 = MVP (Sprints 0 à 5), Phase 2 = bonus
- **Spike DMZ + Wi-Fi guest agence01** : reporté en Sprint 3, ADR de retour d'expérience après

---

## Ajustements pour le Sprint 1

- Rédiger l'**ADR-005** sur la segmentation par VLANs taggés 802.1Q dès le début du sprint

---

## Validation des critères "livrable" (Sprint 0)

| 1. Ça marche (cadre opérationnel, repo en place) | ✅ |
| 2. Versionné sur main | ✅ |
| 3. Documenté (README, ADR, addressing-plan, process) | ✅ |
| 4. Testable / lisible par l'autre membre | ✅ |
| 5. Mis en avant (visible dans le README + index ADR) | ✅ |

**Conclusion** : Sprint 0 **clos**. Bascule vers Sprint 1 (grappe réseau bas-niveau).

---

## Prochaine étape

**Sprint 1 — Réseau bas-niveau**
- Configuration pfSense × 2 (WAN, LAN, VLANs)
- Tunnel WireGuard site-à-site
- Premiers rôles Ansible Linux de base
- Rédaction ADR-005 (segmentation VLAN)
- Cible : ~1 à 2 semaines

**Tag Git** : `sprint-0` à appliquer sur le commit final du sprint.
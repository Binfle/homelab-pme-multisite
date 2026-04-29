# ADR-004 — Hyperviseur de phase 1 : VirtualBox sur PC perso

**Statut** : accepté
**Date** : 28/04/2026
**Remplace** : néant
**Remplacé par** : néant

---

## Contexte

Le projet nécessite un hyperviseur sur lequel seront déployées les VM des deux sites (pfSense, AD, postes clients, services applicatifs). Plusieurs options ont été considérées :

- **Proxmox VE** sur un serveur dédié partagé entre les deux membres
- **Proxmox VE** installé sur les PC perso (en remplacement de Windows)
- **Proxmox VE** sur les machines secondaires (mini PC, vieux PC)
- **VirtualBox** sur les PC perso, par-dessus Windows
- **VMware Workstation Pro** sur les PC perso, par-dessus Windows
- **Hyper-V** (Windows Pro) sur les PC perso
- **Dual-boot** Windows + Proxmox sur SSD séparé

Contraintes identifiées :

1. **Pas de serveur partagé disponible** entre les deux membres (déjà acté ADR-001).
2. **Les PC Pero sont les machines les plus puissantes** disponibles (32 Go RAM chacun), mais doivent rester utilisables pour leur usage normal (jeux, perso).
3. **Les machines secondaires sont insuffisantes** ou risquées :
   - Mini PC (Intel N150, 16 Go RAM) : capable mais limité, plafond hardware à 16 Go.
   - Vieux PC (LGA 1156, 16 Go RAM) : CPU exact à confirmer, support VT-x/VT-d incertain, consommation électrique élevée (TDP 95W pour les CPU usuels de cette génération).
4. **Compétences du binôme** sur l'hyperviseur ciblé : Proxmox = découverte, VirtualBox = familier.
5. **Phase 1 = MVP**. Pas de besoin de cluster, HA, migration à chaud, ZFS avancé pour le périmètre prévu.
6. **Pas de problème CGNAT** confirmé chez les deux membres → faisabilité du WireGuard site-à-site OK quel que soit l'hyperviseur.

---

## Décision

**Phase 1** : utiliser **VirtualBox** sur chaque PC perso, par-dessus Windows, lancé uniquement pendant les sessions de travail sur le projet.

Configuration réseau VirtualBox :

- **WAN pfSense** : Bridged Adapter sur le LAN domestique, port forwarding WireGuard sur la box internet.
- **LAN / DMZ / MGMT internes** : Internal Networks VirtualBox nommés selon le plan d'adressage (`siege-lan-usr`, `siege-lan-srv`, `siege-dmz`, `agence-lan-usr`, `agence-lan-srv`, `agence-mgmt`).

VMware Workstation Pro est une **alternative acceptable** pour qui le préfère (gratuit pour usage personnel, à vérifier sur le site officiel au moment de l'installation). Le choix exact entre VirtualBox et VMware Workstation peut différer entre les deux membres sans conséquence pour le projet, tant que le réseau virtuel est configuré équivalent.

**Phase 2 (bonus)** : Proxmox VE sera déployé sur le mini PC pour héberger les services permanents (NetBox, Grafana, Restic), accessibles 24/7 indépendamment des sessions de travail. Cette phase est explicitement **hors MVP**.

---

## Conséquences

### Positives

- **Aucune perturbation de l'usage personnel** des PC perso (Windows reste l'OS principal).
- **Lancement à la demande** : VirtualBox démarre quand on travaille, s'arrête quand on a fini.
- **Compétence existante** : VirtualBox est familier, pas de courbe d'apprentissage sur l'hyperviseur lui-même.
- **Performances suffisantes** : 32 Go de RAM par PC permettent d'héberger toutes les VM d'un site sans problème.
- **Réversibilité** : si un membre veut migrer plus tard vers Proxmox sur sa machine, c'est possible sans casser le projet (VM exportables au format OVF).
- **Apprentissage Proxmox préservé** en phase 2, sur la machine adaptée (mini PC).

### Négatives

- **Pas de cluster** : impossible de pratiquer la migration à chaud, le HA, le stockage partagé. Reporté en phase 2 ou hors projet.
- **Pas d'apprentissage Proxmox en phase 1** : le binôme ne découvre pas Proxmox VE pendant le MVP (compensé en phase 2).
- **Quand VirtualBox est éteint, le site correspondant disparaît** du réseau cross-sites. À documenter dans le runbook "indisponibilité de site".
- **Performance VirtualBox < KVM/Proxmox** sur des charges lourdes. Acceptable pour le périmètre MVP, à surveiller pour les charges Docker.
- **Sauvegarde des VM** : les fichiers `.vdi` peuvent être volumineux (10-50 Go par VM). Prévoir gestion de l'espace SSD.

### Risques

- **VirtualBox et Hyper-V incompatibles** sur Windows : si Hyper-V est activé, VirtualBox peut refuser de fonctionner. À vérifier dès l'installation.
- **VT-x/AMD-V** doit être activé dans le BIOS — à vérifier en prérequis Sprint 0.
- **Espace disque** sur SSD à surveiller : prévoir au moins 100 Go libres par PC pour les VM.

---

## Alternatives écartées

### Proxmox VE sur serveur partagé
- **Écartée** : pas de serveur disponible (rappel ADR-001).
- Coût (location dédié cloud) jugé non justifié en phase 1.

### Proxmox VE sur PC perso (en remplacement de Windows)
- **Écartée** : transforme le PC perso en serveur pur, casse l'usage personnel.
- Le dual-boot atténue le problème mais demande un reboot à chaque changement d'usage, peu pratique.

### Proxmox VE sur mini PC (en phase 1)
- **Écartée pour la phase 1** : 16 Go de RAM (plafond hardware du N150) insuffisant pour héberger toutes les VM d'un site (pfSense + AD + Docker + clients).
- **Reportée en phase 2** sur un sous-ensemble de services permanents.

### Proxmox VE sur vieux PC
- **Écartée** : incertitudes sur le support VT-d, consommation électrique élevée (TDP CPU LGA 1156 typiquement 95W), perfs limitées pour des VM Windows.
- Risque de perte de temps de débogage matériel disproportionné au gain.

### Dual-boot Windows + Proxmox sur SSD séparé
- **Écartée** : oblige à rebooter pour basculer entre usage perso et projet.
- L'infra n'est pas active en permanence (mêmes contraintes que VirtualBox sans le confort du lancement à la demande).
- Coût d'un SSD supplémentaire (~40 €) non justifié face à VirtualBox.

### Hyper-V (Windows Pro)
- **Non écartée formellement, mais non choisie**.
- Demande Windows Pro (pas garanti d'être présent sur les PC perso).
- Cohabitation difficile avec VirtualBox.
- Les deux membres ont moins d'expérience.

### VMware Workstation Pro
- **Acceptée comme alternative équivalente à VirtualBox**, au choix de chaque membre.
- Souvent plus performant, désormais gratuit pour usage personnel *(à vérifier sur le site officiel VMware au moment de l'installation, conditions susceptibles d'évoluer)*.

---

## Conditions de révision

Cet ADR sera révisé si :

- VirtualBox révèle des limitations bloquantes pendant le Sprint 1 (réseau, performances, stabilité).
- Le binôme acquiert un serveur dédié partagé (don, occasion, budget).
- La phase 2 démarre et que le périmètre Proxmox dépasse "services permanents sur mini PC".

Le cas échéant, un ADR-XXX remplaçant celui-ci sera rédigé.

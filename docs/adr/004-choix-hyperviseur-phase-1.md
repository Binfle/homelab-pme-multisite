# ADR-004 — Hyperviseur de phase 1 : VMware Workstation Pro sur PC perso

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
2. **Les PC perso sont les machines les plus puissantes** disponibles (32 Go RAM chacun), mais doivent rester utilisables pour leur usage normal (jeux, perso).
3. **Les machines secondaires sont insuffisantes** ou risquées :
   - Mini PC (Intel N150, 16 Go RAM) : capable mais limité, plafond hardware à 16 Go.
   - Vieux PC (LGA 1156, 16 Go RAM) : CPU exact à confirmer, support VT-x/VT-d incertain, consommation électrique élevée (TDP 95W pour les CPU usuels de cette génération).
4. **Compétences du binôme** sur l'hyperviseur ciblé : Proxmox = découverte, VirtualBox et VMware = familiers à divers degrés.
5. **Phase 1 = MVP**. Pas de besoin de cluster, HA, migration à chaud, ZFS avancé pour le périmètre prévu.
6. **Pas de problème CGNAT** confirmé chez les deux membres → faisabilité du WireGuard site-à-site OK quel que soit l'hyperviseur.
7. **VMware Workstation Pro est devenu gratuit** pour usages commercial, éducatif et personnel depuis novembre 2024 (annonce Broadcom du 11/11/2024), supprimant l'argument coût en faveur de VirtualBox.

---

## Décision

**Phase 1** : utiliser **VMware Workstation Pro** sur chaque PC perso, par-dessus Windows, lancé uniquement pendant les sessions de travail sur le projet.

**VirtualBox reste accepté** comme alternative individuelle pour qui le préfère. Le format des VM (OVF) permet d'exporter/importer entre les deux hyperviseurs si nécessaire, tant que la configuration réseau virtuelle est équivalente.

Configuration réseau VMware (ou VirtualBox équivalent) :

- **WAN pfSense** : Bridged Adapter sur le LAN domestique, port forwarding WireGuard sur la box internet.
- **LAN / DMZ / MGMT internes** : réseaux internes nommés selon le plan d'adressage (`siege-lan-usr`, `siege-lan-srv`, `siege-dmz`, `siege-mgmt`, `agence-lan-usr`, `agence-lan-srv`, `agence-mgmt`).

**Phase 2 (bonus)** : Proxmox VE sera déployé sur le mini PC pour héberger les services permanents (NetBox, Grafana, Restic), accessibles 24/7 indépendamment des sessions de travail. Cette phase est explicitement **hors MVP**.

---

## Conséquences

### Positives

- **Aucune perturbation de l'usage personnel** des PC perso (Windows reste l'OS principal).
- **Lancement à la demande** : l'hyperviseur démarre quand on travaille, s'arrête quand on a fini.
- **VMware Workstation Pro** offre de meilleures performances et une nested virtualization plus mature que VirtualBox, utiles pour les VM lourdes (Windows Server, Docker host).
- **Performances suffisantes** : 32 Go de RAM par PC permettent d'héberger toutes les VM d'un site sans problème.
- **Compatibilité préservée** : un membre peut choisir VirtualBox sans casser le projet, le format OVF assure l'interopérabilité.
- **Apprentissage Proxmox préservé** en phase 2, sur la machine adaptée (mini PC).

### Négatives

- **Pas de cluster** : impossible de pratiquer la migration à chaud, le HA, le stockage partagé. Reporté en phase 2 ou hors projet.
- **Pas d'apprentissage Proxmox en phase 1** : le binôme ne découvre pas Proxmox VE pendant le MVP (compensé en phase 2).
- **Quand l'hyperviseur est éteint, le site correspondant disparaît** du réseau cross-sites. À documenter dans le runbook "indisponibilité de site".
- **Sauvegarde des VM** : les fichiers `.vmdk` peuvent être volumineux (10-50 Go par VM). Prévoir gestion de l'espace SSD.
- **Compte Broadcom requis** pour télécharger VMware Workstation Pro officiellement (gratuit mais étape d'inscription).

### Risques

- **VMware/VirtualBox et Hyper-V incompatibles** sur Windows : si Hyper-V est activé, ces hyperviseurs peuvent refuser de fonctionner. À vérifier dès l'installation.
- **VT-x/AMD-V** doit être activé dans le BIOS — à vérifier en prérequis Sprint 0.
- **Espace disque** sur SSD à surveiller : prévoir au moins 100 Go libres par PC pour les VM.
- **Si l'un utilise VirtualBox et l'autre VMware**, l'export/import de VM via OVF doit être testé tôt pour éviter les surprises de compatibilité.

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
- L'infra n'est pas active en permanence (mêmes contraintes que VMware sans le confort du lancement à la demande).
- Coût d'un SSD supplémentaire (~40 €) non justifié.

### VirtualBox comme techno principale
- **Non écartée définitivement** : reste accepté comme alternative individuelle.
- VMware Workstation Pro a été préféré comme techno **principale** pour ses meilleures performances, sa gestion plus mature des snapshots, et sa nested virtualization plus stable.
- L'argument historique du coût (VMware payant vs VirtualBox gratuit) ne tient plus depuis novembre 2024.

### Hyper-V (Windows Pro)
- **Écartée** : nécessite Windows Pro (pas garanti d'être présent sur les PC perso).
- Cohabitation difficile avec VMware/VirtualBox.
- Compétences moins répandues dans le binôme.

---

## Conditions de révision

Cet ADR sera révisé si :

- VMware Workstation Pro révèle des limitations bloquantes pendant le Sprint 1 (réseau, performances, stabilité).
- Broadcom modifie les conditions de gratuité de VMware Workstation Pro (à surveiller).
- Le binôme acquiert un serveur dédié partagé (don, occasion, budget).
- La phase 2 démarre et que le périmètre Proxmox dépasse "services permanents sur mini PC".

Le cas échéant, un ADR-XXX remplaçant celui-ci sera rédigé.

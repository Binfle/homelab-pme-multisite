# ADR-007 — Passage à Hyper-V comme hyperviseur de phase 1

**Statut** : proposé
**Date** : 07/05/2026
**Remplace** : ADR-004
**Remplacé par** : néant

---

## Contexte

L'ADR-004 (28/04/2026) actait **VMware Workstation Pro** comme hyperviseur principal de la phase 1, sur les PC perso des deux membres du binôme (par-dessus Windows 11 Pro).

En pratique, lors de la mise en œuvre du Sprint 1, le binôme a basculé sur **Hyper-V** (composant natif de Windows 11 Pro) sur les deux sites. Cet ADR formalise et trace cette décision, et déprécie ADR-004 en remplacement.

### Motivations

La motivation principale du basculement est technique et concrète : **Hyper-V s'est révélé être le seul hyperviseur disponible sur Windows 11 Pro à offrir un support natif et scriptable des trunks VLAN 802.1Q**.

Concrètement :

- **Hyper-V** : la commande PowerShell `Set-VMNetworkAdapterVlan -Trunk -AllowedVlanIdList "10,20,30,40"` configure un trunk VLAN en une ligne, scriptable, reproductible, intégrée à Windows.
- **VMware Workstation Pro** : pas d'équivalent direct côté UI ou CLI sur Workstation. Le tagging VLAN existe sur ESXi mais pas de la même manière sur la version Workstation utilisée en homelab Windows.
- **VirtualBox** : support VLAN limité, pas de mécanisme trunk natif équivalent.

Comme la segmentation VLAN est au cœur du Sprint 1 (cf ADR-005), l'hyperviseur qui la supporte le mieux a logiquement été choisi.

Motivations secondaires :

- **Hyper-V est natif Windows 11 Pro** : pas d'install supplémentaire, pas de compte tiers (vs VMware qui demande un compte Broadcom).
- **Niveau d'apprentissage acceptable** : Hyper-V est documenté, utilisé en entreprise (notamment dans les PME Microsoft-centrées qui sont la cible métier du projet ACME Corp).
- **Cohabitation avec d'autres usages Hyper-V** (Docker Desktop, WSL2) déjà fréquents sur un PC de dev.

---

## Décision

**Phase 1** : utiliser **Hyper-V** (Windows 11 Pro) sur chaque PC perso, comme composant natif de Windows. Lancement des VMs à la demande pendant les sessions de travail.

### Configuration réseau Hyper-V

Pour chaque site (siege01 et agence01) :

- **vSwitch externe** dédié au projet, attaché à la carte physique de l'hôte (équivalent Bridged en VMware) → utilisé pour le port WAN de pfSense.
- **vSwitch interne** ou **privé** dédié au projet (selon besoin de connectivité hôte) → utilisé pour le port LAN de pfSense, configuré en **trunk** côté VM avec les VLANs alloués.
- **Trunk VLAN** côté VM via PowerShell :

```powershell
Set-VMNetworkAdapterVlan -VMName "pfsense-<site>" `
  -VMNetworkAdapterName "LAN" `
  -Trunk -AllowedVlanIdList "10,20,30,40" -NativeVlanId 0
```

- **Postes admin Windows hôte** : carte vEthernet (du vSwitch interne) configurée en **Access VLAN 40** pour accéder à MGMT :

```powershell
Set-VMNetworkAdapterVlan -ManagementOS `
  -VMNetworkAdapterName "<nom_vSwitch>" `
  -Access -VlanId 40
```

### Phase 2 (bonus, non MVP)

**Inchangé par rapport à ADR-004** : Proxmox VE sera déployé sur le mini PC en phase 2 pour héberger les services permanents (NetBox, Grafana, Restic). Hors MVP.

---

## Conséquences

### Positives

- **VLAN trunk natif via PowerShell** : config scriptable, reproductible, traçable dans le repo.
- **Documentation Microsoft riche** sur les scénarios Hyper-V dans Windows 11 Pro et Server.
- **Cohabitation OK avec d'autres usages Hyper-V** (Docker Desktop, WSL2) qui sont fréquents sur un PC de dev.
- **Intégration Windows native** : snapshots et gestion VMs depuis le Gestionnaire Hyper-V.

### Négatives

- **Hyper-V exclusif** : impossible de faire tourner VMware Workstation Pro ou VirtualBox simultanément avec Hyper-V activé (limitation Windows). Si un membre voulait utiliser un autre hyperviseur en parallèle, il devrait désactiver Hyper-V (et perdre l'accès au lab).
- **Nécessite Windows Pro** (vérifié OK chez les deux membres). Une éventuelle bascule sur Windows Famille casserait Hyper-V.
- **Convention de configuration différente de VMware** : apprentissage spécifique à refaire pour les vSwitch, le tagging VLAN, le ManagementOS, etc.
- **Moins de tutoriels homelab** sur Hyper-V que sur VMware/VirtualBox dans la communauté homelab francophone et anglophone — tendance à devoir creuser dans la doc Microsoft officielle.
- **Comportement particulier du toggle Untag/Tag** : observation Sprint 1, après modification réseau majeure côté pfSense, le tunnel Hyper-V ManagementOS peut nécessiter un toggle Untagged ↔ Access pour réinitialiser le driver vSwitch. Cause exacte non investiguée. À documenter dans un runbook de troubleshooting.

### Risques

- **VT-x / AMD-V désactivés dans le BIOS** : Hyper-V refuse de démarrer si la virtualisation matérielle n'est pas activée. Pré-requis à valider à l'install.
- **Mise à jour Windows cassante** : une mise à jour Windows majeure peut perturber Hyper-V (rare mais documenté). Sauvegardes des VMs critiques recommandées avant les Patch Tuesday.
- **Espace SSD** : les fichiers VHDX peuvent être volumineux. Prévoir au moins 100 Go libres par PC (cohérent avec ce qui était prévu dans ADR-004 pour VMware).

---

## Alternatives écartées

### VMware Workstation Pro (ADR-004 initial)

- **Écartée** : pas de support trunk VLAN natif scriptable comme Hyper-V.
- Compte Broadcom requis pour le téléchargement (friction supplémentaire).
- Cohabitation difficile avec d'autres usages Hyper-V (Docker Desktop, WSL2).

### VirtualBox

- **Écartée** : support VLAN limité, pas de mécanisme trunk natif équivalent à Hyper-V.
- Performance et stabilité réseau inférieures dans le contexte d'un lab pfSense + multiples VMs.
- Même problème de cohabitation avec Hyper-V que VMware.

### Proxmox VE en bare-metal sur PC perso

- **Écartée pour la phase 1** : transformerait le PC perso en serveur pur, casserait l'usage personnel.
- Cohérent avec la décision déjà actée dans ADR-004.

### Proxmox VE sur mini PC en phase 1

- **Écartée pour la phase 1** : 16 Go RAM (plafond hardware du N150) insuffisants pour héberger toutes les VMs d'un site.
- **Reportée en phase 2** sur un sous-ensemble de services permanents — inchangé.

---

## Migration depuis VMware (si applicable)

Si des VMs avaient été créées sous VMware avant la bascule, deux options de migration :

1. **Recréer les VMs from scratch** sous Hyper-V (recommandé pour pfSense, où la config est faite via webGUI / config.xml).
2. **Convertir les VMs VMware (.vmdk) en Hyper-V (.vhdx)** via l'outil `Microsoft Virtual Machine Converter` ou `qemu-img` — utile si des VMs Linux avaient déjà été configurées.

Dans le cas du Sprint 1 actuel, pfSense agence01 a été installé directement sous Hyper-V — pas de migration à effectuer.

---

## Conditions de révision

Cet ADR sera révisé si :

- Hyper-V révèle des limitations bloquantes pendant les Sprints 2-5 (réseau, performances, stabilité).
- Microsoft modifie les conditions d'inclusion d'Hyper-V dans Windows 11 Pro (très improbable).
- La phase 2 démarre et impose des choix d'hyperviseur différents (Proxmox VE pour les services permanents).
- Le binôme acquiert un serveur dédié partagé (don, occasion, budget) qui justifierait un Proxmox cluster.

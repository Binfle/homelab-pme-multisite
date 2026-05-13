# ADR-011 — Stratégie d'attribution d'IP par VLAN : classification fonctionnelle + DHCP réservé

**Statut** : proposé
**Date** : 12/05/2026
**Remplace** : néant
**Remplacé par** : néant
**Amende** : ADR-008 (backend DHCP Kea)

---

## Contexte

À l'issue du Sprint 1, deux dettes méthodologiques liées à l'adressage IP ont été identifiées :

1. **Classification floue dans les plages statiques.** `conventions.md` § 3 définit des plages très larges (`.2-99` pour les IPs statiques) sans sous-classification fonctionnelle. À l'usage, on se rend compte qu'il serait utile de réserver des sous-plages par type de machine (équipements réseau, outils d'admin, observabilité, etc.) pour faciliter la lecture du plan d'adressage et la cohérence inter-sites.

2. **ADR-008 dit « Pas de DHCP sur MGMT »**, avec la justification « les administrateurs configurent leur poste manuellement ». Cette justification supposait un usage **humain** de MGMT. Or, depuis l'introduction du contrôleur Ansible en MGMT (Sprint 1), on a une **machine** sur ce VLAN, et la question se pose à nouveau : faut-il rester en statique manuel ou activer un DHCP avec réservations strictes pour centraliser l'inventaire ?

Le binôme a discuté plusieurs alternatives au cours du Sprint 1 :
- Tout statique manuel partout (sécurisé mais lourd à maintenir)
- DHCP partout (cohérent mais fragile pour les machines critiques)
- Fallback automatique DHCP → statique (complexe et fragile sur Debian standard)
- Hybride par criticité (la voie retenue)

L'objectif de cet ADR est de **figer une stratégie cohérente** applicable à tous les VLANs du projet, et qui couvre à la fois la classification fonctionnelle des plages d'adresses et le mode d'attribution (statique manuel vs DHCP réservé).

---

## Décision

### 1. Classification fonctionnelle par dizaines

Sur les plages /24 utilisées par le projet, les IPs sont classifiées en **sous-plages fonctionnelles par dizaines** :

| Plage | Famille fonctionnelle | Stratégie d'attribution |
|---|---|---|
| `.0` | Adresse réseau | — |
| `.1` | Gateway (pfSense) | Statique manuel |
| `.2-9` | Équipements réseau (switches, AP, autres FW) | Statique manuel |
| `.10-29` | Services métier / outils d'administration / annuaire / DNS | Statique manuel ou DHCP réservé selon criticité |
| `.30-49` | Observabilité, conteneurs, applications | DHCP réservé |
| `.50-99` | Réserve statique additionnelle | Statique manuel |
| `.100-200` | Plage DHCP dynamique libre | DHCP libre (uniquement LAN_USR) |
| `.201-254` | Réserve VIPs, équipements spéciaux, redondance (CARP/HSRP) | Statique manuel |
| `.255` | Broadcast | — |

Cette classification s'applique à tous les VLANs (LAN_USR, LAN_SRV, DMZ, MGMT) sur les deux sites (`siege01`, `agence01`). Les usages effectifs par VLAN diffèrent (par exemple, MGMT n'utilise jamais la plage `.100-200` puisque pas de DHCP libre), mais la **convention de découpage reste constante**.

### 2. Stratégie d'attribution selon la criticité

| Catégorie de machine | Stratégie | Exemples |
|---|---|---|
| **Critique service-métier** | Statique manuel (jamais DHCP) | pfSense, AD DC1, DNS server primaire |
| **Modéré service-métier** | DHCP réservé avec lease long (7 jours) | Reverse proxy, Docker host, observabilité |
| **Outils d'administration** | DHCP réservé | Contrôleur Ansible, bastion, Vault, AWX |
| **Postes utilisateurs** | DHCP libre | Postes physiques ou VM dans LAN_USR |
| **Équipements réseau physiques** | Statique manuel | Switches managés, AP, IPMI |

**Justification de la division** : les machines dont la disponibilité IP est **vitale** pour le bon fonctionnement du réseau (gateway, identité, DNS) restent en statique manuel — elles doivent avoir leur IP même si le DHCP tombe. Les autres bénéficient du DHCP réservé pour la centralisation de l'inventaire et la facilité de renumbering, avec un lease long pour ne pas perdre l'IP en cas de coupure brève du DHCP.

### 3. Activation de Kea sur tous les VLANs en mode hybride

**Amendement à ADR-008** : Kea est activé sur **tous les VLANs**, y compris MGMT et LAN_SRV. Sur les VLANs sans usage utilisateur (MGMT, LAN_SRV, DMZ) :

- **Pas de plage dynamique libre** déclarée
- **Uniquement des « static mappings »** (réservations par MAC → IP)
- Aucun appareil non listé ne peut obtenir d'IP via DHCP sur ces VLANs

Sur LAN_USR :
- **Plage dynamique** `.100-200` ouverte aux postes utilisateurs
- Réservations également possibles dans `.2-99` ou `.201-254` si besoin

### 4. Mise en œuvre opérationnelle

- **Sur pfSense**, dans `Services > DHCP Server > <VLAN>`, configurer :
  - Range vide pour MGMT/LAN_SRV/DMZ
  - Range `.100-200` pour LAN_USR
  - Static mappings ajoutés au cas par cas
- **Lease time** : 168h (7 jours) pour les VLANs serveurs (MGMT, LAN_SRV, DMZ), 24h pour LAN_USR (postes itinérants)
- **MAC** des VMs Hyper-V à figer via `Set-VMNetworkAdapter -StaticMacAddress` pour garantir que la MAC ne change pas entre reboots (sinon la réservation DHCP ne matche plus)

### 5. Conséquences sur les conventions

`docs/conventions.md` § 3 est mis à jour pour refléter cette classification (sous-plages explicites + stratégie d'attribution).

`docs/addressing-plan.md` reste largement inchangé sur le fond, mais la section "IPs réservées" peut être amendée pour refléter la stratégie (statique vs DHCP réservé) au cas par cas.

---

## Conséquences

### Positives

- **Cohérence lisible** du plan d'adressage : à la simple lecture d'une IP, on sait à quelle famille fonctionnelle elle appartient.
- **Inventaire centralisé dans Kea** : la liste des couples MAC↔IP autorisés vit dans pfSense, plus dans chaque VM. Facile à exporter, sauvegarder, auditer.
- **Renumbering facile** pour les machines en DHCP réservé : changer une IP dans Kea, la VM la prend au prochain renew sans intervention sur la machine elle-même.
- **Résilience des machines critiques préservée** : pfSense, AD DC, DNS primaire ne dépendent pas du DHCP.
- **Compatible avec les pratiques d'entreprise** : ce pattern est largement utilisé en PME multi-sites.
- **Convention extensible** : les sous-plages par dizaines laissent de la place pour les VLANs futurs (Wi-Fi corp/guest, IoT, etc.).

### Négatives

- **Activation de Kea sur MGMT et LAN_SRV** alors qu'ADR-008 disait l'inverse pour MGMT. Cet ADR amende ADR-008 sur ce point, à tracer proprement.
- **Complexité opérationnelle accrue** : il faut gérer les MAC des VMs (figer côté Hyper-V, déclarer côté Kea). Plus de boutons à cliquer à l'ajout d'une nouvelle VM.
- **Dépendance à Kea** pour les machines modérées et les outils d'admin : si Kea tombe pour une durée supérieure au lease, ces machines perdent leur IP. Mitigation : lease long (7 jours).
- **Migration nécessaire** des machines déjà déployées qui ne respectent pas la classification (notamment le contrôleur Ansible actuellement à `.100`, doit migrer à `.10`).

### Risques

- **Drift MAC ↔ IP** : si une VM est recréée avec une MAC différente, sa réservation DHCP ne matche plus → pas d'IP. Mitigation : `Set-VMNetworkAdapter -StaticMacAddress` systématique à la création.
- **Oubli de réservation pour une nouvelle VM** : sans plage libre sur MGMT/LAN_SRV/DMZ, une VM nouvellement créée ne reçoit pas d'IP tant que sa MAC n'est pas déclarée dans Kea. C'est aussi un avantage (contrôle d'accès strict) mais peut surprendre à l'usage.
- **Stratégie complexe à expliquer** à un nouveau venu : 5 catégories de machines, 4 plages fonctionnelles, 2 modes d'attribution. À documenter clairement dans `conventions.md`.

---

## Alternatives écartées

### Maintenir « Pas de DHCP sur MGMT » (ADR-008 inchangé)

- Tout est statique manuel sur MGMT.
- Écarté car : pas d'inventaire centralisé, doit éditer chaque VM pour changer son IP, pas extensible quand le nombre d'outils d'admin grandira (Sprint 4+ : monitoring, sauvegarde, etc.).

### Tout DHCP réservé partout (y compris pfSense, AD DC)

- Centralisation maximale.
- Écarté pour la résilience : les machines vitales (gateway, identité) ne doivent pas dépendre du DHCP. Si Kea tombe pour une longue durée, ces machines perdent leur IP → panne en cascade.

### Tout statique manuel partout

- Aucune dépendance DHCP.
- Écarté car : pas d'inventaire centralisé, lourdeur à l'usage, pas extensible.

### Fallback automatique DHCP → statique sur chaque machine

- Discuté en Sprint 1.
- Écarté car : aucune solution propre et standard sur Debian ifupdown, scripts maison fragiles, fausse sécurité (la double IP simultanée n'est pas un vrai fallback).

### Classification par incrément de 5 (au lieu de dizaines)

- Sous-plages plus fines.
- Écarté car : trop fin pour le périmètre actuel, complique la mémorisation. Les dizaines sont un compromis lisibilité/granularité standard.

---

## Conditions de révision

Cet ADR sera révisé si :

- Kea révèle des limitations bloquantes sur l'usage en static-mappings-only sur certains VLANs.
- Le nombre de réservations devient ingérable manuellement (passage à NetBox + outillage automatique).
- Une mise en place de HA Kea (deux serveurs en failover) modifie le risque "dépendance DHCP".
- Le binôme s'aperçoit qu'une catégorie de machine n'est pas couverte par les 5 catégories actuelles.
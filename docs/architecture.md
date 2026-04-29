# Architecture

> Description en prose de l'architecture cible. Complète le schéma [`diagrams/network-overview.txt`](../diagrams/network-overview.txt) et le plan d'adressage [`addressing-plan.md`](addressing-plan.md).

---

## 1. Vue d'ensemble

L'infrastructure modélise une PME fictive, **ACME Corp**, organisée sur **deux sites distincts** :

- **Siège (Site A)** : héberge la majorité des services centraux (annuaire, supervision, applications métier).
- **Agence (Site B)** : antenne régionale, autonome sur certains services (DNS, IPAM) mais dépendante du siège pour l'authentification.

Les deux sites sont reliés par un **tunnel WireGuard site-à-site** établi entre leurs pare-feux pfSense respectifs. Ce tunnel transporte tout le trafic inter-sites et constitue l'unique lien privé entre les deux LAN.

Du point de vue des hébergeurs, chaque site est virtualisé sur un **PC physique distinct**, situé chez un membre du binôme. Le tunnel WireGuard passe donc à travers internet via les box des deux foyers — c'est un tunnel **réel**, pas une simulation.

---

## 2. Segmentation réseau

Chaque site est segmenté en plusieurs sous-réseaux distincts (matérialisés en VLANs ou en Internal Networks VirtualBox selon la phase). La séparation respecte le principe de **zones de confiance décroissantes** :

### Site A (Siège)

| Zone | Rôle | Accès depuis internet |
|---|---|---|
| **LAN-USR** | Postes utilisateurs (admin, commerce, R&D) | Non (sortant uniquement) |
| **LAN-SRV** | Serveurs internes (AD, monitoring, file share) | Non |
| **DMZ** | Services exposés (site web, reverse proxy) | Oui (entrant filtré) |
| **MGMT** | Administration out-of-band (consoles, IPMI, switches) | Non |
| **WiFi corp** | Postes mobiles authentifiés | Sortant uniquement |
| **WiFi guest** | Invités, isolé du SI | Sortant uniquement |

### Site B (Agence)

| Zone | Rôle | Accès depuis internet |
|---|---|---|
| **LAN-USR** | Postes utilisateurs agence | Non |
| **LAN-SRV** | Serveurs locaux (BIND9, NetBox) | Non |
| **MGMT** | Administration | Non |
| **WiFi corp** | Postes mobiles agence | Sortant uniquement |

⚠️ **Phase 1** : seuls LAN-USR, LAN-SRV, DMZ (siège) et MGMT seront effectivement déployés. Les VLANs WiFi sont dans le plan d'adressage mais ne seront concrétisés que si nécessaire.

---

## 3. Flux principaux

### 3.1 Authentification

- **Postes Windows / Linux du siège** → AD DC1 (siège, LAN-SRV) directement
- **Postes Windows / Linux de l'agence** → AD DC1 via le tunnel WireGuard (cross-sites)
- **Linux (siège ou agence)** → SSSD jointe à AD, kerberos pour SSO

### 3.2 Résolution DNS

DNS en **vue split-horizon** :

- **Vue interne** : résolution des noms `*.corp.acme.local` (AD)
- **Vue externe** : résolution des noms `*.acme.example` (DMZ uniquement, à acquérir si démo publique)
- Le **DNS du siège** (intégré à l'AD) est la source de vérité pour `corp.acme.local`
- Le **BIND9 de l'agence** est secondaire pour `corp.acme.local` (résilience locale en cas de coupure du tunnel) et primaire pour les zones spécifiques agence

### 3.3 Sortie Internet

Chaque site sort par sa propre box internet, via son pfSense local. **Pas** de tromboning forcé : le trafic de l'agence vers internet ne passe **pas** par le siège.

### 3.4 Services exposés (DMZ siège)

- Site web public via Traefik (reverse proxy + TLS)
- Application métier accessible via Traefik (potentiellement protégée par auth basique en phase 1, OIDC ou similaire en bonus)

### 3.5 Sauvegardes

Restic centralisé : agents sur les VM critiques → repository chiffré sur un stockage (NAS dédié, dossier monté, ou éventuellement S3/B2). **Le test de restauration est obligatoire**, sinon la sauvegarde n'est pas considérée comme livrée.

### 3.6 Supervision

- **Prometheus** scrape les `node_exporter` de toutes les VM Linux
- **WMI exporter** sur les VM Windows
- **Grafana** consolide les dashboards

---

## 4. Hébergement physique (phase 1)

### Site A (Siège)
- **Hôte physique** : PC perso membre 1, Windows 10/11, 32 Go RAM
- **Hyperviseur** : VirtualBox (cf. ADR-004)
- **VM** : pfSense A, AD DC1, Docker host (Traefik + monitoring), site web DMZ, postes clients de test

### Site B (Agence)
- **Hôte physique** : PC perso membre 2, Windows 10/11, 32 Go RAM
- **Hyperviseur** : VirtualBox (cf. ADR-004)
- **VM** : pfSense B, BIND9, NetBox, postes clients de test

### Phase 2 (bonus, non MVP)
- **Mini PC (N150 + 16 Go)** : Proxmox VE, services permanents 24/7 (NetBox, Grafana long terme, Restic repo)
- **Vieux PC (LGA 1156)** : éventuellement poste client physique pour démo réelle de jointure AD

---

## 5. Connexion inter-sites

### Tunnel WireGuard
- Endpoint A : pfSense siège, port UDP forwardé sur la box du membre 1
- Endpoint B : pfSense agence, port UDP forwardé sur la box du membre 2
- Réseau de transit : `10.99.99.0/30`
- Routes statiques bidirectionnelles entre les LANs des deux sites

### Modes dégradés
- **Tunnel down** : chaque site reste opérationnel localement (DNS local, sortie internet)
- **Auth AD coupée pour l'agence** : les postes Linux jointes via SSSD continuent en cache. Les postes Windows : comportement à valider, dépend de la GPO et du cache local.

---

## 6. Sécurité (vue d'ensemble)

Le détail des règles firewall et du modèle de menace est traité dans `docs/threat-model.md` (à venir, bonus). Principes posés ici :

- **Default deny** sur les pfSense, règles explicites sortantes et entrantes.
- **Aucun secret en clair** dans le repo (Ansible Vault).
- **MFA recommandé** pour les comptes admin AD (à acter en bonus).
- **Mises à jour** : automatiques pour les paquets de sécurité Debian (rôle Ansible `common`), manuelles avec test pour pfSense / Windows Server.
- **Sauvegardes hors-site** : objectif phase 2 (sauvegarder l'agence chez le siège ou vice-versa via le tunnel).

---

## 7. Limites et hypothèses connues

⚠️ Ce projet est explicitement **un projet d'apprentissage** et non une mise en production. Limites assumées :

- Pas de redondance des pare-feux (un seul pfSense par site).
- Pas de redondance AD en phase 1 (un seul DC).
- Pas de PRA / PCA formalisé.
- Pas de conformité réglementaire (RGPD, NIS2, etc.) bien que les bonnes pratiques soient observées.
- Domaine externe `acme.example` non acquis : la démo publique reste une option en bonus.
- Performances limitées par l'hyperviseur (VirtualBox) et la connectivité résidentielle des hébergeurs.

Ces limites sont documentées dans les ADR concernés et n'invalident pas le caractère pédagogique et démonstratif du projet.

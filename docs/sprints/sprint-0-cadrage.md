# Sprint 0 — Cadrage

> Phase préparatoire avant l'écriture de toute infrastructure. Ce document acte les choix structurants, identifie les risques et liste les prérequis à valider avant de basculer en Sprint 1.

**Statut** : à valider à deux
**Date** : <à remplir>

---

## 1. Objectif du projet

Construire en binôme l'infrastructure réseau et système d'une PME fictive (ACME Corp, 40 personnes, 2 sites), en *Infrastructure as Code*, à des fins **d'apprentissage et de portfolio**.

Le projet n'a pas vocation à être livré dans un délai contraint. La priorité est :

1. **Apprendre les technos en profondeur** (pas de raccourcis "clic-clic" pour gagner du temps)
2. **Produire des livrables incrémentaux** publiables sur GitHub à chaque sprint
3. **Maîtriser une chaîne complète** (réseau, système, IaC, supervision, sauvegarde)

L'IA est utilisée comme **accélérateur d'apprentissage** (explication de concepts, relecture de code, débogage assisté), pas comme générateur de solutions clé en main.

---

## 2. Décisions structurantes actées

### 2.1 Architecture (rappel ADR-001)

Deux sites distincts hébergés respectivement chez chaque membre, reliés par un **tunnel WireGuard site-à-site réel** entre deux pare-feux pfSense.

- **Site A (Siège)** : hébergé chez le membre 1, sur PC perso (32 Go RAM)
- **Site B (Agence)** : hébergé chez le membre 2, sur PC perso (32 Go RAM)
- **CGNAT** : confirmé absent chez les deux membres → faisabilité réseau OK

### 2.2 Hyperviseur de phase 1 — VirtualBox (ou VMware Workstation Pro)

**Décision** : VirtualBox installé par-dessus Windows sur chaque PC perso, lancé uniquement pendant les sessions de projet.

**Justification** :

- Permet de garder Windows comme OS principal (gaming, usage perso intact)
- Aucune perturbation du quotidien
- Réseaux virtuels (Internal Network) suffisants pour reproduire la segmentation VLAN
- pfSense en mode "Bridged Adapter" sur le LAN domestique pour le WAN

**Alternative à considérer** : VMware Workstation Pro, devenu gratuit pour usage personnel (à vérifier sur le site officiel au moment du choix), souvent plus performant.

⚠️ **À valider** : sujet d'un futur ADR-004 ("Choix de l'hyperviseur de phase 1").

### 2.3 Proxmox repoussé en phase 2

Proxmox VE n'est pas utilisé en phase 1. Reporté en phase 2 ou bonus, déployé éventuellement sur le mini PC (Intel N150, 16 Go RAM) pour héberger des **services permanents** (NetBox, Grafana, Restic) accessibles 24/7 indépendamment des sessions de projet.

Le vieux PC (socket LGA 1156) reste optionnel, son CPU exact reste à identifier. Usage envisagé : poste client physique réel pour démontrer la jointure AD.

### 2.4 Stack IaC — Ansible (rappel ADR-002)

**Ansible seul** en phase 1, conformément à l'ADR-002. Apprentissage assumé comme **livrable du projet**, pas comme un coût à minimiser.

L'apprentissage Ansible se fait **en parallèle de l'utilisation**, pas avant. On commence par des rôles simples (paquets, users, fichiers de config) sur les VM Debian, puis on monte en complexité progressivement.

Pour les technos où Ansible n'est pas naturel (pfSense web UI, AD GUI), la configuration manuelle reste acceptable, **à condition d'être documentée** en runbook.

### 2.5 step-ca — décision reportée

step-ca reste dans la stack cible mais sa mise en œuvre concrète est **reportée** à un sprint dédié. Décision finale (le garder, le remplacer par Lets Encrypt + auto-signés, ou le mettre en bonus) à reprendre après le Sprint 2.

---

## 3. Périmètre MVP — découpage en grappes

Le projet est découpé en **grappes thématiques cohérentes**, chacune correspondant à un sprint, chacune produisant un **livrable publiable sur GitHub** (rôles Ansible, runbooks, ADR, captures, schéma).

### Grappe 1 — Réseau bas-niveau (Sprint 1)

**Thème** : pfSense, WireGuard, Ansible sur VM Linux de base.

- 2 × pfSense installés et configurés
- VLANs, NAT, règles de base
- Tunnel WireGuard site-à-site fonctionnel, ping de bout en bout
- Premiers rôles Ansible (paquets de base, users, SSH hardening) sur 1-2 VM Debian "vanilla"

**Livrable GitHub** : rôles Ansible `common`, `ssh-hardening` ; ADR-004 (hyperviseur), ADR-005 (config WG) ; runbook "Reset pfSense", "Reconfig tunnel WG" ; schéma réseau à jour.

### Grappe 2 — Identité et postes clients (Sprint 2)

**Thème** : Active Directory, BIND9, jointure des clients.

- AD DC1 monté sur le siège
- BIND9 sur l'agence avec split-horizon
- 1 poste Windows joint à AD
- 1 poste Linux joint à AD via SSSD
- Rôles Ansible pour la config des clients Linux

**Livrable GitHub** : rôle Ansible `sssd-ad-join`, `bind9-secondary` ; ADR-006 (DNS split-horizon) ; runbooks de jointure ; doc utilisateur (comment se connecter).

### Grappe 3 — Services applicatifs (Sprint 3)

**Thème** : Docker, Traefik, app métier en DMZ.

- Docker host sur le siège
- Traefik configuré avec certificats (auto-signés ou Lets Encrypt selon contexte DMZ)
- 1 application métier déployée (à choisir : Nextcloud, Wiki.js, GLPI, autre)
- Reverse proxy fonctionnel depuis l'extérieur (test depuis un téléphone 4G)

**Livrable GitHub** : `docker-compose.yml` versionnés, rôle Ansible `docker-host` ; ADR-007 (choix de l'appli métier) ; doc d'exploitation.

### Grappe 4 — Observabilité et résilience (Sprint 4)

**Thème** : Prometheus, Grafana, Restic.

- Prometheus + node_exporters sur tous les hôtes
- Grafana avec dashboards de base
- Restic configuré pour sauvegarder les VM critiques
- **Test de restauration documenté** (sinon ça ne compte pas)

**Livrable GitHub** : rôle Ansible `prometheus`, `restic-client` ; dashboards Grafana exportés en JSON ; runbook "Restauration depuis Restic".

### Grappe 5 — Source de vérité (Sprint 5)

**Thème** : NetBox, documentation finale.

- NetBox déployé sur l'agence
- Inventaire complet (sites, racks, VM, IP, VLANs, circuits)
- Synchronisation Ansible ↔ NetBox (inventaire dynamique en bonus)

**Livrable GitHub** : doc d'utilisation de NetBox dans le projet ; export de l'inventaire.

### Hors phase 1 (assumé)

Reportés en phase 2, bonus ou abandonnés selon l'envie après MVP :

- step-ca / PKI interne (à arbitrer)
- 2e DC AD pour HA
- Wazuh (SIEM)
- Suricata (IDS)
- IPv6
- CI GitHub Actions
- Tests Goss
- GPO de durcissement Windows

---

## 4. Cartographie des compétences (binôme)

| Techno | Membre 1 | Membre 2 | Niveau combiné | Risque |
|---|---|---|---|---|
| Ansible | 2/10 | <à remplir> | faible | 🟡 moyen |
| pfSense | 0 | <à remplir> | nul | 🔴 élevé |
| Active Directory | 5/10 | <à remplir> | correct | 🟢 faible |
| WireGuard | bon (serveur) | <à remplir> | bon | 🟢 faible |
| Docker | bon (prod) | <à remplir> | très bon | 🟢 nul |
| BIND9 | <à remplir> | <à remplir> | <à remplir> | 🟡 moyen |
| Prometheus/Grafana | <à remplir> | <à remplir> | <à remplir> | 🟡 moyen |
| Restic | <à remplir> | <à remplir> | <à remplir> | 🟡 moyen |
| NetBox | <à remplir> | <à remplir> | <à remplir> | 🟡 moyen |

**Lecture** : forces solides sur Docker et WireGuard (à exploiter en grappe 3). Trou critique sur pfSense (à attaquer dès la grappe 1). Ansible à apprendre **en l'appliquant** sur du concret simple au début.

---

## 5. Risques et plans B

| Risque | Probabilité | Impact | Plan B |
|---|---|---|---|
| Tunnel WG ne monte pas (NAT, MTU, port forwarding box) | moyenne | bloquant | Tester WG entre les 2 box **dès le début du Sprint 1** ; fallback Tailscale en repli |
| L'un des deux décroche / abandonne | moyenne | bloquant | Documentation propre dès le début, ADR à jour ; un solo peut reprendre |
| Périmètre trop ambitieux dans le délai souhaité | élevée | dégradant | Découpage en grappes incrémentales — chaque grappe est un livrable autonome |
| pfSense plus complexe que prévu | élevée | retard | Sprint 1 dédié uniquement à la grappe réseau, pas de mélange avec AD/Docker |
| L'un éteint sa machine, l'autre site isolé | quasi certaine | dégradant | Documenté dans runbook ; mini PC en service permanent en phase 2 |
| Box CGNAT découverte en cours de route | faible (déjà vérifié) | bloquant | Tailscale ou VPS relais |
| Conflit de plage IP avec LAN domestique | moyenne | gênant | Plan d'adressage en 10.10.x.x / 10.20.x.x choisi spécifiquement pour éviter les 192.168.1.x usuels |

---

## 6. Prérequis techniques à valider avant Sprint 1

Cases à cocher par chaque membre **avant** de démarrer le Sprint 1.

### Membre 1
- [ ] VT-x / AMD-V activé dans le BIOS du PC perso
- [ ] VirtualBox (ou VMware Workstation Pro) installé et fonctionnel
- [ ] Au moins 100 Go libres sur le SSD pour les VM
- [ ] ISO téléchargées : Debian 12/13, Windows Server 2022 Eval, pfSense CE
- [ ] Test "VM jetable Debian" : la VM se lance, a internet, je peux la supprimer
- [ ] Vérification CGNAT (déjà faite, ✅)
- [ ] Accès admin à la box internet (pour port forwarding ultérieur)

### Membre 2
- [ ] Idem ci-dessus

### Communs
- [ ] Repo GitHub créé avec la structure complète
- [ ] Branch protection sur `main`
- [ ] Templates PR + issues en place
- [ ] README, CONTRIBUTING, LICENSE, .gitignore
- [ ] ADR-001, 002, 003 mergés
- [ ] **ADR-004 rédigé** (choix hyperviseur — ce que ce document acte)
- [ ] Schéma réseau draw.io (premier jet, export PNG)
- [ ] Plan d'adressage relu et validé à deux
- [ ] Premier CR de réunion dans `docs/meetings/`
- [ ] Cartographie des compétences complétée à deux (section 4 de ce doc)

---

## 7. Definition of Done — Sprint 0

Le Sprint 0 est clos quand :

1. ✅ Toutes les décisions structurantes sont actées (sections 2 et 3)
2. ✅ La cartographie des compétences est complète à deux (section 4)
3. ✅ Tous les prérequis techniques sont validés (section 6)
4. ✅ Tous les ADR fondateurs sont mergés (001, 002, 003, 004)
5. ✅ Le repo est prêt à recevoir le premier commit d'infrastructure
6. ✅ Une réunion de clôture Sprint 0 a eu lieu, CR rédigé, point de bascule vers Sprint 1 acté

---

## 8. Annexe — Pièges identifiés à éviter

Notes pour ne pas reproduire des erreurs classiques :

- **Ne pas attaquer 5 technos en parallèle**. Les grappes existent pour ça : on absorbe une grappe à la fois.
- **Ne pas négliger la documentation au fil de l'eau**. Un ADR rédigé 3 sprints plus tard = ADR perdu. Idem pour les runbooks.
- **Ne pas confondre "ça marche chez moi" et "c'est livrable"**. Un service est livrable quand : config en Ansible (ou clic-clic documenté), runbook de redémarrage, supervision branchée, sauvegarde testée.
- **Ne pas oublier la résilience à l'humain**. Si l'un des deux est malade 1 semaine, le projet doit pouvoir avancer sur la grappe en cours.
- **Ne pas se laisser piéger par le "syndrome de la stack à la mode"**. step-ca, Suricata, Wazuh, K8s : si personne n'en a besoin pour le MVP, ça reste hors phase 1.

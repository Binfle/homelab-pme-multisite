# Sprint 1 — Réseau bas-niveau

> Grappe 1 : pfSense × 2 + WireGuard site-à-site + premiers rôles Ansible Linux + segmentation VLAN.

---

## Objectif

Mettre en place les **fondations réseau** du projet : deux pfSense fonctionnels (un par site), un tunnel WireGuard site-à-site qui relie les deux réseaux, la segmentation VLAN sur le siège, et les premiers rôles Ansible pour automatiser la configuration de base des serveurs Linux.

À la fin du sprint, **un poste utilisateur du siège doit pouvoir pinger un serveur de l'agence** (et inversement), via le tunnel WireGuard.

---

## Périmètre

### Inclus
- Installation pfSense × 2 sur VM (VMware Workstation Pro)
- Configuration WAN, LAN-USR, LAN-SRV, DMZ, MGMT (siege01) avec VLANs
- Configuration WAN, LAN-USR, LAN-SRV, MGMT (agence01) avec VLANs
- Tunnel WireGuard site-à-site fonctionnel
- Routes statiques inter-sites
- Règles firewall de base (autoriser inter-sites via tunnel, bloquer le reste depuis WAN)
- Premier rôle Ansible `common` (config de base des VM Linux : hostname, sudo, SSH key, paquets de base)
- Inventory Ansible structuré
- Documentation des procédures (procédures d'installation + runbooks)
- ADR-005 — Segmentation par VLANs taggés

### Exclus (pour des sprints suivants)
- Active Directory et BIND9 (Sprint 2)
- Services applicatifs et DMZ peuplée (Sprint 3)
- Monitoring et sauvegardes (Sprint 4)
- IPAM et NetBox (Sprint 5)

---

## Livrables attendus

### ADR
- **ADR-005** : Segmentation par VLANs taggés sur trunk 802.1Q (à rédiger en début de sprint, avant l'implémentation)
- **ADR-006** : Séparation procédures d'installation vs runbooks de remédiation (rédigé en cours de sprint, statut `proposé`)

### Configuration
- VM pfSense `siege01` opérationnelle
- VM pfSense `agence01` opérationnelle
- Configurations exportées (`config.xml` de chaque pfSense) versionnées dans `pfsense/`

### Code Ansible
- `ansible/inventory/hosts.yml` (inventaire siège + agence)
- `ansible/roles/common/` (rôle de base appliqué à toutes les VM Linux)
- `ansible/playbooks/site.yml` (playbook qui applique le rôle common partout)

### Documentation

**Procédures d'installation** (`docs/installations/`)
- `pfsense.md` — installation pfSense sur VMware (siege01 / agence01)

**Runbooks** (`docs/runbooks/`)
- `wireguard-restart.md` — comment redémarrer le tunnel WG
- `pfsense-config-reset.md` — reset de pfSense en cas de mauvaise config
- `connectivity-troubleshooting.md` — diagnostic de coupure inter-sites

### Schémas
- Mise à jour de `diagrams/network-overview.png` si besoin (avec IPs réelles)
- (Optionnel) `diagrams/network-detailed-siege01.png` (Schéma 2)
- (Optionnel) `diagrams/network-detailed-agence01.png` (Schéma 2)

---

## Tâches détaillées

### Bloc 1 — Installation pfSense (jour 1-2)

- [ ] Télécharger l'ISO pfSense CE (dernière version stable) sur les deux PC
- [ ] Créer la VM pfSense sur VMware Workstation Pro (siège et agence)
- [ ] Installer pfSense (procédure standard)
- [ ] Configurer les interfaces réseau VMware (WAN bridged + LAN trunk)
- [ ] Premier accès au webGUI depuis un poste admin
- [ ] Setup wizard de base : hostname, mot de passe, fuseau horaire
- [ ] Rédiger / valider la procédure `installations/pfsense.md` (test croisé)

### Bloc 2 — Segmentation VLAN (jour 2-3)

- [ ] **Rédiger ADR-005** sur la segmentation VLAN
- [ ] Configurer les sous-interfaces VLAN sur la carte LAN de pfSense (VLAN 10, 20, 30, 40 sur siege01 ; VLAN 10, 20, 40 sur agence01)
- [ ] Configurer le trunk côté VMware (Internal Network avec VLAN tagging)
- [ ] Tester l'accès depuis une VM cliente sur chaque VLAN
- [ ] Créer les règles firewall de base par interface

### Bloc 3 — Tunnel WireGuard (jour 3-5)

- [ ] Configurer le port forwarding sur la box ISP pour le port WireGuard (UDP)
- [ ] Installer le package WireGuard sur les deux pfSense
- [ ] Générer les paires de clés sur chaque pfSense
- [ ] Configurer l'interface WireGuard avec les peers respectifs
- [ ] Configurer les routes statiques (siege01 voit `10.20.0.0/16`, agence01 voit `10.10.0.0/16`)
- [ ] Tester la connectivité bidirectionnelle (ping cross-sites)
- [ ] Rédiger le runbook `wireguard-restart.md`

### Bloc 4 — Premiers rôles Ansible (jour 5-7)

- [ ] Installer Ansible sur le poste de contrôle (laptop / VM dédiée)
- [ ] Créer la structure du dossier `ansible/`
- [ ] Rédiger l'inventaire `hosts.yml`
- [ ] Créer le rôle `common` (hostname, sudo, SSH keys, paquets de base, mise à jour)
- [ ] Créer une VM Linux de test (Debian 13 minimal)
- [ ] Appliquer le rôle `common` sur la VM de test
- [ ] Vérifier l'**idempotence** (relancer le playbook : aucun changement)
- [ ] Documenter dans `ansible/README.md`

### Bloc 5 — Validation et clôture (jour 7-8)

- [ ] Vérifier les 5 critères de "livrable" sur chaque tâche
- [ ] Mettre à jour le `README.md` principal avec les nouvelles fonctionnalités
- [ ] Mettre à jour `addressing-plan.md` si l'IP réelle de Traefik / Docker host a évolué
- [ ] Réunion de clôture Sprint 1 + CR
- [ ] Tag Git `sprint-1`

---

## Critères de succès (Definition of Done)

Le Sprint 1 est terminé quand **toutes** les conditions sont réunies :

1. ✅ **Connectivité inter-sites validée** : un ping fonctionne dans les deux sens via le tunnel WG
2. ✅ **Segmentation VLAN active** : un poste sur LAN-USR ne peut pas accéder directement à LAN-SRV sans passer par les règles firewall pfSense
3. ✅ **Rôle Ansible `common` idempotent** : le playbook s'applique sans erreur et ne fait aucun changement à la 2e exécution
4. ✅ **ADR-005 mergé** : la décision sur la segmentation VLAN est tracée
5. ✅ **Procédure `installations/pfsense.md` validée** par test croisé
6. ✅ **3 runbooks rédigés et testés** : WireGuard restart, pfSense reset, troubleshooting connectivité
7. ✅ **Configurations pfSense versionnées** : les `config.xml` sont dans le repo
8. ✅ **README à jour** : la nouvelle infrastructure réseau est mentionnée

---

## Risques identifiés

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Box ISP en CGNAT empêchant le port forwarding | Faible (déjà vérifié) | Bloquant | Plan B : Tailscale ou VPS relais |
| pfSense difficile à apprendre (niveau initial 0) | Élevée | Modéré | Tutoriels vidéo + lab pas-à-pas + entraide en stream |
| Ansible cassé sur première exécution | Moyenne | Modéré | Tester en boucle sur VM jetable, pas en prod |
| Conflit de plage IP avec LAN domestique | Moyenne | Bloquant | `addressing-plan.md` choisi en `10.10.x.x` / `10.20.x.x` (peu commun) |
| Configuration trunk VLAN VMware capricieuse | Élevée (point neuf) | Modéré | Rédiger ADR-005 et tester sur 1 VLAN avant de tout mettre |

---

## Approche de travail

- **Mode** : duplication confrontante (chacun configure son site, comparaison et fusion des bonnes pratiques)
- **Pair-programming Parsec** : pour les sujets complexes (premier WG, premier rôle Ansible)
- **Stream Discord permanent** : communication en continu
- **Workflow Git** : branche par tâche, PR vers main avec review obligatoire (vu compte commun, review en commentaire)

---

## Ce qu'il faut avoir validé en début de Sprint 1

Avant de démarrer les tâches, vérifier :

- [ ] Sprint 0 clos (CR signé, tag `sprint-0` poussé)
- [ ] ADR-005 rédigé et mergé
- [ ] ISO pfSense téléchargée sur les deux PC
- [ ] Ansible installé sur le poste de contrôle de chacun
- [ ] Question tranchée : Internal Networks séparés VS trunk VLAN dans VMware (cf ADR-005)

---

## Estimation

**Durée cible** : 1 à 2 semaines selon la disponibilité du binôme.

⚠️ **Précision honnête** : c'est un sprint **chargé** (3 grosses briques : pfSense, WG, Ansible). Si vous sentez qu'il faut allonger ou réduire le périmètre en cours de route, c'est légitime. Mieux vaut un sprint qui dure 2 semaines avec un livrable solide qu'un sprint bâclé en 1 semaine.

---

## Notes

- Le **rôle Ansible `common`** sera **réutilisé** dans tous les sprints suivants. Soigner sa qualité maintenant = gain de temps plus tard.
- La **configuration pfSense** sera **étendue** au Sprint 3 (DMZ peuplée) et Sprint 4 (règles monitoring). Garder la config simple ici, on raffinera après.
- Le **tunnel WireGuard** étant la **clé de voûte** du projet multi-sites, prendre le temps de bien le valider et le documenter.
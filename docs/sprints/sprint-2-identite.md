# Sprint 2 — Identité & DNS multi-sites

> Grappe 2 : AD primaire au siège + BIND9 multi-rôle à l'agence + clients joints + comparatif Windows Server / Samba 4.

---

## Objectif

Mettre en place les **fondations identité et DNS** du projet : un contrôleur de domaine Active Directory au siège (technologie tranchée en cours de sprint après comparaison Windows Server 2022 vs Samba 4), un serveur BIND9 multi-rôle à l'agence (secondaire de la zone AD du siège, primaire d'une zone agence dédiée, et forwarder pour internet), et quatre clients joints au domaine (1 Windows + 1 Linux par site) qui s'authentifient en SSO via le tunnel WireGuard.

À la fin du sprint, **un poste Linux de l'agence doit pouvoir se connecter en SSO Kerberos** avec un compte AD du siège, et la résolution DNS interne doit continuer à fonctionner **même si le tunnel WireGuard tombe**.

Dimension pédagogique explicite : le binôme installe **Windows Server 2022 et Samba 4 séquentiellement** sur siege01 (et en lab sur agence01) pour maîtriser les deux technologies avant de trancher laquelle garder. Démarche cohérente avec la mouvance de souveraineté numérique en France et l'objectif de portfolio bilingue Windows/Linux.

---

## Rôles attendus de BIND9 sur agence01

BIND9 sur agence01 cumule trois rôles complémentaires :

| Rôle | Description | Bénéfice |
|---|---|---|
| **Secondaire de la zone AD du siège** | Détient une copie synchronisée de la zone `corp.acme.lan` (transfert AXFR/IXFR depuis le DC du siège) | Résolution des noms internes même si le tunnel WireGuard tombe |
| **Primaire de la zone agence** | Détient et sert la zone `agence.acme.lan` (records spécifiques agence) | Autonomie locale pour les noms propres à l'agence |
| **Forwarder/cache pour internet** | Forwarder récursif pour les requêtes externes des clients agence | Résolution internet rapide depuis le cache local, pas d'aller-retour systématique vers les DNS publics |

---

## Périmètre

### Inclus

- **AD primaire** sur siege01, en deux variantes successives pour comparaison :
  - Windows Server 2022 (eval 180j), forêt `corp.acme.lan`
  - Samba 4 sur Debian 13, forêt `corp.acme.lan`
- **Lab d'apprentissage AD** sur agence01 (forêt distincte `lab-agence.acme.lan`) — éteint et archivé en fin de sprint
- **ADR-012** qui tranche la techno AD retenue pour l'archi cible
- **BIND9 multi-rôle** sur agence01 (secondaire zone siège + primaire zone agence + forwarder, conservé pour l'archi cible)
- **Lab d'apprentissage BIND9** sur siege01 — éteint et archivé en fin de sprint
- **4 clients** joints au domaine (1 Windows + 1 Linux par site)
- **SSSD** sur les clients Linux pour SSO Kerberos
- **GPO baseline** (politique mot de passe, audit minimal)
- **Tests cross-sites** via tunnel WireGuard
- **Test mode dégradé** (tunnel WG coupé) : l'agence doit rester résolutive en DNS interne
- **Rôles Ansible** : `sssd`, `bind9` (idempotents)

### Exclus (Sprints suivants ou bonus)

- 2e DC HA, réplication AD multi-site (bonus phase 2)
- File server / partage SMB (Sprint 3 ou bonus)
- Cache local SSSD pour résilience auth en mode dégradé (bonus du sprint si temps)
- GPO de durcissement (bonus phase 2)
- Comparatif SSSD vs realmd vs winbind (SSSD par défaut, comparatif = bonus)
- Services applicatifs Traefik / DMZ peuplée (Sprint 3)
- Monitoring (Sprint 4)
- Sauvegarde (Sprint 4)

---

## Politique d'archivage des labs d'apprentissage

Les VMs montées à des fins purement pédagogiques (lab AD sur agence01, lab BIND9 sur siege01, techno AD non retenue sur siege01) **ne sont pas détruites** en fin de sprint mais :

1. **Éteintes proprement** depuis l'OS invité
2. **Renommées** avec un suffixe `-archive` (ex : `dc01-lab-agence01-winserver-archive`)
3. **IP libérée** (si elle entre en conflit avec une VM cible) : changement vers une IP de la plage libre du même VLAN
4. **Tracées** dans `docs/ip-registry.md` avec l'état `Archivé (lab Sprint 2)`

Justification : permet de revenir consulter la config si besoin pour un futur sprint ou un bonus (notamment un 2e DC HA), sans risque technique (forêts isolées, DC unique → pas de problème d'USN rollback).

Estimation d'espace disque : ~80 Go cumulés pour 4 VMs archivées (~30 Go × 2 Windows + ~10 Go × 2 Debian). Suppression définitive possible au Sprint 5 si jamais réutilisées d'ici là.

---

## Livrables attendus

### ADR

- **ADR-012** — Choix AD final : Windows Server 2022 vs Samba 4 (à rédiger après le comparatif empirique en cours de sprint, statut `proposé` en fin de sprint)
- **ADR-013** — Stratégie DNS multi-sites : BIND9 multi-rôle sur agence (formalise ce qui est déjà esquissé dans `architecture.md` § 3.2, statut `proposé`)

### Configuration

- VM DC active sur siege01 (techno retenue), forêt `corp.acme.lan`
- VM BIND9 active sur agence01, multi-rôle (secondaire siège + primaire agence + forwarder)
- 4 VMs clients joints au domaine
- VMs archivées (lab agence + lab siège + techno non retenue) éteintes et renommées
- Configurations exportées versionnées (configs BIND9, exports des stratégies AD)

### Code Ansible

- `ansible/roles/sssd/` — jointure d'une VM Linux au domaine AD via SSSD
- `ansible/roles/bind9/` — configuration BIND9 avec les trois rôles paramétrables
- `ansible/roles/samba4_dc/` — **uniquement si Samba 4 retenu** par ADR-012
- Si Windows Server retenu : pas de rôle Ansible côté DC (l'install reste manuelle + procédure d'installation détaillée). Possibilité d'ajouter des rôles `ansible.windows` en bonus pour automatiser des points précis (création OU, comptes de service).

### Documentation

**Procédures d'installation** (`docs/installations/`)

- `ad-windows-server-2022.md` — installation initiale + promotion DC + création forêt
- `samba4-dc.md` — installation initiale Samba 4 mode DC + provision domaine
- `bind9-multi-role.md` — configuration des trois rôles (secondaire, primaire zone agence, forwarder)
- `ad-jointure-windows-client.md` — jointure d'un poste Windows au domaine
- `sssd-jointure-debian.md` — jointure d'un poste Debian au domaine via SSSD

**Runbooks** (`docs/runbooks/`)

- `ad-reset-mot-de-passe-utilisateur.md`
- `bind9-restart-zone-transfer.md` — relancer un transfert AXFR si le secondaire est désynchronisé
- `sssd-debug-jointure.md` — diagnostic de jointure SSSD qui échoue
- `ad-promotion-demotion-dc.md` — démontage propre d'un DC (utilisé en interne pendant le sprint)

### Schémas

- Mise à jour de `diagrams/network-overview.txt` avec les nouvelles VMs
- `diagrams/dns-flow.txt` (optionnel) — schéma de résolution interne en mode normal et en mode dégradé

### Mise à jour de docs existantes

- `docs/ip-registry.md` — ajout des IPs des nouvelles VMs (DC, BIND9, clients) et des VMs archivées
- `docs/conventions.md` — ajout éventuel de conventions OU AD, naming utilisateurs, etc.
- `README.md` racine — mention du Sprint 2 livré dans le tableau d'état

---

## Tâches détaillées

### Bloc 0 — Pré-requis

- [ ] Migration contrôleur Ansible agence01 `.100` → `.10` (dette Sprint 1)
- [ ] ISO Windows Server 2022 téléchargée (eval 180j Microsoft)
- [ ] ISO Debian 13 téléchargée (pour BIND9 et Samba 4)
- [ ] ISO Windows 11 téléchargée (pour les clients)

### Bloc 1 — AD Windows Server 2022 

- [ ] **1.1** Création VM `dc01-siege01` (LAN-SRV, IP statique selon conventions / ADR-011)
- [ ] **1.2** Installation Windows Server 2022, hostname, IP statique, fuseau horaire
- [ ] **1.3** Promotion DC : création de la forêt `corp.acme.lan`, niveau fonctionnel 2016 ou 2022
- [ ] **1.4** Vérification DNS intégré AD : zones forward + reverse créées
- [ ] **1.5** Création d'OUs de base : `Users`, `Computers`, `ServiceAccounts`, `Servers`
- [ ] **1.6** Création de comptes test : 1 admin de domaine, 2 utilisateurs standard
- [ ] **1.7** GPO baseline : politique mot de passe, politique d'audit minimal
- [ ] **1.8** Lab d'apprentissage côté agence (réplication par le binôme agence) : VM `dc01-lab-agence01`, forêt **distincte** `lab-agence.acme.lan` — installation seulement, pas de tentative de réplication avec siege01
- [ ] **1.9** Rédaction de `installations/ad-windows-server-2022.md` (procédure validée par test croisé)

### Bloc 2 — AD Samba 4 

- [ ] **2.1** Extinction propre du Windows Server sur siege01 : démotion DC, renommage `dc01-siege01-winserver-archive`, changement d'IP vers une IP libre du même VLAN
- [ ] **2.2** Extinction du lab Windows Server sur agence01 : renommage `dc01-lab-agence01-winserver-archive`
- [ ] **2.3** Création VM `dc01-siege01` (même nom, mais Debian 13 cette fois), IP statique identique à celle utilisée par Windows Server
- [ ] **2.4** Installation Samba 4 mode DC, provision de la forêt `corp.acme.lan`
- [ ] **2.5** Reproduction du contexte Windows : OUs équivalentes, mêmes comptes test
- [ ] **2.6** GPO baseline équivalente (les GPO Samba 4 sont gérées via `samba-tool gpo` et `samba-tool ntacl`)
- [ ] **2.7** Lab d'apprentissage côté agence en Samba 4 (réplication par le binôme agence) : VM `dc01-lab-agence01` en Samba 4, forêt `lab-agence.acme.lan`
- [ ] **2.8** Rédaction de `installations/samba4-dc.md`

### Bloc 3 — Décision et reconstruction de la cible 

- [ ] **3.1** Comparatif empirique des deux expériences sur 5 critères : effort d'installation, automation Ansible, intégration GPO, expérience admin (Windows MMC vs `samba-tool`), intégration des clients Linux
- [ ] **3.2** Rédaction d'**ADR-012** (statut `proposé`) qui tranche la techno retenue pour l'archi cible
- [ ] **3.3** Extinction et archivage des labs agence01 : les deux labs agence (Windows + Samba) sont archivés avec suffixe `-archive`, aucun ne fait partie de l'archi cible
- [ ] **3.4** Extinction et archivage de la techno AD non retenue sur siege01 : `dc01-siege01-<techno>-archive`, IP libérée
- [ ] **3.5** Si reconstruction nécessaire (selon ordre des blocs 1-2 et techno retenue) : remontage du DC final dans la techno cible avec restauration des OUs et comptes test

### Bloc 4 — BIND9 multi-rôle sur agence01

Peut **commencer en parallèle** des blocs 1-2 pour les parties qui ne dépendent pas du DC (primaire zone agence + forwarder).

- [ ] **4.1** Création VM `bind9-agence01` (Debian 13, LAN-SRV agence)
- [ ] **4.2** Installation BIND9, hardening initial (chroot facultatif, ACL, logging)
- [ ] **4.3** **Forwarder/cache** : configurer `forwarders` (DNS publics ou DNS de la box agence), ACL `allow-query` limitée au LAN agence
- [ ] **4.4** Test forwarder : un client agence résout `google.com` via BIND9
- [ ] **4.5** **Primaire de la zone agence** : créer la zone `agence.acme.lan`, fichier de zone avec SOA, NS, quelques records A de test
- [ ] **4.6** Test primaire zone agence : `dig` sur un record A `agence.acme.lan` depuis le LAN agence
- [ ] **4.7** **Secondaire de la zone du siège** (dépend du choix Bloc 3) :
  - Côté DC siège : autoriser `allow-transfer` vers l'IP de BIND9 agence (TSIG si possible, sinon ACL par IP en V1)
  - Côté BIND9 agence : déclarer la zone secondaire `corp.acme.lan`, déclencher AXFR initial
  - Vérifier : `dig` sur un record AD (`dc01-siege01.corp.acme.lan`) répond correctement depuis BIND9 agence
- [ ] **4.8** Lab d'apprentissage BIND9 sur siege01 (réplication par le binôme siège) : installation BIND9 en mode secondaire de `corp.acme.lan`, juste pour comprendre le mécanisme côté secondaire. Éteint et archivé `bind9-lab-siege01-archive` en fin de bloc.
- [ ] **4.9** Rédaction du rôle Ansible `bind9/` (idempotent, testé sur VM neuve)
- [ ] **4.10** Rédaction de `installations/bind9-multi-role.md`
- [ ] **4.11** Rédaction d'**ADR-013** (stratégie DNS multi-sites)

### Bloc 5 — Clients joints au domaine

- [ ] **5.1** Création + jointure `client-win11-siege01-01` : VM Win 11, IP DHCP du LAN_USR siège, DNS pointant vers le DC, jointure au domaine `corp.acme.lan`, test login avec compte AD
- [ ] **5.2** Création + jointure `client-debian-siege01-01` : VM Debian 13, DNS pointant vers le DC, rôle Ansible `common` puis `sssd`, test SSO Kerberos (`kinit`, `id`, `ssh -K` éventuel)
- [ ] **5.3** Rédaction du rôle Ansible `sssd/` (paramétrable par domaine, idempotent)
- [ ] **5.4** Création + jointure `client-win11-agence01-01` : DNS pointant vers BIND9 agence (qui résout `corp.acme.lan` via le secondaire), jointure cross-site via le tunnel WG, test login
- [ ] **5.5** Création + jointure `client-debian-agence01-01` : DNS pointant vers BIND9 agence, SSSD, test SSO Kerberos cross-site
- [ ] **5.6** Rédaction de `installations/ad-jointure-windows-client.md` et `installations/sssd-jointure-debian.md`

### Bloc 6 — Tests mode dégradé

- [ ] **6.1** Couper le tunnel WireGuard côté pfSense agence (désactivation interface ou règle firewall bloquante)
- [ ] **6.2** Depuis un client agence, vérifier que la résolution DNS interne fonctionne toujours (BIND9 secondaire répond depuis ses données cachées localement)
- [ ] **6.3** Vérifier comportement auth : login échouera côté agence (DC inaccessible). Documenter ce comportement honnêtement. Si cache local SSSD activé en bonus, vérifier qu'un compte déjà loggé peut continuer à utiliser son cache.
- [ ] **6.4** Restaurer le tunnel, vérifier que tout reprend
- [ ] **6.5** Documenter le mode dégradé dans `installations/bind9-multi-role.md` ou un fichier dédié

### Bloc 7 — Clôture

- [ ] Vérifier les 5 critères de "livrable" sur chaque tâche
- [ ] Runbooks rédigés et testés croisés
- [ ] Mise à jour de `docs/ip-registry.md` (nouvelles VMs + VMs archivées avec leur état)
- [ ] Mise à jour de `README.md` (tableau d'état des sprints : Sprint 2 ✅ Terminé)
- [ ] Mise à jour de `docs/conventions.md` si conventions nouvelles à acter
- [ ] Tag Git annoté `sprint-2`

---

## Critères de succès (Definition of Done)

Le Sprint 2 est terminé quand **toutes** les conditions sont réunies :

1. ✅ **Authentification cross-sites validée** : un compte AD du siège permet de se connecter sur un client Linux et un client Windows de l'agence, via le tunnel WireGuard
2. ✅ **SSO Kerberos validé** sur les clients Linux (commande `kinit` + service utilisant un ticket Kerberos)
3. ✅ **ADR-012 mergé** (statut `proposé` minimum) : la techno AD retenue est tracée et justifiée
4. ✅ **BIND9 multi-rôle actif** sur agence01 avec les trois rôles validés (secondaire siège, primaire zone agence, forwarder)
5. ✅ **Mode dégradé validé** : tunnel WG coupé → la résolution DNS interne continue à fonctionner depuis l'agence
6. ✅ **Rôles Ansible `sssd` et `bind9` idempotents** : 2e exécution → aucun changement
7. ✅ **Procédures d'installation rédigées et testées** : 5 procédures (AD Windows, Samba, BIND9, jointure Windows, SSSD)
8. ✅ **Runbooks rédigés et testés** : au moins 3 runbooks
9. ✅ **VMs labs étiquetées proprement** : suffixe `-archive`, IPs libérées si conflit, état tracé dans `ip-registry.md`
10. ✅ **README, ip-registry, addressing-plan à jour**

---

## Risques identifiés

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Samba 4 en mode DC : documentation française rare, debug en anglais | Élevée | Modéré | Pair-programming en stream, accepter le temps de lecture des docs officielles Samba |
| Zone transfer AD → BIND9 secondaire capricieux (ACL, TSIG) | Moyenne | Modéré | Commencer sans TSIG (ACL par IP), sécuriser avec TSIG ensuite |
| Jointure cross-site (Win agence vers DC siège) lente ou cassée | Moyenne | Élevé | Tester la résolution DNS et la connectivité Kerberos (TCP/UDP 88, 464, 389, 636) côté pfSense **avant** la jointure |
| GPO baseline trop ambitieuse (perte de temps) | Élevée | Modéré | Limiter strictement au minimum DoD : politique MDP + audit |
| Conflit DNS lors du switch Windows ↔ Samba sur les clients | Moyenne | Modéré | Vider explicitement le cache DNS Windows (`ipconfig /flushdns`) et redémarrer `systemd-resolved` côté Linux entre les migrations |
| Espace disque pris par les VMs archivées (~80 Go) | Moyenne | Faible | Vérifier l'espace disponible sur les hôtes en début de sprint ; suppression définitive possible Sprint 5 si jamais réutilisées |
| Sprint qui déborde (scope ambitieux) | Élevée | Faible | Accepté en amont : "2 semaines pour la forme", on peut déborder sans drame |
| Cache local SSSD trop complexe à configurer correctement | Moyenne | Faible | C'est un bonus, pas dans le DoD |

---

## Approche de travail

- **Mode** : pair-programming intégral en stream sur les blocs structurants (1, 2, 3, 4, 6). Le pilote tape, l'autre suit en direct, intervient, prend des notes pour la procédure.
- **Sur les répliques d'apprentissage** (lab agence pour AD, lab siège pour BIND9) : le binôme du site concerné pilote sur sa machine, l'autre membre reste disponible en stream pour debug.
- **Stream Discord permanent** : décisions de détail à voix haute, tracées en commits / ADR / procédures selon portée.
- **Workflow Git** : branche par bloc (`feat/sprint-2/ad-windows`, `feat/sprint-2/samba4`, `feat/sprint-2/bind9-multi-role`, `feat/sprint-2/clients-jointure`), PR vers `main` avec review obligatoire.

---

## Ce qu'il faut avoir validé en début de Sprint 2

- [x] Sprint 1 clos (tag `sprint-1` poussé)
- [ ] Migration `.100` → `.10` du contrôleur Ansible agence01 effectuée
- [ ] ISO Windows Server 2022, Debian 13, Windows 11 téléchargées sur les deux PC
- [x] Nom de domaine tranché : `corp.acme.lan`
- [x] Choix techno BIND9 tranché : multi-rôle (secondaire + primaire zone agence + forwarder)
- [x] Choix stratégique tranché : Windows Server **et** Samba 4 testés séquentiellement avant décision
- [x] Politique d'archivage des labs actée (extinction + suffixe `-archive`, pas de destruction)

---

## Estimation

⚠️ **Précision** : ce sprint est **chargé** par construction (double exploration Windows Server + Samba 4, BIND9 multi-rôle, 4 clients, tests cross-sites + mode dégradé). Le binôme a accepté en amont que la durée puisse déborder sans drame. La priorité est la qualité de l'apprentissage et la solidité des livrables, pas la vitesse.

---

## Notes

- Le **rôle Ansible `sssd`** sera **réutilisé** sur tous les futurs clients Linux du projet (Sprints 3-5). Soigner sa qualité maintenant = gain de temps plus tard.
- Le **rôle Ansible `bind9`** sera **étendu** au Sprint 5 lors de l'intégration NetBox (peuplement DNS depuis NetBox en bonus).
- La **techno AD finale** retenue par ADR-012 sera réutilisée tel quel jusqu'à la fin du projet. Pas de rebascule en cours de route.
- Le **mode dégradé** (tunnel coupé) sera retesté à chaque sprint qui touche au DNS ou à l'auth, pour vérifier qu'on n'a pas régressé.
- Les **VMs labs archivées** sont conservées éteintes pour permettre de revenir consulter la config si besoin (notamment pour le bonus "2e DC HA" en phase 2). Suppression définitive possible Sprint 5 si jamais réutilisées.

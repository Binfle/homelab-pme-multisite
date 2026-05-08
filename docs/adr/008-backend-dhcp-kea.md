# ADR-008 — Choix du backend DHCP : Kea (au lieu d'ISC)

**Statut** : proposé
**Date** : 07/05/2026
**Remplace** : néant
**Remplacé par** : néant

---

## Contexte

Lors de la configuration des serveurs DHCP par interface dans pfSense (Sprint 1), deux backends sont disponibles :

- **ISC DHCP** : implémentation historique d'ISC (Internet Systems Consortium), utilisée depuis ~25 ans dans pfSense et l'industrie.
- **Kea DHCP** : implémentation moderne, développée par la même fondation (ISC) comme successeur officiel d'ISC DHCP.

Sources factuelles :

- ISC a annoncé en **fin 2022** la fin du développement actif d'ISC DHCP. Le serveur passe en mode "maintenance only" (uniquement les correctifs de sécurité, pas de nouvelles fonctionnalités). Kea est désigné comme remplaçant officiel.
- pfSense 2.8.1 marque ISC DHCP comme **"Deprecated"** dans son interface (vu dans `System > Advanced > Networking > Server Backend`) et affiche un bandeau d'avertissement sur les pages DHCP correspondantes.
- pfSense intègre Kea comme alternative depuis ses versions 2.7+ (à confirmer précisément si une trace est nécessaire).

---

## Décision

Utiliser **Kea DHCP** comme backend DHCP dans pfSense, sur les deux sites (siege01 et agence01).

### Mise en œuvre

Bascule effectuée via :

```
System > Advanced > Networking > DHCP Options > Server Backend = Kea DHCP
Save (en bas de page)
```

Une fois basculé, l'écran de configuration DHCP par interface (`Services > DHCP Server > <interface>`) s'adapte aux options Kea. La structure générale reste similaire à ISC mais quelques différences mineures :

- Onglet "Settings" partagé pour les options globales.
- Notion de "Primary Address Pool" + "Additional Pools" (Kea permet plusieurs pools dans un même subnet).
- Options "DNS Registration" / "Early DNS Registration" (par défaut "Track server").
- Pas de champ BOOTP (Kea ne supporte pas BOOTP, jugé obsolète).

### Périmètre d'application Sprint 1

- DHCP server activé sur **LAN_USR** uniquement (par site), avec une plage `.100 → .200` du sous-réseau correspondant.
- Pas de DHCP sur LAN_SRV (les serveurs auront des IPs statiques par convention).
- Pas de DHCP sur DMZ (les serveurs exposés ont des IPs statiques par convention de sécurité).
- Pas de DHCP sur MGMT (les administrateurs configurent leur poste manuellement).

### Évolution prévue (Sprint 2)

Lorsque l'AD DC1 sera installé (`10.10.20.10` selon `addressing-plan.md`), il faudra modifier le champ "DNS Servers" de la config DHCP sur LAN_USR pour pointer vers l'AD au lieu de pfSense. Une issue GitHub `feature/sprint-2-dns-via-ad` peut être créée pour ne pas l'oublier.

---

## Conséquences

### Positives

- **Backend en développement actif** : pas de dette technique liée à un produit EOL.
- **API REST native** (JSON-RPC) : ouvre la possibilité d'automatiser la gestion DHCP depuis NetBox, Ansible, ou tout autre outil — utile en phase 2 quand l'infra IaC sera plus mature.
- **Architecture modulaire** : hooks pour extension, support de backends de stockage variés (mémoire, MySQL, PostgreSQL).
- **IPv6 natif** dès la conception (cohérent si on étend le projet à IPv6 en bonus).
- **Future-proof** : Kea sera le seul backend supporté dans pfSense à terme.

### Négatives

- **Documentation web moins riche** que pour ISC : 25 ans d'antériorité d'ISC, énormément de tutos/Stack Overflow le concernent. Kea progresse mais le rapport est encore inégal.
- **Interface pfSense légèrement différente** : nécessite de relire la doc pfSense pour Kea (mineur, l'essentiel des concepts est commun).
- **Certaines options ISC peuvent ne pas être encore portées sur Kea** dans pfSense 2.8.1 — à découvrir au cas par cas. Pour le périmètre actuel (range simple, domain name, DNS, gateway), tout est présent.
- **Comportement moins éprouvé** sur les edge cases : potentiel de petits bugs ou comportements inattendus à découvrir.

### Risques

- **Régression Kea sur une fonctionnalité utilisée plus tard** (ex : statiques mappings complexes, options avancées, intégration AD-DNS) — à monitorer au fur et à mesure des sprints.
- **Évolution de l'API pfSense** : si pfSense modifie l'intégration Kea entre versions, certains réglages pourraient nécessiter une révision.

---

## Alternatives écartées

### ISC DHCP

- **Écartée** : marqué EOL par ISC depuis 2022, marqué "Deprecated" dans pfSense 2.8.1.
- Choix non-pérenne pour un projet nouveau démarré en 2026.
- Acceptable seulement si Kea posait des problèmes bloquants (non observés à ce stade).

### Backend DHCP externe (non-pfSense)

- **Écartée** : sortirait pfSense de son rôle de "boîte centrale" de chaque site.
- Sur-complexe pour un périmètre PME : nécessiterait un serveur dédié (Linux + ISC ou Kea, ou Windows Server + DHCP role) avec relais DHCP par VLAN.
- Pourrait être envisagé en phase 2 si l'AD Microsoft devient le serveur DHCP de référence pour les LAN internes (pratique courante en environnement Active Directory).

### Pas de DHCP du tout (toutes IPs statiques partout)

- **Écartée** : non viable pour un VLAN utilisateurs où les postes peuvent être nombreux et changer fréquemment.
- Acceptable pour LAN_SRV / DMZ / MGMT (déjà en statique par décision dans ce même ADR), mais pas pour LAN_USR.

---

## Conditions de révision

Cet ADR sera révisé si :

- Kea révèle des bugs ou limitations bloquantes pendant les Sprints suivants (notamment lors de l'intégration avec AD-DNS au Sprint 2).
- pfSense décide à terme de retirer Kea (improbable : c'est l'inverse du sens de l'histoire).
- L'AD Windows devient le serveur DHCP de référence pour certains VLANs (pratique courante en environnement Microsoft) : un nouvel ADR pourrait acter ce partage.

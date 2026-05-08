# ADR-009 — Choix de WireGuard pour le tunnel site-à-site et configuration sur pfSense

**Statut** : Validé
**Date** : 07/05/2026
**Remplace** : néant
**Remplacé par** : néant

---

## Contexte

L'architecture deux sites avec VPN site-à-site (cf ADR-001) impose un tunnel chiffré entre les pfSense de siege01 et agence01 pour permettre la communication transparente des services internes entre les deux sites (LAN_USR, LAN_SRV).

Trois solutions VPN site-à-site sont disponibles nativement dans pfSense 2.8.1 :

- **IPsec** : standard historique de l'industrie, support enterprise large.
- **OpenVPN** : VPN open-source mature, basé sur SSL/TLS.
- **WireGuard** : protocole moderne (2016, intégré au kernel Linux 5.6), basé sur cryptographie moderne (Curve25519, ChaCha20, Poly1305, BLAKE2s).

Plusieurs aspects ont été pris en compte :

1. **Simplicité de configuration** : nombre d'options à comprendre, lisibilité du fichier de config.
2. **Modernité cryptographique** : algorithmes utilisés, taille des clés, agilité cryptographique.
3. **Disponibilité dans pfSense 2.8.1** : intégration native ou package, niveau de maturité.

---

## Décision

### 1. Choix du protocole : WireGuard

Le tunnel site-à-site entre siege01 et agence01 utilise **WireGuard** (package `pfSense-pkg-WireGuard 0.2.9_6`).

### 2. Topologie du tunnel

- **Subnet de transit** : `10.99.99.0/30` (cf `addressing-plan.md`)
  - `10.99.99.1/30` : pfSense siege01
  - `10.99.99.2/30` : pfSense agence01
- **Port d'écoute UDP** : `47790` (port aléatoire dans la range non-standard, choix défensif vs port WireGuard par défaut 51820 plus exposé aux scanners)
- **MTU** : 1500 (par défaut sur l'interface)
- **Persistent Keepalive** : 25 secondes (recommandation officielle WireGuard pour traversée NAT)

### 3. Configuration cryptographique

- Génération des clés Curve25519 directement par pfSense (interface `VPN > WireGuard > Tunnels > Generate`)
- Pas de Pre-shared Key (PSK) en première mise en route — peut être ajoutée ultérieurement comme couche de sécurité supplémentaire
- Échange des clés publiques entre les deux sites via canal sécurisé (messagerie privée pour la phase de mise en place)

### 4. Configuration réseau pfSense — pièges identifiés

Trois pièges spécifiques à pfSense 2.8.1 + pkg WireGuard 0.2.9_6 ont été identifiés et appliqués lors de la mise en place :

#### 4.1. IP du tunnel sur l'interface assignée, pas sur le tunnel

Deux méthodes existent pour assigner l'IP de transit du tunnel :

- **Méthode A** : IP renseignée dans `VPN > WireGuard > Tunnels > Edit > Interface Addresses`, et `IPv4 Configuration Type = None` sur l'interface assignée.
- **Méthode B** : IP renseignée sur l'interface assignée (`Interfaces > WG > IPv4 Configuration Type = Static IPv4 = 10.99.99.X/30`), section "Interface Addresses" du tunnel laissée vide.

→ **La Méthode A ne fonctionne pas** dans cette version de pfSense pour le routage et les diagnostics (`Diagnostics > Ping` avec source `WG` échoue car l'IP n'est pas bound au niveau OS sur l'interface). La **Méthode B est obligatoire**.

#### 4.2. Allowed IPs doit inclure le subnet de transit

Les `Allowed IPs` du peer servent à la fois :
1. De **règle de routage** ("envoyer ce trafic vers ce peer")
2. De **filtre entrant** ("n'accepter QUE le trafic dont la source matche ces IPs")

Si on ne met que les subnets LAN distants (`10.10.0.0/16` côté agence pour atteindre siege), tout trafic provenant de `10.99.99.X` (IP des pfSense eux-mêmes sur le tunnel) est **droppé silencieusement** côté réception (filtre entrant).

→ **Il faut ajouter `10.99.99.0/30` aux Allowed IPs du peer**, en plus du subnet LAN distant.

Configuration finale des Allowed IPs :
- Côté agence01 (peer wg-peer-siege01) : `10.10.0.0/16` + `10.99.99.0/30`
- Côté siege01 (peer wg-peer-agence01) : `10.20.0.0/16` + `10.99.99.0/30`

#### 4.3. Routes statiques manuelles obligatoires

Contrairement à `wg-quick` en Linux natif (qui crée automatiquement les routes à partir des Allowed IPs), **pfSense ne génère pas de routes statiques automatiquement** à partir de la config WireGuard.

→ Il faut créer manuellement :
1. Une **gateway** dans `System > Routing > Gateways` :
   - Interface : WG
   - Gateway : IP du peer (10.99.99.1 côté agence, 10.99.99.2 côté siege)
   - Disable Gateway Monitoring : **coché** (sinon pings de monitoring saturent ou marquent à tort le tunnel down)
2. Une **route statique** dans `System > Routing > Static Routes` :
   - Destination : subnet du site distant (`10.10.0.0/16` côté agence, `10.20.0.0/16` côté siege)
   - Gateway : la gateway WG créée à l'étape 1

Sans ces routes, le trafic vers le site distant emprunte la route par défaut (WAN → Internet) et se perd.

### 5. Règles firewall

- **WAN entrant** : Pass UDP port 47790 depuis l'IP publique du peer (filtrage strict par IP source pour limiter la surface d'attaque)
- **Interface WG** : Pass any/any (permissif en première mise en route, à raffiner par service en Sprint 3+ quand des services réels seront déployés)
- **Bonus** sur les box internet : port forwarding UDP 47790 vers l'IP WAN de pfSense

---

## Conséquences

### Positives

- **Simplicité de configuration** : la config WireGuard tient sur 1 page (tunnel + peer), contre plusieurs onglets et dizaines d'options pour IPsec.
- **Performance native** : WireGuard est implémenté au niveau kernel, beaucoup plus performant que OpenVPN en userspace. Latence ~12-20ms confirmée sur le lab (FAI résidentiels Bouygues + autre).
- **Cryptographie moderne** : pas de choix d'algorithme à faire, tout est figé sur des primitives fortes et auditées (Curve25519, ChaCha20, etc.). Pas de risque de mauvaise config crypto.
- **Configuration minimale** : 4 paramètres essentiels par peer (clé publique, allowed IPs, endpoint, keepalive). Lisible et reproductible.
- **Reproductibilité** : la config WireGuard est exportable (config.xml pfSense ou export propre via le package). Facile à partager, sauvegarder, versionner.

### Négatives

- **Pas de tunnel chiffré au niveau IPsec ESP** : interopérabilité limitée avec d'autres équipements (routeurs Cisco, Juniper, équipements MPLS) qui parlent IPsec mais pas WireGuard. Non bloquant pour ce projet (deux pfSense de chaque côté).
- **Pièges spécifiques pfSense** (cf section 4) : 3 pièges non documentés clairement par Netgate qui peuvent coûter du temps en première mise en place — d'où ce présent ADR pour les figer dans la doc projet.
- **Pas de support officiel Netgate** : Netgate (l'éditeur de pfSense) ne fournit pas de support commercial sur WireGuard, contrairement à IPsec et OpenVPN.
- **Pas de tunnel "manageable" type IKEv2** : pas de mécanisme de re-négociation de clés intégré (les clés WireGuard sont statiques). Pour un environnement enterprise avec rotation de clés, ça nécessite de la gymnastique manuelle.

### Risques

- **Si pfSense modifie l'intégration WireGuard** dans une version future (déjà arrivé une fois entre 2.5 et 2.6), la config peut nécessiter une migration. À monitorer lors des mises à jour pfSense.
- **Pas de fallback** : si WireGuard pose problème en production, il faudra basculer manuellement sur IPsec ou OpenVPN. Pas de tunnel de secours configuré actuellement (hors périmètre MVP).

---

## Alternatives écartées

### IPsec (IKEv2)

- **Écarté** pour la complexité de configuration (multiples phases IKE, propositions cryptographiques à choisir, edge cases NAT-T, etc.).
- Mature et stable, mais moins formateur sur un projet portfolio qui valorise la modernité.
- À reconsidérer si un jour l'interopérabilité avec un équipement tiers (routeur enterprise) devient un besoin.

### OpenVPN

- **Écarté** principalement pour les performances : userspace, plus lent que WireGuard surtout sur petites VM.
- Configuration plus verbeuse (certificats, CRL, options TLS).
- Reste une alternative crédible si WireGuard pose problème (mature, très bien documenté dans pfSense).

### Tunnel L2TP/IPsec

- **Écarté** : protocole obsolète, prévu pour le client-to-site mobile, pas pour le site-à-site.

### Tunnel via VPS relais (alternative en cas de CGNAT)

- **Non écarté définitivement** : reste comme plan B documenté dans `sprint-1-reseau.md` § Risques si l'un des deux sites passait en CGNAT.
- Pas activé dans le MVP car pas de CGNAT chez l'un ou l'autre du binôme (vérifié au début du Bloc 3).

---

## Conditions de révision

Cet ADR sera révisé si :

- WireGuard révèle des limitations bloquantes (instabilité, pertes de paquets, incidents de sécurité).
- L'un des sites passe en CGNAT et nécessite une bascule sur la solution VPS relais.
- Un nouveau site (agence02, etc.) est ajouté avec un équipement tiers qui ne supporte pas WireGuard, imposant IPsec.
- Netgate change de stratégie sur le package WireGuard (dépréciation, retrait, etc.).
- Une rotation de clés régulière devient nécessaire (un nouvel ADR pourrait acter une procédure outillée).

# Procédure d'installation — Tunnel WireGuard site-à-site

> Procédure d'installation et de configuration d'un tunnel WireGuard site-à-site entre deux pfSense, dans le contexte du projet ACME Corp (sites siege01 ↔ agence01).
>
> **Référence ADR** : [ADR-009](../adr/009-wireguard-site-to-site.md)
> **Référence plan d'adressage** : [addressing-plan.md](../addressing-plan.md) § "Subnet de transit VPN"
> **Référence flux** : [ADR-005](../adr/005-segmentation-vlan-et-flux.md) (matrice de flux inter-VLAN)

---


## Pré-requis

- pfSense 2.8.1 CE installé et opérationnel sur les deux sites (cf [`pfsense-hyperv.md`](pfsense-hyperv.md))
- Au moins un VLAN configuré côté chaque site avec au moins une VM cliente (cf [ADR-005](../adr/005-segmentation-vlan-et-flux.md))
- Les deux box internet :
  - **Pas en CGNAT** (vérifier avec `whatismyipaddress.com` vs IP WAN affichée dans l'interface admin de la box — si différentes et IP en `100.64.X.X`, c'est du CGNAT, plan B nécessaire)
  - Port forwarding UDP fonctionnel
- Clé publique et IP publique du peer (à échanger via canal sécurisé, ex: messagerie privée Discord)

## Pièges connus à éviter

Lire d'abord ces 2 pièges avant de suivre la procédure. Ils sont **toujours rencontrés** sur pfSense 2.8.1 + pkg WireGuard 0.2.9_6 :

1. **Allowed IPs doit inclure le subnet de transit** (`10.99.99.0/30`) en plus du subnet LAN distant. Sinon les pings entre les pfSense (10.99.99.X ↔ 10.99.99.X) sont droppés silencieusement par WireGuard à la réception.

2. **Routes statiques manuelles obligatoires**. Contrairement à `wg-quick` Linux, pfSense ne génère pas automatiquement les routes à partir des Allowed IPs. Il faut créer une **gateway** sur l'interface WG, puis une **route statique** pour le subnet du site distant.

---

## Phase 0 — Vérifications préalables

### 0.1. Pas de CGNAT

Sur chaque site :

1. Aller sur https://whatismyipaddress.com (ou équivalent) → noter l'IPv4 publique
2. Se connecter à l'interface admin de la box internet → onglet "État Internet" / "WAN" → noter l'IP WAN box
3. **Comparer** : si les deux IPs sont identiques, pas de CGNAT.
4. Si l'IP WAN box est en `100.64.X.X` à `100.127.X.X` (range CGNAT 100.64.0.0/10) → CGNAT confirmé, basculer sur le plan B (Tailscale ou VPS relais — hors périmètre de cette procédure).

### 0.2. Port forwarding faisable

Vérifier dans l'interface admin de la box que la fonction **NAT/PAT** ou **Redirections de ports** est disponible. Pas la peine de créer la règle maintenant, elle sera créée à la Phase 6.

### 0.3. Choix du port d'écoute

Convenir d'un port UDP commun entre les deux sites. Le port WireGuard standard est `51820`, mais peut être remplacé par un port aléatoire dans la range non-standard pour réduire la surface d'attaque face aux scanners automatiques.

**Convention projet** : `47790` (utilisé dans cette procédure).

---

## Phase 1 — Installation du package WireGuard

À faire sur les **deux pfSense** :

1. `System > Package Manager > Available Packages`
2. Rechercher `WireGuard`
3. Cliquer **Install** sur le package `pfSense-pkg-WireGuard`
4. Confirmer et attendre la fin de l'installation
5. Le menu `VPN > WireGuard` apparaît dans la barre supérieure (refresh de la page si nécessaire)

Note : malgré ce que disent certains tutos, WireGuard **n'est pas intégré nativement** dans pfSense 2.8.1 CE — l'installation du package est obligatoire.

---

## Phase 2 — Activer WireGuard et créer le tunnel

### 2.1. Activer globalement

`VPN > WireGuard > Settings`

- Cocher **Enable WireGuard**
- Save → Apply Changes

### 2.2. Créer le tunnel

`VPN > WireGuard > Tunnels > Add Tunnel`

| Champ | Valeur côté siege01 | Valeur côté agence01 |
|---|---|---|
| Description | `Tunnel siege01 to agence01` | identique |
| Listen Port | `47790` | identique |
| Interface Keys | Cliquer **Generate** (génère la paire de clés) | idem |
| Interface Addresses | **laisser vide** | identique |

Save Tunnel.

### 2.3. Récupérer la clé publique

Une fois le tunnel sauvegardé, retourner dans la liste : la **Public Key** est affichée. La copier et la transmettre au peer via canal sécurisé. Demander en retour la clé publique du peer.

---

## Phase 3 — Créer le peer

`VPN > WireGuard > Peers > Add Peer`

| Champ | Valeur côté siege01 | Valeur côté agence01 |
|---|---|---|
| Enable | O | O |
| Tunnel | `tun_wg0 (Tunnel siege01 to agence01)` | identique |
| Description | `wg-peer-agence01` | `wg-peer-siege01` |
| Dynamic Endpoint | **décoché** (Endpoint en dur pour pouvoir initier) | décoché |
| Endpoint | IP publique de **agence01** | IP publique de **siege01** |
| Endpoint Port | `47790` | `47790` |
| Keep Alive | `25` | `25` |
| Public Key | clé publique du peer (envoyée par l'autre site) | clé publique du peer |
| Pre-shared Key | (laisser vide pour la mise en route initiale) | identique |

**Allowed IPs** (ajouter **deux** entrées avec le bouton "Add Allowed IP") :

| # | Subnet (côté siege01) | Subnet (côté agence01) | Description |
|---|---|---|---|
| 1 | `10.20.0.0/16` | `10.10.0.0/16` | Réseau du site distant |
| 2 | `10.99.99.0/30` | `10.99.99.0/30` | Subnet de transit VPN piège 2 |

Save Peer.

---

## Phase 4 — Assigner et configurer l'interface WireGuard

### 4.1. Assigner l'interface

`Interfaces > Assignments`

- Dans **Available network ports**, sélectionner `tun_wg0` dans le menu déroulant
- Cliquer **Add**
- Une nouvelle interface (généralement `OPTx` ou similaire) apparaît
- Save

### 4.2. Configurer l'interface

Cliquer sur le lien de la nouvelle interface (ex: `OPTx`) :

| Champ | Valeur |
|---|---|
| Enable | O |
| Description | `WG` |
| **IPv4 Configuration Type** | **Static IPv4** piège 1 (NE PAS mettre "None") |
| **IPv4 Address** | `10.99.99.1` / `30` côté siege01<br>`10.99.99.2` / `30` côté agence01 |

Save → Apply Changes.

### 4.3. Vérification

Sur la console pfSense (VMConnect → menu CLI), la liste des interfaces doit afficher :

```
WG (optx) -> tun_wg0 -> v4: 10.99.99.X/30
```

Si l'IP n'apparaît pas en CLI, vérifier que `IPv4 Configuration Type = Static IPv4` (pas `None`).

---

## Phase 5 — Règles firewall

### 5.1. Sur l'interface WAN — autoriser le trafic WireGuard entrant

`Firewall > Rules > WAN > Add` (en haut de page)

| Champ | Valeur |
|---|---|
| Action | Pass |
| Interface | WAN |
| Address Family | IPv4 |
| Protocol | UDP |
| Source | Single host or alias = IP publique du peer |
| Destination | WAN address |
| Destination Port Range | `47790` |
| Description | `Allow WireGuard from <site_distant>` |

Save → Apply Changes.

! Filtrage par IP source = posture de sécurité plus stricte qu'un `any` ouvert. Si l'IP du peer est dynamique et change, basculer vers un alias avec DDNS plus tard.

### 5.2. Sur l'interface WG — autoriser le trafic du tunnel

`Firewall > Rules > WG > Add`

| Champ | Valeur |
|---|---|
| Action | Pass |
| Interface | WG |
| Address Family | IPv4 |
| Protocol | Any |
| Source | Any |
| Destination | Any |
| Description | `Allow tunnel traffic (à raffiner)` |

Save → Apply Changes.

La règle "any/any" est **permissive volontairement** pour la première mise en route. À raffiner ensuite par service (ex: Pass WG_subnet → LAN_USR_subnets port 53,80,443) selon la matrice de flux.

---

## Phase 6 — Port forwarding sur la box internet

Sur chaque box internet (Bouygues Bbox, Livebox, Freebox, etc.), section NAT/PAT ou Redirections de ports :

| Champ | Valeur |
|---|---|
| Nom | `pfSense WireGuard` |
| Protocole | UDP |
| Port externe | `47790` |
| Équipement | l'host correspondant à l'IP WAN du pfSense (ex: `192.168.1.147` pour agence01) |
| Port interne | `47790` |
| État | Activé (toggle ON) |

Si la box a plusieurs hosts portant le même nom (résidu de DHCP suite à des installations successives), bien vérifier le **MAC address** :
- Préfixe `00:15:5d:XX:XX:XX` = MAC Hyper-V (le pfSense actuel)
- Préfixe `00:0c:29:XX:XX:XX` = MAC VMware (résidus à ignorer)

---

## Phase 7 — Routes statiques (piège 3)

Les Allowed IPs WireGuard ne génèrent **pas** de routes automatiques sur pfSense. Il faut créer manuellement la gateway et la route.

### 7.1. Créer la gateway

`System > Routing > Gateways > Add`

| Champ | Valeur côté siege01 | Valeur côté agence01 |
|---|---|---|
| Interface | WG | WG |
| Address Family | IPv4 | IPv4 |
| Name | `WG_GATEWAY` | identique |
| Gateway | `10.99.99.2` (IP du peer agence) | `10.99.99.1` (IP du peer siege) |
| Description | `Gateway tunnel WireGuard vers agence01` | `Gateway tunnel WireGuard vers siege01` |
| **Disable Gateway Monitoring** | ⚠️ **coché** (sinon pings de monitoring saturent et peuvent marquer la gateway down à tort) | coché |

Save → Apply Changes.

### 7.2. Créer la route statique

`System > Routing > Static Routes > Add`

| Champ | Valeur côté siege01 | Valeur côté agence01 |
|---|---|---|
| Destination network | `10.20.0.0` / `16` | `10.10.0.0` / `16` |
| Gateway | `WG_GATEWAY` | `WG_GATEWAY` |
| Description | `Route vers agence01 via tunnel WG` | `Route vers siege01 via tunnel WG` |

Save → Apply Changes.

### 7.3. Vérification

`Diagnostics > Routes` doit afficher une nouvelle ligne :

```
Côté siege01 :   10.20.0.0/16 -> 10.99.99.2 -> tun_wg0
Côté agence01 :  10.10.0.0/16 -> 10.99.99.1 -> tun_wg0
```

---

## Phase 8 — Tests de validation

À effectuer une fois que **les deux sites** ont terminé l'intégralité des phases ci-dessus.

### 8.1. Statut WireGuard

`VPN > WireGuard > Status`

- Tunnel UP (flèche verte)
- Peer affiché avec `Latest Handshake` récent (quelques secondes / minutes)
- RX/TX > 0 bytes

### 8.2. Ping pfSense ↔ pfSense via le tunnel

`Diagnostics > Ping`

| Champ | Valeur |
|---|---|
| Hostname | IP de transit du peer (`10.99.99.2` côté siege, `10.99.99.1` côté agence) |
| Source Address | WG |
| IP Protocol | IPv4 |

Attendu : 0% packet loss.

### 8.3. Ping cross-site depuis VM cliente

Depuis une VM Debian sur LAN_USR :

```bash
# Côté agence01, vers une VM ou la gateway du LAN_USR siege
ping -c 4 10.10.10.1     # gateway pfSense LAN_USR siege
ping -c 4 10.10.10.100   # VM Debian LAN_USR siege (si présente)
```

Attendu : 0% packet loss, TTL=62 (deux sauts via le tunnel).

---

## Bonus — Améliorations à prévoir (hors mise en place initiale)

- **Pre-shared Key** : ajouter une couche de sécurité supplémentaire en générant et configurant une PSK (à `Pre-shared Key` côté peer des deux côtés). (a étudier prochainement)
- **DDNS** si l'une des IPs publiques devient dynamique : configurer un service DynDNS (DuckDNS, NoIP, Cloudflare, etc.) et mettre le hostname DDNS comme Endpoint au lieu de l'IP en dur.
- **Raffinement firewall WG** : remplacer la règle "any/any" par des règles spécifiques par service (Pass `WG_subnet` → `LAN_SRV` port 80,443 ; Pass `WG_subnet` → `LAN_USR` ICMP ; etc.).
- **Monitoring** : ajouter le tunnel WireGuard à la supervision (NetBox, Grafana en phase 2), avec alerting sur l'absence de handshake récent.
- **Backup config** : exporter régulièrement la config pfSense (`Diagnostics > Backup & Restore`) et versionner les exports dans le repo Git (sans les clés privées, à gitignorer).

---

## Annexe — Troubleshooting rapide

| Symptôme | Cause probable | Action |
|---|---|---|
| Status WireGuard : pas de handshake | Problème de port forwarding box ou règle WAN firewall manquante | Vérifier les 2 box (UDP `<port>` → IP WAN pfSense), vérifier la règle Pass UDP sur WAN |
| Handshake OK, ping pfSense ↔ pfSense KO | Allowed IPs n'inclut pas `10.99.99.0/30` (piège 2) | Ajouter `10.99.99.0/30` aux Allowed IPs des deux peers |
| Ping pfSense OK, ping VM cross-site KO | Routes statiques manquantes (piège 3) | Créer la gateway WG_GATEWAY + route statique vers le subnet distant |
| Diagnostics > Ping avec source=WG ne sait pas quelle IP utiliser | IP non bound à l'interface (piège 1) | Mettre `IPv4 Configuration Type = Static IPv4` sur l'interface WG |
| RX bas, TX élevé (asymétrie) | Le peer ne reçoit pas / ne peut pas répondre | Vérifier côté peer : règle WG firewall, port forwarding, routes |

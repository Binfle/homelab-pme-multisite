# Plan d'adressage

> *Hypothèse de départ — à valider et ajuster collectivement.*

## Convention de nomenclature

Format des identifiants techniques (réseaux, VM, groupes Ansible) :

```
<role><NN>-<segment>
```

- `<role>` : `siege` ou `agence` (rôle fonctionnel du site)
- `<NN>` : numéro à 2 chiffres (`01`, `02`, ...) — préfixe `0` obligatoire pour le tri alphabétique
- `<segment>` : `lan-usr`, `lan-srv`, `dmz`, `mgmt`, `wifi-corp`, `wifi-guest`

**Exemples** :

- `siege01-lan-srv` : LAN serveurs du premier (et seul) siège
- `agence01-lan-usr` : LAN utilisateurs de la première agence
- `agence02-lan-usr` : (futur) LAN utilisateurs de la seconde agence

⚠️ Cette convention vaut pour les identifiants **techniques**. Dans la **prose** et la **documentation narrative**, on continue d'écrire "le Siège" et "l'Agence" en français lisible.

---

## Sites

| Site | Identifiant | Réseau global | Description |
|---|---|---|---|
| Siège (A) | `siege01` | `10.10.0.0/16` | Site principal |
| Agence (B) | `agence01` | `10.20.0.0/16` | Antenne régionale |
| Transit VPN | – | `10.99.99.0/30` | Lien WireGuard site-à-site |

### Convention d'extension (sites futurs)

Le plan est dimensionné pour accueillir des sites supplémentaires sans refonte. Convention de plages :

| Site | Plage IP réservée | État |
|---|---|---|
| `siege01` | `10.10.0.0/16` | actif |
| `agence01` | `10.20.0.0/16` | actif |
| `agence02` | `10.30.0.0/16` | réservé (extension future) |
| `agence03` | `10.40.0.0/16` | réservé (extension future) |
| `agence04` | `10.50.0.0/16` | réservé (extension future) |
| ... | `10.<10 + 10*N>.0.0/16` | – |

Formule générique : pour `agenceNN`, la plage est `10.<10 + 10*N>.0.0/16` (donc `agence01` = 10.20, `agence02` = 10.30, etc.).

Capacité théorique : jusqu'à 24 agences avant de saturer la plage `10.0.0.0/8` selon cette convention.

---

## VLANs Site `siege01` (siège)

| VLAN | CIDR | Usage | Gateway |
|---|---|---|---|
| 10 | `10.10.10.0/24` | LAN users | `10.10.10.1` |
| 20 | `10.10.20.0/24` | LAN serveurs | `10.10.20.1` |
| 30 | `10.10.30.0/24` | DMZ | `10.10.30.1` |
| 40 | `10.10.40.0/24` | MGMT (out-of-band) | `10.10.40.1` |
| 50 | `10.10.50.0/24` | Wi-Fi corp | `10.10.50.1` |
| 60 | `10.10.60.0/24` | Wi-Fi guest | `10.10.60.1` |

## VLANs Site `agence01` (agence)

| VLAN | CIDR | Usage | Gateway |
|---|---|---|---|
| 10 | `10.20.10.0/24` | LAN users | `10.20.10.1` |
| 20 | `10.20.20.0/24` | LAN serveurs | `10.20.20.1` |
| 40 | `10.20.40.0/24` | MGMT | `10.20.40.1` |
| 50 | `10.20.50.0/24` | Wi-Fi corp | `10.20.50.1` |

## Asymétrie des segments entre sites

L'agence dispose de **moins de segments** que le siège. Ce choix reflète une asymétrie fonctionnelle classique en PME multi-sites :

- **Pas de DMZ à l'agence** : les services exposés sur internet (site web public, applications accessibles depuis l'extérieur) sont **mutualisés au siège**. Cela réduit la surface d'attaque, simplifie la supervision et concentre l'effort de durcissement sur un seul point d'exposition. L'agence consomme ces services centraux, elle ne les héberge pas.

- **Pas de Wi-Fi guest à l'agence** : peu de visiteurs externes sont attendus sur une antenne régionale de 10 personnes. Si le besoin émerge plus tard, le segment pourra être ajouté en réservant le CIDR `10.20.60.0/24`.

Cette asymétrie est **assumée** : tous les sites n'ont pas vocation à être identiques, leurs rôles fonctionnels diffèrent. Elle pourra être levée en phase 2 si la fiction du projet évolue (ex: l'agence devient un site équivalent au siège suite à une fusion).

## IPs réservées

> Cette table liste les IPs des services en **phase 1 (MVP)**. Tous les services ne sont pas présents sur les deux sites : la répartition reflète une **centralisation classique en PME**, où la plupart des services sont au siège, avec quelques composants (BIND9 secondaire, NetBox) répartis à l'agence pour équilibrer la charge entre les deux membres du binôme et démontrer une vraie répartition multi-sites.
>
> La haute disponibilité (2e DC, BIND secondaire au siège, Prometheus fédéré, etc.) est explicitement reportée en **phase 2 / bonus**.

| Service | IP | Site |
|---|---|---|
| pfSense `siege01` | `10.10.40.1` | siege01 |
| pfSense `agence01` | `10.20.40.1` | agence01 |
| AD DC1 | `10.10.20.10` | siege01 |
| BIND9 (agence) | `10.20.20.10` | agence01 |
| Traefik | `10.10.20.20` | siege01 |
| Docker host | `10.10.20.30` | siege01 |
| Prometheus | `10.10.20.40` | siege01 |
| NetBox | `10.20.20.20` | agence01 |


## Transit VPN

- Endpoints WireGuard sur le réseau `10.99.99.0/30` (point-à-point /30, 2 IPs utilisables)
- `10.99.99.1` : pfSense `siege01`
- `10.99.99.2` : pfSense `agence01`

## DNS

- Domaine AD : `corp.acme.local`
- Domaine externe (DMZ) : `acme.example` (à acquérir si démo publique)
- Split-horizon : vue interne (LAN) ↔ vue externe (DMZ uniquement)

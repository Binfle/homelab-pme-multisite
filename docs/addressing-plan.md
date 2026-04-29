# Plan d'adressage

> *Hypothèse de départ — à valider et ajuster collectivement.*

## Sites

| Site | Réseau global | Description |
|---|---|---|
| Siège (A) | `10.10.0.0/16` | Site principal |
| Agence (B) | `10.20.0.0/16` | Antenne régionale |
| Transit VPN | `10.99.99.0/30` | Lien WireGuard site-à-site |

## VLANs Site A (siège)

| VLAN | CIDR | Usage | Gateway |
|---|---|---|---|
| 10 | `10.10.10.0/24` | LAN users | `10.10.10.1` |
| 20 | `10.10.20.0/24` | LAN serveurs | `10.10.20.1` |
| 30 | `10.10.30.0/24` | DMZ | `10.10.30.1` |
| 40 | `10.10.40.0/24` | MGMT (out-of-band) | `10.10.40.1` |
| 50 | `10.10.50.0/24` | Wi-Fi corp | `10.10.50.1` |
| 60 | `10.10.60.0/24` | Wi-Fi guest | `10.10.60.1` |

## VLANs Site B (agence)

| VLAN | CIDR | Usage | Gateway |
|---|---|---|---|
| 10 | `10.20.10.0/24` | LAN users | `10.20.10.1` |
| 20 | `10.20.20.0/24` | LAN serveurs | `10.20.20.1` |
| 40 | `10.20.40.0/24` | MGMT | `10.20.40.1` |
| 50 | `10.20.50.0/24` | Wi-Fi corp | `10.20.50.1` |

## IPs réservées

| Service | IP | Site |
|---|---|---|
| pfSense A | `10.10.40.1` | A |
| pfSense B | `10.20.40.1` | B |
| AD DC1 | `10.10.20.10` | A |
| BIND9 (agence) | `10.20.20.10` | B |
| Traefik | `10.10.20.20` | A |
| Docker host | `10.10.20.30` | A |
| Prometheus | `10.10.20.40` | A |
| NetBox | `10.20.20.20` | B |

## DNS

- Domaine AD : `corp.acme.local`
- Domaine externe (DMZ) : `acme.example` (à acquérir si démo publique)
- Split-horizon : vue interne (LAN) ↔ vue externe (DMZ uniquement)

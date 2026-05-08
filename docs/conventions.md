# Conventions

> Document vivant rassemblant les conventions de nommage, formatage et standards récurrents du projet. À enrichir au fil des sprints. Toute convention ici a été prise par décision du binôme (parfois tracée dans un ADR pour les sujets structurants).

---

## 1. Nommage des sites

Format : `<role><NN>` en minuscules, sans séparateur entre rôle et numéro.

- `<role>` : `siege` ou `agence`
- `<NN>` : numéro à 2 chiffres avec préfixe `0` obligatoire pour le tri alphabétique (`01`, `02`, …)

**Exemples** : `siege01`, `agence01`, `agence02` (futur).

**Source** : `addressing-plan.md` § "Convention de nomenclature".

Dans la **prose et la documentation narrative**, on continue d'écrire "le Siège" et "l'Agence" en français lisible. La convention technique vaut pour les identifiants (réseaux, VMs, groupes Ansible, fichiers de config).

---

## 2. Nommage des segments réseau (VLANs)

Format dans la **prose et la documentation** : kebab-case minuscule avec tiret.

- `lan-usr` : LAN utilisateurs
- `lan-srv` : LAN serveurs
- `dmz` : DMZ
- `mgmt` : MGMT (administration)
- `wifi-corp`, `wifi-guest` : Wi-Fi (futur)

Format dans la **configuration pfSense** (Description des interfaces, règles firewall, etc.) : underscore au lieu de tiret.

- `LAN_USR`, `LAN_SRV`, `DMZ`, `MGMT`

**Pourquoi cette différence ?** pfSense **n'accepte pas le tiret `-`** dans le champ Description des interfaces (testé en pratique sur pfSense 2.8.1 lors du Sprint 1). L'underscore `_` est accepté et reste lisible. Donc on transcrit `lan-usr` (doc) en `LAN_USR` (pfSense).

**Implication** : tous les éléments connexes doivent suivre la même convention `LAN_USR` (avec underscore) côté pfSense :

| Endroit | Format |
|---|---|
| Description de l'interface | `LAN_USR` |
| Hostname DHCP / Domain name | si possible `LAN_USR` (sinon adapter au cas par cas) |
| Description des règles firewall | `Block LAN_USR to MGMT` |
| Aliases | `LAN_USR_subnets` |

---

## 3. Convention IP

### Adressage par site

Format général : `10.<2e_octet>.<3e_octet>.<4e_octet>`

- 2e octet : numéro de site (siege01 = 10, agence01 = 20, agence02 = 30, …)
- 3e octet : numéro du VLAN (LAN_USR = 10, LAN_SRV = 20, DMZ = 30, MGMT = 40, …)
- 4e octet : adresse hôte dans le sous-réseau

### Adresses spéciales par sous-réseau /24

- **`.0`** : adresse réseau (network address) — non utilisable
- **`.1`** : gateway — toujours pfSense par convention dans ce projet
- **`.2 → .99`** : réservé pour IPs statiques (serveurs, périphériques fixes)
- **`.100 → .200`** : pool DHCP (sur les VLANs où DHCP est activé, c'est-à-dire LAN_USR uniquement en phase 1)
- **`.201 → .254`** : libre, réserve d'IPs statiques additionnelles
- **`.255`** : adresse de broadcast — non utilisable

**Source** : `addressing-plan.md` + ADR-005.

---

## 4. Nommage des VMs

Convention adoptée pour le projet, à étendre au fur et à mesure de la création de nouvelles familles de VMs (Sprint 2+) :

- pfSense : `pfsense-<site>` → `pfsense-siege01`, `pfsense-agence01`
- AD Domain Controller : `dc<NN>-<site>` → `dc01-siege01` (Sprint 2)
- Serveur Linux générique : `<role>-<site>` → `bind9-agence01`, `traefik-siege01`, `netbox-agence01`
- Postes clients de test : `client-<role>-<site>-<NN>` → `client-debian-agence01-01`

---

## 5. Description des règles firewall

Format : `<Action> <Source> to <Destination> (<commentaire optionnel>)`

**Exemples** :

- `Block LAN_USR to MGMT (no admin access from users)`
- `Block LAN_USR to DMZ (HTTP/HTTPS pass à ajouter quand services en DMZ)`
- `Allow LAN_USR to any`
- `Block DMZ to LAN_SRV (Pass à ajouter sur ports spécifiques quand services en place)`

**Pourquoi un format normalisé ?** Faciliter la lecture par un binôme/recruteur, et permettre la review croisée (chacun comprend immédiatement ce que fait la règle sans relire les colonnes Source/Destination).

---

## 6. Convention de commits Git

Format : **Conventional Commits**.

```
<type>(<scope>): <description>
```

**Types** : `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`, `style`, `perf`.

**Exemples** :

- `feat(pfsense): add VLAN segmentation for agence01`
- `fix(dns): resolve BIND9 zone transfer error`
- `docs(adr): add ADR-005 on VLAN segmentation`
- `chore: bootstrap repo with sprint-0 scaffold`

**Source** : `CONTRIBUTING.md` du repo.

---

## 7. Format des branches Git

Format : `<type>/<sujet-en-kebab-case>`.

**Exemples** :

- `feat/wireguard-site-to-site`
- `fix/dns-agence`
- `docs/adr-005-vlan`
- `chore/sprint-0-bootstrap`

**Source** : `CONTRIBUTING.md`.

---

## 8. Format des CR de réunion

Stockés dans `docs/meetings/AAAA-MM-JJ-<sujet>.md`.

**Exemples** :

- `2026-04-30-cloture-sprint-0.md`
- `2026-05-15-decision-monitoring.md`

Modèle dans `docs/meetings/template.md`.

**Source** : `process.md` § 2.4.

---

## 9. Numérotation des ADR

- Format : `<NNN>-<titre-court-en-kebab-case>.md` dans `docs/adr/`.
- Numérotation séquentielle simple, ne reflète pas l'ordre chronologique strict (un numéro peut être réservé en planification avant d'être effectivement rédigé).
- Exemple : `005-segmentation-vlan-et-flux.md`.

**Source** : `docs/adr/README.md`.

---

## 10. Pré-requis Hyper-V (postes du binôme)

Pour qu'un PC du binôme puisse héberger son site, vérifier :

- Windows 11 **Pro** (Famille ne supporte pas Hyper-V).
- Virtualisation matérielle activée dans le BIOS (VT-x / AMD-V).
- Hyper-V activé dans `Fonctionnalités Windows`.
- Au moins **100 Go** d'espace libre sur le SSD système.
- Au moins **32 Go** de RAM (recommandé pour héberger toutes les VMs d'un site simultanément).

**Source** : ADR-007.

---

## Notes

Ce fichier est **vivant** : à mettre à jour à chaque convention nouvelle ou révisée, dans la même PR que la décision concernée. Les modifications doivent être tracées dans le commit message (ex : `docs(conventions): add VM naming convention`).

Si une convention est suffisamment structurante pour mériter un ADR (impact sur l'archi, sur la stack, sur la méthode), elle est tracée dans un ADR ET référencée ici.

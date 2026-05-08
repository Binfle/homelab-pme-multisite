# ADR-005 — Segmentation par VLANs taggés et matrice de flux inter-VLAN

**Statut** : proposé
**Date** : 07/05/2026
**Remplace** : néant
**Remplacé par** : néant

---

## Contexte

Le Sprint 1 (réseau bas-niveau) impose de segmenter les réseaux internes de chaque site selon les zones définies dans `addressing-plan.md` (LAN_USR, LAN_SRV, DMZ, MGMT). Plusieurs choix techniques se sont posés, chacun ayant des implications structurantes pour la suite du projet :

1. **Mode de segmentation** : réseau plat unique, réseaux physiques séparés, ou VLANs 802.1Q taggés sur un trunk unique.
2. **Statut du VLAN MGMT** : tagué (cohérent avec les autres VLANs) ou non-tagué/natif (plus simple à mettre en place).
3. **Matrice de flux inter-VLAN** : qui peut initier des connexions vers qui, et dans quelle mesure on applique le principe de moindre privilège.
4. **Niveau de granularité des règles firewall** : règles "any to any" permissives (rapides) versus règles spécifiques par destination (plus sûres mais plus verbeuses).

Contraintes pratiques :

- Hyper-V (Windows 11 Pro) sur les deux sites en phase 1 (cf ADR-007). Un seul vSwitch par site, support natif du tagging 802.1Q via `Set-VMNetworkAdapterVlan`.
- pfSense est le routeur/firewall central de chaque site et supporte nativement les sous-interfaces VLAN sur ses ports physiques.
- La DMZ d'agence01 est conservée à des fins pédagogiques jusqu'à la fin du projet (cf note dans `addressing-plan.md`), bien qu'elle ne soit pas dans le périmètre cible.

---

## Décision

### 1. Segmentation par VLANs 802.1Q taggés sur trunk unique

Tous les VLANs (10, 20, 30, 40) sont **taggués** et passent par le même trunk Hyper-V vers pfSense. Configuration côté Hyper-V :

```powershell
Set-VMNetworkAdapterVlan -VMName "pfsense-agence01" `
  -VMNetworkAdapterName "LAN" `
  -Trunk -AllowedVlanIdList "10,20,30,40" -NativeVlanId 0
```

Côté pfSense, chaque VLAN est créé comme sous-interface de l'interface physique LAN (ex : `hn1.10`, `hn1.20`, `hn1.30`, `hn1.40` sur siege01 / agence01).

### 2. MGMT taggué (VLAN 40)

Le réseau MGMT est **taggué au même titre que les autres VLANs**, pas en natif/non-tagué. Les postes administrateurs (Windows hôte côté homelab, équipements physiques côté production future) sont configurés en **Access VLAN 40** sur leur interface réseau.

### 3. Matrice de flux inter-VLAN (cible)

La matrice ci-dessous décrit qui peut **initier** une connexion vers qui. Le firewall pfSense étant **stateful**, le retour d'une connexion autorisée est implicitement autorisé sans règle inverse.

| Source ↓ \ Destination → | LAN_USR | LAN_SRV | DMZ | MGMT | Internet |
|---|---|---|---|---|---|
| **LAN_USR** | (intra) | O Pass | X Block (sauf 80/443 si DMZ peuplée) | X Block | O Pass |
| **LAN_SRV** | O Pass | (intra) | O Pass (à raffiner sur ports spécifiques) | X Block | O Pass |
| **DMZ** | X Block | X Block (Pass à ajouter sur ports spécifiques quand services en place) | (intra) | X Block | O Pass |
| **MGMT** | O Pass | O Pass | O Pass | (intra) | O Pass |

**Lecture** : "O Pass LAN_USR → LAN_SRV" signifie qu'un poste utilisateur peut initier une connexion vers un serveur interne (par exemple pour s'authentifier auprès de l'AD ou consulter un partage de fichiers). Le retour est automatique grâce au stateful firewall.

**Principes structurants** :

- **MGMT en réception** : isolé. Aucun autre VLAN ne peut initier vers MGMT. Seuls les administrateurs (qui sont eux-mêmes dans MGMT) peuvent atteindre MGMT, et depuis MGMT ils peuvent atteindre toutes les autres zones pour faire leur travail.
- **DMZ en émission** : isolée. La DMZ ne peut pas initier de connexion vers les zones internes (sauf règles spécifiques sur ports métiers à ajouter quand des services seront en place). Si la DMZ est compromise, l'attaquant ne peut pas pivoter en interne.
- **LAN_USR** : peut sortir librement (Internet, LAN_SRV) mais bloqué vers MGMT et DMZ.
- **LAN_SRV** : peut sortir partout sauf MGMT. Cohérent avec les besoins légitimes des serveurs (monitoring sortant, sauvegarde, déploiement vers DMZ, mises à jour Internet).

### 4. Granularité des règles firewall

- **Default deny implicite** sur chaque interface (comportement natif pfSense). Toute interface OPT bloque tout par défaut tant qu'aucune règle Pass n'est créée.
- **Règles Block explicites** placées **en haut** de chaque interface pour matérialiser les exceptions ("Block X to Y").
- **Règles Pass à la fin** ("Allow X to any") pour le trafic légitime restant (typiquement Internet).
- **Pas de granularité par port** dans la matrice initiale : on commence par filtrer sur les sous-réseaux. Le raffinement par port sera fait service par service quand ils seront déployés (ex : ouverture du 443 vers la DMZ pour le reverse proxy quand Traefik sera installé).

### 5. Application miroir sur les deux sites

La même matrice s'applique aux deux sites avec les sous-réseaux respectifs :

- siege01 : `10.10.X.X` (LAN_USR=10, LAN_SRV=20, DMZ=30, MGMT=40, (BONUS) Wi-Fi corp/guest=50/60)
- agence01 : `10.20.X.X` (LAN_USR=10, LAN_SRV=20, DMZ=30 *temporaire*, MGMT=40, (BONUS) Wi-Fi corp=50)

**Note** : la DMZ d'agence01 est conservée pour pédagogie jusqu'à la fin du projet (cf `addressing-plan.md`). Elle suit la même politique firewall que la DMZ du siège, bien qu'elle ne soit pas dans l'asymétrie cible.

---

## Conséquences

### Positives

- **Vraie isolation niveau 2 (Ethernet)** entre les zones. Sans VLAN, un appareil compromis pourrait attaquer ses voisins en local (ARP poisoning, sniffing, broadcasts) sans que le firewall n'en voie rien.
- **Conforme aux pratiques d'entreprise** : segmentation VLAN + filtrage stateful inter-VLAN est le pattern standard documenté (NIST 800-41, Cisco SBA, et la majorité des architectures Zero Trust modernes).
- **Posture least-privilege par défaut** : chaque VLAN ne peut atteindre que ce qui est explicitement nécessaire à son rôle.
- **Symétrie entre sites** : la même matrice s'applique des deux côtés, simplifie la duplication confrontante et la review croisée.
- **Évolutivité** : ajouter un nouveau VLAN consiste à créer un nouvel onglet de règles avec sa propre politique, sans toucher aux VLANs existants.
- **Support de la pédagogie** : la matrice est explicite, peut être expliquée à un recruteur ou à un nouvel arrivant.

### Négatives

- **Complexité de configuration** : trunk Hyper-V + VLANs pfSense + règles firewall par interface = plusieurs couches à coordonner. Une erreur dans une seule couche peut couper la connectivité.
- **Plus de règles à maintenir** : ajouter un service nécessite d'ajouter une règle Pass spécifique. Risque d'oubli ou de règle mal placée dans l'ordre.
- **Risque de blocage opérationnel** : default-deny implique que toute interface OPT bloque tout tant qu'aucune règle Pass n'a été créée. Si on oublie une règle, le service est bloqué et on ne le voit pas tout de suite (pas d'erreur, juste pas de réponse).

### Risques

- **Désynchronisation entre sites** : si l'un des deux ne respecte pas la matrice, le tunnel WireGuard peut acheminer du trafic qui sera bloqué côté destination, donnant l'illusion d'une connectivité partielle.

---

## Alternatives écartées

### Réseau plat unique (pas de segmentation)

- **Écartée** : sécurité quasi nulle. Un appareil compromis peut attaquer tous les autres en local. Aucune segmentation par rôle métier.
- Non viable même pour un lab pédagogique de PME.

### Réseaux physiques séparés (un vSwitch par zone)

- **Écartée** : nécessiterait plusieurs vSwitch et plusieurs cartes virtuelles par VM côté Hyper-V. Sur-complexe pour un lab à deux sites.
- N'apporte rien de plus que les VLANs en termes de sécurité (les VLANs taggés offrent une isolation niveau 2 réelle dans Hyper-V).

### MGMT non-tagué (VLAN natif)

- **Écartée** : plus simple à mettre en place mais incohérent avec les pratiques d'entreprise où **tous** les VLANs sont taggés (incluant MGMT) pour des raisons de rigueur de configuration et de cohérence opérationnelle.
- Aurait obligé à un traitement spécial de MGMT par rapport aux autres VLANs, source de confusion.
- Discussion en équipe au début du Sprint 1 a tranché en faveur du tagging, considéré comme plus formateur.

### VLANs sans firewall inter-VLAN (segmentation niveau 2 seule)

- **Écartée** : la segmentation niveau 2 isole les domaines de broadcast, mais sans règles firewall, **rien ne route entre les VLANs**. Les utilisateurs ne pourraient pas accéder aux serveurs ni à Internet. Inutilisable.
- Le firewall stateful pfSense est indispensable pour autoriser sélectivement les flux légitimes.

### Règles "any to any" permissives sur tous les VLANs

- **Écartée** : posture de sécurité faible, et signal négatif sur un portfolio sysadmin. Une règle "Pass LAN_USR to any" autorise implicitement LAN_USR à joindre MGMT (parce que `any` inclut RFC1918), ce qui n'est pas voulu.
- La matrice définie ici demande seulement quelques règles Block supplémentaires par interface — coût marginal pour un gain de sécurité majeur.

### Granularité par port dès maintenant

- **Reportée**, pas écartée. Pour cette première itération, on reste au niveau sous-réseau (Source / Destination = subnets). Le raffinement par port viendra service par service quand ils seront déployés (Sprint 2 : AD, BIND9 ; Sprint 3 : DMZ peuplée).
- Définir tous les ports en amont aurait alourdi le sprint sans valeur immédiate (les services ne sont pas encore là).

---

## Conditions de révision

Cet ADR sera révisé si :

- Une nouvelle zone (Wi-Fi corp, Wi-Fi guest, IoT) est ajoutée et nécessite une politique distincte.
- Le passage à un firewall non-pfSense (ex : OPNsense, dispositif physique) impose des contraintes différentes.
- Un retour d'expérience opérationnel (incident, audit) montre que la matrice est insuffisante ou trop restrictive.
- Le passage en phase 2 (Proxmox + cluster) modifie la topologie réseau sous-jacente.

Le cas échéant, un ADR remplaçant celui-ci sera rédigé.

# Procédure d'installation — pfSense (siege01 / agence01)

## Objet

Procédure d'installation initiale de **pfSense Community Edition** sur **VMware Workstation Pro**, dans le cadre du projet `homelab-pme-multisite`.

Ce guide couvre les **deux sites** (siege01 et agence01) via un système de variables. La procédure est identique, seules les valeurs (IPs, sous-réseaux, VMnet) changent selon le site.

## Public visé

Membre du binôme déployant pfSense côté siège ou côté agence.

## Pré-requis

| Pré-requis | Détail |
|---|---|
| Hyperviseur | VMware Workstation Pro 17.x ou supérieur (gratuit pour usage personnel) |
| Connexion réseau | **Filaire (Ethernet) recommandé**. Wi-Fi possible mais peut nécessiter un fallback NAT |
| Espace disque | ~25 Go libres pour la VM |
| Privilèges Windows | Identifiants admin pour modifier les VMnet (Éditeur de réseau virtuel) |
| ISO pfSense | `netgate-installer-vX.X.X-RELEASE-amd64.iso` téléchargé depuis https://www.pfsense.org/download/ et **décompressé** (.iso, pas .iso.gz) |
| Accès box ISP | Capacité à consulter la liste des appareils connectés (validation du bridge) |

## Variables selon le site

| Variable | Valeur siege01 | Valeur agence01 |
|---|---|---|
| `<SITE>` | siege01 | agence01 |
| `<NOM_VM>` | pfsense-siege01 | pfsense-agence01 |
| `<VMNET_LAN>` | VMnet2 | VMnet1 |
| `<SOUS_RESEAU_LAN>` | 10.10.40.0/24 | 10.20.40.0/24 |
| `<IP_LAN_PFSENSE>` | 10.10.40.1 | 10.20.40.1 |
| `<DHCP_RANGE_START>` | 10.10.40.100 | 10.20.40.100 |
| `<DHCP_RANGE_END>` | 10.10.40.199 | 10.20.40.199 |
| `<IP_WINDOWS_VMNET>` | 10.10.40.2 | 10.20.40.2 |
| `<HOSTNAME>` | pfsense-siege01 | pfsense-agence01 |
| `<DOMAINE>` | corp.acme.local | corp.acme.local |

 **Précision** : les sous-réseaux des deux sites correspondent au **VLAN MGMT (40)** prévu dans le plan d'adressage. Le VLAN sera configuré ultérieurement, pour l'instant on utilise directement ce sous-réseau comme LAN principal.

---

## Phase 1 — Configuration matérielle de la VM

### 1.1 — Création de la VM

1. **VMware → Fichier → Nouvelle machine virtuelle**
2. **Personnalisée (avancée)** → Suivant
3. **Compatibilité matérielle** : Workstation 17.x ou la plus récente → Suivant
4. **Installation du système d'exploitation** : choisir **"J'installerai le système d'exploitation ultérieurement"** → Suivant
5. **Système d'exploitation invité** :
   - Type : **Autre** (ou Linux selon les versions)
   - Version : **FreeBSD 14 64-bit**
6. **Nom de la machine virtuelle** : `<NOM_VM>`
7. **Emplacement** : choisir un dossier dédié (ex: `D:\VMs\<NOM_VM>\`)
8. **Configuration des processeurs** : 1 socket × 2 cœurs = 2 vCPU
9. **Mémoire** : 1024 Mo (1 Go)
10. **Type de réseau** : sera reconfigué après
11. **Type de contrôleur SCSI** : LSI Logic
12. **Type de disque** : SCSI
13. **Disque virtuel** : Créer un nouveau disque virtuel
14. **Capacité du disque** : 20 Go, "Stocker comme un seul fichier"
15. **Fichier de disque** : laisser le défaut

### 1.2 — Personnalisation du matériel

Avant de cliquer **Terminer**, cliquer sur **"Personnaliser le matériel..."** :

- **Carte réseau** (déjà présente) : laisser en **Pont (Automatique)** pour l'instant (sera ajusté en phase 2)
- **Ajouter** → **Carte réseau** → mode **Personnalisé : `<VMNET_LAN>`**
- **Lecteur CD/DVD** : sélectionner **"Utiliser un fichier ISO"** → Parcourir → choisir l'ISO pfSense décompressé. Cocher **"Connecté"** et **"Se connecter lors de la mise sous tension"**
- **Supprimer** (optionnel) : carte son, contrôleur USB

Cliquer **Fermer** puis **Terminer**.

---

## Phase 2 — Pré-configuration réseau VMware

**Phase Critique**. Elle prépare l'environnement réseau pour que pfSense ait internet et que le LAN soit isolé proprement.

### 2.1 — Lancer VMware en administrateur

**Indispensable** pour modifier les VMnet :

1. Fermer VMware s'il est ouvert
2. Clic droit sur l'icône VMware Workstation Pro → **Exécuter en tant qu'administrateur**
3. Confirmer l'élévation Windows

### 2.2 — Configurer VMnet0 (Bridged pour le WAN)

Par défaut VMware met le bridge en "Automatic", qui peut sélectionner la mauvaise carte si plusieurs interfaces existent.

1. **Édition → Éditeur de réseau virtuel**
2. Sélectionner **VMnet0**
3. Type : **Pont (Bridged)**
4. **Ponté sur :** sélectionner **explicitement** la carte physique active :
   - Ethernet/CPL : la carte Ethernet (ex: `Realtek PCIe GbE Family Controller`)
   - Wi-Fi : la carte Wi-Fi (ex: `Intel Wi-Fi 6 AX201`)
   - **Ne pas laisser "Automatic"**
5. **Apply**

**Pour identifier ta carte active** : `cmd → ipconfig` puis chercher la carte avec ton IP de box ISP (typiquement 192.168.1.X).

### 2.3 — Configurer (ou créer) le VMnet pour le LAN

#### Si le VMnet n'existe pas (cas typique pour VMnet2)
1. Bouton **Modifier** puis **Éditeur de réseau virtuel**
2. Bouton **"Ajouter un réseau"**
3. Choisir **`<VMNET_LAN>`** (ex: VMnet2)
4. **OK**

#### Configurer le VMnet (existant ou nouveau)

1. Sélectionner **`<VMNET_LAN>`** dans la liste
2. Type : **Hôte uniquement (Host-only)**
3. Cocher **"Connecter un hôte et un adaptateur virtuel à ce réseau"**
4. **Décocher** "Utiliser le service DHCP local pour distribuer les adresses IP aux machines virtuelles"
   - **Raison** : pfSense fournit son propre DHCP sur sa LAN. Deux DHCP sur le même segment = comportement aléatoire.
5. **Adresse IP de sous-réseau** : `<SOUS_RESEAU_LAN>` (ex: `10.10.40.0`)
6. **Masque de sous-réseau** : `255.255.255.0`
7. **Apply**
8. **OK** pour fermer l'éditeur

Windows va créer la **carte virtuelle "VMware Network Adapter `<VMNET_LAN>`"** dans les secondes qui suivent.

---

## Phase 3 — Installation pfSense

### 3.1 — Démarrer la VM et booter sur l'ISO

1. Sélectionner la VM `<NOM_VM>` dans la bibliothèque
2. **Mettre sous tension cette machine virtuelle**
3. La VM boote sur l'ISO pfSense

### 3.2 — Installer pfSense

| Étape installer | Choix |
|---|---|
| Welcome screen | **Install** → OK |
| Copyright | **Accept** |
| Active Subscription | **Install CE** (Community Edition) |
| Software version | **Current Stable Version** (ex: 2.8.1) |
| Installation Options | OK (laisser le défaut) |
| Disk Selection | OK |
| ZFS Configuration | OK (laisser le défaut) |
| Confirmation | **Yes** |

### 3.3 — Connectivity Check (étape Netgate)

L'installer Netgate vérifie la connexion internet. Cela peut prendre quelques minutes.

**Si l'étape échoue** ("Cannot assign the network interface") : voir section **Erreur & solutions**.

### 3.4 — WAN Interface Assignment

**Choisir l'interface qui correspond à la carte Bridged** :

1. **em0** (la première carte ajoutée à la VM = WAN)
2. **Si doute sur l'ordre** :
   - VM → Paramètres → Carte réseau (la WAN/Bridge) → Avancé → noter la **MAC address**
   - Croiser avec la MAC affichée par pfSense dans la liste em0/em1
   - Choisir l'interface correspondant à la MAC du Bridge
3. OK

### 3.5 — WAN Network Mode Setup

Laisser les défauts :
- Interface Mode : **DHCP (client)**
- VLAN Settings : **VLAN Tagging disabled**
- Use local resolver : **false**

→ **Continue** → OK

### 3.6 — LAN Interface Assignment

Sélectionner l'interface LAN (**em1**, la 2e carte ajoutée à la VM).

→ OK

### 3.7 — LAN Network Mode Setup

Laisser les défauts (192.168.1.1/24). On modifiera l'IP en Phase 4 pour respecter le plan d'adressage du projet.

→ **Continue** → OK

**Pourquoi pas mettre directement la bonne IP ici** : par sécurité, on garde une procédure linéaire qui marche systématiquement. La modification post-install via le menu CLI est plus fiable.

### 3.8 — Reboot

**AVANT de cliquer Reboot** : **retirer l'ISO du lecteur** sinon la VM rebootera sur l'installer en boucle.

1. **VM → Paramètres → CD/DVD (IDE)**
2. **Décocher** "Connecté"
3. **Décocher** "Se connecter lors de la mise sous tension"
4. **OK**
5. Revenir dans la VM, **valider Reboot**

### 3.9 — Premier boot post-install

Après reboot, pfSense démarre et affiche le **menu CLI principal** avec :

```
WAN (wan) -> em0 -> v4/DHCP4: 192.168.X.Y/24    (IP attribuée par la box)
LAN (lan) -> em1 -> v4: 192.168.1.1/24            (IP par défaut, à modifier)
```

**Si pas d'IP sur WAN** : voir section **Erreurs & solutions**.

---

## Phase 4 — Modification de l'IP LAN

### Pourquoi cette étape

Par défaut, pfSense propose **192.168.1.1/24** sur le LAN. Si la box ISP est aussi en **192.168.1.0/24**, conflit invisible : pfSense ne distingue plus son WAN de son LAN, le routage est cassé.

On configure donc le LAN sur le sous-réseau cible du projet (`<SOUS_RESEAU_LAN>`), qui est **différent** du sous-réseau de la box.

### Procédure

Dans le menu CLI pfSense :

1. Choisir option **`2`) Set interface(s) IP address**
2. Choisir l'interface **LAN** (typiquement option `2` du sous-menu)
3. *Configure IPv4 address LAN interface via DHCP?* → **`n`**
4. *Enter the new LAN IPv4 address* → `<IP_LAN_PFSENSE>` (ex: `10.10.40.1`)
5. *Enter the new LAN IPv4 subnet bit count* → **`24`**
6. *For a LAN, press <ENTER> for none* (gateway) → **Entrée** (vide)
7. *Configure IPv6 address LAN interface via DHCP6?* → **`n`**
8. *Enter the new LAN IPv6 address* → **Entrée** (vide)
9. *Do you want to enable the DHCP server on LAN?* → **`y`**
10. *Enter the start address of the IPv4 client address range* → `<DHCP_RANGE_START>` (ex: `10.10.40.100`)
11. *Enter the end address of the IPv4 client address range* → `<DHCP_RANGE_END>` (ex: `10.10.40.199`)
12. *Do you want to revert to HTTP as the webConfigurator protocol?* → **`n`** (on garde HTTPS)

pfSense affiche : *"The IPv4 LAN address has been set to `<IP_LAN_PFSENSE>`/24"*. Le menu principal reflète le changement.

---

## Phase 5 — Accès au webGUI depuis le PC hôte

### 5.1 — Vérifier l'apparition de la carte VMnet sur Windows

```cmd
ipconfig
```

Chercher la carte **"VMware Network Adapter `<VMNET_LAN>`"** :
- Si IP en `169.254.X.X` : carte présente mais sans IP, à fixer (étape 5.2)
- Si IP en `<IP_LAN_PFSENSE>` : **conflit avec pfSense**, à corriger immédiatement (étape 5.2)
- Si la carte n'apparaît pas : VMnet pas créé/appliqué, retour Phase 2.3

### 5.2 — Configurer une IP statique sur la carte VMnet

**Indispensable** : Windows doit avoir une IP **différente** de pfSense sur le même sous-réseau.

1. `Touche Windows` → `ncpa.cpl` → Entrée
2. Trouver **"VMware Network Adapter `<VMNET_LAN>`"**
3. **Clic droit** → **Propriétés**
4. Sélectionner **Internet Protocol Version 4 (TCP/IPv4)** → **Propriétés**
5. Choisir **"Utiliser l'adresse IP suivante"** :
   - **Adresse IP** : `<IP_WINDOWS_VMNET>` (ex: `10.10.40.2`)
   - **Masque de sous-réseau** : `255.255.255.0`
   - **Passerelle par défaut** : (laisser **vide**)
6. **DNS** : "Obtenir automatiquement"
7. **OK** → **OK**

**Précision** : la passerelle est **vide** intentionnellement. Si on mettait pfSense comme gateway, Windows utiliserait pfSense pour son trafic internet, ce qui n'est pas le but (Windows garde son internet via la carte Ethernet principale).

### 5.3 — Tester la connectivité

```cmd
ping <IP_LAN_PFSENSE>
```

Doit répondre. Si timeout : voir **Erreurs & solutions**.

### 5.4 — Accès au webGUI

1. Ouvrir un navigateur
2. Aller sur `https://<IP_LAN_PFSENSE>` (ex: `https://10.10.40.1`)
3. Avertissement de sécurité (certificat auto-signé) → **Avancé** → **Continuer vers le site**
4. Page de login pfSense :
   - **Username** : `admin`
   - **Password** : `pfsense` (par défaut, sera changé au wizard)

---

## Phase 6 — Wizard initial pfSense

Au premier login, pfSense lance automatiquement le **Setup Wizard**.

### 6.1 — General Information

| Champ | Valeur |
|---|---|
| **Hostname** | `<HOSTNAME>` (ex: `pfsense-siege01`) |
| **Domain** | `<DOMAINE>` (ex: `corp.acme.local`) |
| **Primary DNS** | `1.1.1.1` (Cloudflare) |
| **Secondary DNS** | `1.0.0.1` |
| **Override DNS** | Décoché |

Plus tard, ces DNS publics seront remplacés par l'AD-DNS interne (Sprint 2).

### 6.2 — Time Server Information

| Champ | Valeur |
|---|---|
| **Time server hostname** | `pool.ntp.org` (par défaut) |
| **Timezone** | `Europe/Paris` |

### 6.3 — Configure WAN Interface

**POINT CRITIQUE** : tout en bas de la page WAN, **DÉCOCHER** la case **"Block RFC1918 Private Networks"**.

| Champ | Valeur |
|---|---|
| **Selected Type** | DHCP (déjà configuré) |
| **Block RFC1918 Private Networks** | **DÉCOCHÉ** |
| **Block bogon networks** | Coché |

**Pourquoi décocher RFC1918** : le WAN reçoit des paquets venant du sous-réseau privé de la box (192.168.1.X). Si on bloque, internet ne marche plus depuis pfSense. En production avec un vrai WAN public, on garderait coché.

### 6.4 — Configure LAN Interface

Vérifier (sans modifier) :

| Champ | Valeur |
|---|---|
| **LAN IP Address** | `<IP_LAN_PFSENSE>` |
| **Subnet Mask** | 24 |

### 6.5 — Set Admin WebGUI Password

**Critique** : changer le mot de passe par défaut.

| Champ | Valeur |
|---|---|
| **Admin Password** | (mot de passe fort) |
| **Admin Password AGAIN** | (le même) |

### 6.6 — Reload

pfSense applique tous les changements et redémarre les services. Tu seras déconnecté et redirigé vers la page de login.

### 6.7 — Re-login avec le nouveau mot de passe

- Username : `admin`
- Password : (le **nouveau** mot de passe défini en 6.5)

Tu arrives sur le **Dashboard pfSense** : install terminée. ✅

---

## Phase 7 — Snapshot de sécurité

1. Dans VMware, clic droit sur la VM `<NOM_VM>` dans la bibliothèque
2. **Snapshot → Take Snapshot...**
3. **Nom** : `pfSense X.Y.Z - wizard configuré`
4. **Description** : 
   ```
   Install pfSense terminée, wizard initial complété.
   - Hostname : <HOSTNAME>
   - WAN : DHCP via box ISP
   - LAN : <IP_LAN_PFSENSE>/24
   - DHCP LAN : <DHCP_RANGE_START> à <DHCP_RANGE_END>
   - Block RFC1918 décoché
   - Mot de passe admin changé
   ```
5. **Take Snapshot**

---

## Validation finale (Definition of Done)

L'install est considérée terminée quand **tous** ces critères sont validés :

- [ ] VM démarrée et stable, menu CLI accessible
- [ ] WAN actif avec une IP de la box (visible dans menu CLI : `v4/DHCP4: 192.168.X.Y/24`)
- [ ] LAN actif avec l'IP cible (visible dans menu CLI : `v4: <IP_LAN_PFSENSE>/24`)
- [ ] VM apparaît dans l'admin de la box ISP (clients DHCP, MAC commençant par `00:0c:29`)
- [ ] Ping pfSense → 8.8.8.8 réussi (option 7 du menu CLI)
- [ ] Ping Windows → pfSense LAN réussi
- [ ] Accès webGUI fonctionnel via `https://<IP_LAN_PFSENSE>`
- [ ] Login admin avec **nouveau** mot de passe (pas le défaut)
- [ ] Wizard initial complété (hostname, domaine, DNS, time, WAN, LAN, password)
- [ ] Block RFC1918 décoché côté WAN (vérifier dans Interfaces → WAN)
- [ ] Snapshot de la VM pris ("install propre")

---

## Erreurs connus & solutions

### Connectivity Check Netgate Installer plante

**Symptôme** : "Cannot assign the network interface" pendant l'install.

**Cause** : la VM n'a pas internet via WAN bridgé.

**Solutions** :
1. Vérifier que VMnet0 est sur la **bonne carte explicite** (pas Automatic) — cf Phase 2.2
2. Si Wi-Fi : tenter de basculer la carte WAN en NAT temporairement (VM Settings → Carte réseau → NAT). À rebasculer en Bridged après l'install.
3. Vérifier que la box ISP voit la VM dans ses clients DHCP (MAC `00:0c:29:XX:XX:XX`)

### Pas de gateway dans une VM cliente du LAN

**Symptôme** : "no default route was set" pendant un install Linux/Windows sur le LAN.

**Cause** : conflit DHCP VMware vs pfSense sur le VMnet.

**Solution** : décocher "Utiliser le service DHCP local" sur le VMnet concerné — cf Phase 2.3.

### Sous-réseau VMnet incompatible avec LAN pfSense

**Symptôme** : ping VM → pfSense LAN timeout.

**Cause** : VMnet par défaut souvent en `192.168.175.0/24`, ne match pas le LAN pfSense.

**Solution** : aligner le sous-réseau VMnet sur celui du LAN pfSense (`<SOUS_RESEAU_LAN>`) — cf Phase 2.3.

### Conflit IP Windows ↔ pfSense

**Symptôme** : `ERR_CONNECTION_REFUSED` sur `https://<IP_LAN_PFSENSE>`.

**Cause** : Windows a auto-attribué `<IP_LAN_PFSENSE>` (la même IP que pfSense) à sa carte VMnet.

**Solution** : forcer une IP statique différente sur la carte Windows (`<IP_WINDOWS_VMNET>`, ex: `.2` au lieu de `.1`) — cf Phase 5.2.

### Block RFC1918 coupe internet pfSense

**Symptôme** : pas d'internet via pfSense, ping vers 8.8.8.8 échoue.

**Cause** : "Block RFC1918 Private Networks" coché sur le WAN, mais le WAN est sur sous-réseau privé (192.168.1.X de la box).

**Solution** : décocher "Block RFC1918 Private Networks" dans le wizard ou via Interfaces → WAN après le wizard — cf Phase 6.3.

### Cohabitation de plusieurs pfSense sur la même machine

**Symptôme** : conflit de routage, ping qui marche pour un site mais pas l'autre.

**Cause** : deux pfSense sur le même VMnet ou sur des VMnet aux sous-réseaux qui se chevauchent.

**Solution** :
- 1 pfSense = 1 VMnet dédié
- Sous-réseaux différents (10.10.40.0/24 pour siege01, 10.20.40.0/24 pour agence01)
- 2 cartes Windows VMnet correspondantes, chacune avec sa propre IP statique

### Plus de mot de passe admin

**Symptôme** : impossible de se logger sur le webGUI.

**Solution** :
1. Dans le menu CLI pfSense, option `3` : Reset webConfigurator password
2. Suivre les instructions, le mot de passe redevient `pfsense`
3. Se reconnecter et **changer immédiatement** le mot de passe

---

## Liens et références

- [Site officiel pfSense](https://www.pfsense.org/)
- [Documentation pfSense](https://docs.netgate.com/pfsense/en/latest/)
- [Plan d'adressage du projet](../addressing-plan.md)
- [ADR-004 : Choix d'hyperviseur (VMware Workstation Pro)](../adr/004-choix-hyperviseur-phase-1.md)
- [ADR-005 : Segmentation par VLANs taggés](../adr/005-segmentation-vlan-802.1q.md) — à venir

---
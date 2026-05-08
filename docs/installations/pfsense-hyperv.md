# Procédure d'installation — pfSense sur Hyper-V (siege01 / agence01)

## Objet

Procédure d'installation initiale de **pfSense Community Edition** sur **Microsoft Hyper-V**, dans le cadre du projet `homelab-pme-multisite`.

Ce guide couvre les **deux sites** (siege01 et agence01) via un système de variables. La procédure est identique, seules les valeurs (IPs, sous-réseaux, vSwitch) changent selon le site.

Pour la procédure équivalente sur VMware Workstation Pro, voir [pfsense-vmware.md](pfsense-vmware.md).

## Pré-requis

| Pré-requis | Détail |
|---|---|
| OS | Windows 10/11 **Pro** ou Enterprise (Hyper-V indisponible sur Home) |
| Hyperviseur | **Hyper-V** activé (`Activer ou désactiver des fonctionnalités Windows` → Hyper-V) |
| Connexion réseau | **Filaire (Ethernet) recommandé**. Wi-Fi possible avec restrictions sur le mode External |
| Espace disque | ~25 Go libres pour la VM |
| Privilèges Windows | Identifiants admin pour activer Hyper-V et créer les vSwitch |
| ISO pfSense | `netgate-installer-vX.X.X-RELEASE-amd64.iso` téléchargé depuis https://www.pfsense.org/download/ et **décompressé** (.iso, pas .iso.gz) |
| Accès box ISP | Capacité à consulter la liste des appareils connectés (validation du bridge External) |

## Variables selon le site

| Variable | Valeur siege01 | Valeur agence01 |
|---|---|---|
| `<SITE>` | siege01 | agence01 |
| `<NOM_VM>` | pfsense-siege01 | pfsense-agence01 |
| `<VSWITCH_WAN>` | vSwitch-siege01-wan | vSwitch-agence01-wan |
| `<VSWITCH_LAN>` | vSwitch-siege01-lan | vSwitch-agence01-lan |
| `<SOUS_RESEAU_LAN>` | 10.10.40.0/24 | 10.20.40.0/24 |
| `<IP_LAN_PFSENSE>` | 10.10.40.1 | 10.20.40.1 |
| `<DHCP_RANGE_START>` | 10.10.40.100 | 10.20.40.100 |
| `<DHCP_RANGE_END>` | 10.10.40.199 | 10.20.40.199 |
| `<IP_WINDOWS_VETH>` | 10.10.40.2 | 10.20.40.2 |
| `<HOSTNAME>` | pfsense-siege01 | pfsense-agence01 |
| `<DOMAINE>` | corp.acme.local | corp.acme.local |

**Précision** : les sous-réseaux des deux sites correspondent au **VLAN MGMT (40)** prévu dans le plan d'adressage. Le VLAN sera configuré ultérieurement, pour l'instant on utilise directement ce sous-réseau comme LAN principal.

---

## Phase 1 — Pré-configuration des commutateurs virtuels Hyper-V

**Phase critique**. Elle prépare l'environnement réseau pour que pfSense ait internet et que le LAN soit isolé proprement.

### 1.1 — Vérifier qu'Hyper-V est activé

Si Hyper-V n'est pas activé :

1. `Touche Windows` → taper "Activer ou désactiver des fonctionnalités Windows"
2. Cocher **Hyper-V** (et toutes ses sous-cases qui se cochent automatiquement)
3. **OK** → redémarrer Windows

### 1.2 — Créer le vSwitch WAN (Externe)

1. **Gestionnaire Hyper-V → panneau Actions à droite → Gestionnaire de commutateur virtuel**
2. **Nouveau commutateur réseau virtuel**
3. Type : **Externe** → **Créer le commutateur virtuel**
4. **Nom** : `<VSWITCH_WAN>` (ex: `vSwitch-agence01-wan`)
5. **Type de connexion** : **Réseau externe** + sélectionner **explicitement** la carte Ethernet physique (ne pas laisser Automatic)
6. **Cocher** "Autoriser le système d'exploitation de gestion à partager cette carte réseau"
7. **Décocher** "Activer l'identification LAN virtuelle pour le système d'exploitation de gestion"
8. **Appliquer** + **OK**

Une **micro-coupure réseau** de quelques secondes est normale pendant la création du vSwitch External — Hyper-V réorganise la pile réseau. L'icône Windows peut afficher "Pas d'accès Internet" temporairement, à ignorer si `ping 8.8.8.8` répond.

### 1.3 — Créer le vSwitch LAN (Interne)

1. Toujours dans le Gestionnaire de commutateur virtuel
2. **Nouveau commutateur réseau virtuel**
3. Type : **Interne** → **Créer le commutateur virtuel**
4. **Nom** : `<VSWITCH_LAN>` (ex: `vSwitch-agence01-lan`)
5. **Type de connexion** : **Réseau interne**
6. **Décocher** "Activer l'identification LAN virtuelle"
7. **Appliquer** + **OK**

### 1.4 — Configurer le profil réseau de la carte vEthernet LAN

**Étape obligatoire après création du vSwitch LAN**. Windows classe automatiquement la nouvelle carte `vEthernet (<VSWITCH_LAN>)` en profil **Public**, ce qui bloque l'ICMP entrant. Sans cette étape, le ping pfSense → Windows échouera plus tard (Phase 5.3).

En **PowerShell admin** :

```powershell
Set-NetConnectionProfile -InterfaceAlias "vEthernet (<VSWITCH_LAN>)" -NetworkCategory Private
```

Vérifier avec :

```powershell
Get-NetConnectionProfile
```

`NetworkCategory` doit afficher **Private** pour la carte vEthernet correspondant au vSwitch LAN.

**Sur certaines configs Windows** (antivirus tiers, GPO restrictives), le profil Private ne suffit pas. Si plus tard le ping pfSense → Windows échoue malgré ce changement, créer en plus une règle pare-feu explicite :

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In Lab" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

### 1.5 — Ne pas utiliser le Default Switch

Hyper-V crée automatiquement un **Default Switch** au type NAT (équivalent VMnet8 NAT de VMware, **pas** VMnet0 Bridged). Ne pas l'utiliser pour pfSense — il fait du double-NAT et ne supporte pas le VLAN tagging.

---

## Phase 2 — Configuration matérielle de la VM

### 2.1 — Création de la VM via le wizard

1. **Gestionnaire Hyper-V → panneau Actions à droite → Nouveau → Ordinateur virtuel**
2. **Avant de commencer** → Suivant
3. **Nom** : `<NOM_VM>` (ex: `pfsense-agence01`). Emplacement : par défaut ou personnalisé
4. **Spécifier la génération** : **Génération 1**  (pas Génération 2 — incompatibilité historique avec FreeBSD)
5. **Affecter de la mémoire** :
   - Mémoire de démarrage : **1024 Mo**
   - **Décocher** "Utiliser la mémoire dynamique"  (instabilité connue avec pfSense)
6. **Configurer la mise en réseau** : sélectionner **`<VSWITCH_WAN>`**
   - Le wizard ne permet de configurer qu'**une seule carte**. La 2ème (LAN) sera ajoutée juste après.
7. **Connecter un disque dur virtuel** :
   - Créer un disque dur virtuel
   - Nom : `<NOM_VM>.vhdx`
   - Taille : **20 Go**
8. **Options d'installation** :
   - Cocher "Installer un système d'exploitation à partir d'un fichier image de démarrage"
   - **Parcourir** → sélectionner l'ISO pfSense (.iso, pas .iso.gz)
9. **Terminer**

### 2.2 — Ajouter la 2ème carte réseau (LAN)

 **Étape obligatoire avant de démarrer la VM**. Sans ça, l'installer pfSense ne verra qu'un WAN sans LAN et l'install est à refaire.

1. Dans Hyper-V Manager, **clic droit** sur la VM → **Paramètres**
2. Dans la liste à gauche, tout en haut : **Ajouter un matériel**
3. Sélectionner **Carte réseau** → **Ajouter**
4. Dans les propriétés à droite : **Commutateur virtuel** = `<VSWITCH_LAN>`
5. **Appliquer** + **OK**

La VM a maintenant 2 cartes réseau : une vers WAN (Externe), une vers LAN (Interne).

### 2.3 — Cartes réseau classiques vs héritées

Hyper-V propose deux types de cartes :
- **Carte réseau** (driver `hn` côté pfSense) : moderne, meilleures perfs
- **Carte réseau héritée** (driver `de` côté pfSense) : ancienne, compatibilité maximale

Sur les **versions récentes de pfSense (2.7+)**, les cartes réseau classiques **fonctionnent dès l'installer**.

Sur des versions plus anciennes ou en cas de blocage à l'étape Connectivity Check : basculer en **Carte réseau héritée** (Paramètres → Ajouter du matériel → Carte réseau héritée).

---

## Phase 3 — Installation pfSense

### 3.1 — Démarrer la VM et booter sur l'ISO

1. Hyper-V Manager → clic droit sur `<NOM_VM>` → **Démarrer**
2. Clic droit → **Connecter** (ouvre VMConnect, équivalent de la console VMware)

Garder la fenêtre VMConnect à sa taille par défaut. Redimensionner peut bloquer l'affichage de l'installer.

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

**Si l'étape échoue** ("Cannot assign the network interface") : voir section **Erreurs connues**.

### 3.4 — Interface Assignment

L'installer pfSense identifie les cartes par leur driver Hyper-V :
- **`hn0`** = première carte = WAN (sur `<VSWITCH_WAN>`)
- **`hn1`** = deuxième carte = LAN (sur `<VSWITCH_LAN>`)

Vérifier l'écran "Interface Assignment" : tu dois voir **les deux cartes**, pas une seule. Si LAN est `(not assigned)`, c'est que la 2ème carte n'a pas été ajoutée → arrêter, désactiver la VM, ajouter la carte (cf. Phase 2.2), recommencer.

→ **Continue** → OK

### 3.5 — WAN Network Mode Setup

Laisser les défauts :
- Interface Mode : **DHCP (client)**
- VLAN Settings : **VLAN Tagging disabled**
- Use local resolver : **false**

→ **Continue** → OK

### 3.6 — LAN Network Mode Setup

Laisser les défauts (192.168.1.1/24). On modifiera l'IP en Phase 4 pour respecter le plan d'adressage du projet.

→ **Continue** → OK

**Pourquoi pas mettre directement la bonne IP ici** : par sécurité, on garde une procédure linéaire qui marche systématiquement. La modification post-install via le menu CLI est plus fiable.

### 3.7 — Reboot — étape critique

Hyper-V est **strict** sur l'ordre de boot. Avant de cliquer Reboot dans l'installer :

1. **Garder VMConnect ouvert**
2. Aller dans Hyper-V Manager → clic droit sur la VM → **Paramètres**
3. **Lecteur DVD** (à gauche) → cocher **Aucun** → **Appliquer**
4. **BIOS** (à gauche) → sélectionner **IDE** → cliquer **Monter** jusqu'à mettre IDE en premier
   - L'ordre final doit être : `IDE` → `CD` → `Carte réseau` → `Disquette`
5. **Appliquer** + **OK**
6. Revenir dans VMConnect → cliquer **Reboot**

Possiblement, sans ces étapes, la VM reboote sur l'ISO et l'installer pfSense propose de réinstaller en boucle ("Do you want to restart the installation?").

### 3.8 — Premier boot post-install

Après reboot, pfSense démarre et affiche le **menu CLI principal** avec :

```
WAN (wan) -> hn0 -> v4/DHCP4: 192.168.X.Y/24    (IP attribuée par la box)
LAN (lan) -> hn1 -> v4: 192.168.1.1/24            (IP par défaut, à modifier)
```

**Si pas d'IP sur WAN** : voir section **Erreurs connues**.

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
4. *Enter the new LAN IPv4 address* → `<IP_LAN_PFSENSE>` (ex: `10.20.40.1`)
5. *Enter the new LAN IPv4 subnet bit count* → **`24`**
6. *For a LAN, press <ENTER> for none* (gateway) → **Entrée** (vide)
7. *Configure IPv6 address LAN interface via DHCP6?* → **`n`**
8. *Enter the new LAN IPv6 address* → **Entrée** (vide)
9. *Do you want to enable the DHCP server on LAN?* → **`y`**
10. *Enter the start address of the IPv4 client address range* → `<DHCP_RANGE_START>` (ex: `10.20.40.100`)
11. *Enter the end address of the IPv4 client address range* → `<DHCP_RANGE_END>` (ex: `10.20.40.199`)
12. *Do you want to revert to HTTP as the webConfigurator protocol?* → **`n`** (on garde HTTPS)

pfSense affiche : *"The IPv4 LAN address has been set to `<IP_LAN_PFSENSE>`/24"*. Le menu principal reflète le changement.

---

## Phase 5 — Accès au webGUI depuis le PC hôte

### 5.1 — Vérifier l'apparition de la carte vEthernet sur Windows

```cmd
ipconfig
```

Chercher la carte **"vEthernet (`<VSWITCH_LAN>`)"**. Hyper-V la crée automatiquement quand le vSwitch Interne est créé.

La carte côté Windows s'appelle `vEthernet (<nom-du-vSwitch>)`, pas `VMware Network Adapter VMnetX` comme chez VMware.

Si la carte n'apparaît pas : retour Phase 1.3 (vSwitch LAN pas créé).

### 5.2 — Configurer une IP statique sur la carte vEthernet

**Indispensable** : Windows doit avoir une IP **différente** de pfSense sur le même sous-réseau.

1. `Touche Windows` → `ncpa.cpl` → Entrée
2. Trouver **"vEthernet (`<VSWITCH_LAN>`)"**
3. **Clic droit** → **Propriétés**
4. Sélectionner **Internet Protocol Version 4 (TCP/IPv4)** → **Propriétés**
5. Choisir **"Utiliser l'adresse IP suivante"** :
   - **Adresse IP** : `<IP_WINDOWS_VETH>` (ex: `10.20.40.2`)
   - **Masque de sous-réseau** : `255.255.255.0`
   - **Passerelle par défaut** : (laisser **vide**)
6. **DNS** : "Obtenir automatiquement"
7. **OK** → **OK**

**Précision** : la passerelle est **vide** intentionnellement. Si on mettait pfSense comme gateway, Windows utiliserait pfSense pour son trafic internet, ce qui n'est pas le but (Windows garde son internet via la carte Ethernet principale).

### 5.3 — Tester la connectivité bidirectionnelle

Depuis Windows hôte :

```cmd
ping <IP_LAN_PFSENSE>
```

Doit répondre.

Depuis le menu CLI pfSense → option **7 (Ping host)** → `<IP_WINDOWS_VETH>` :

Doit répondre aussi.

Si le ping pfSense → Windows échoue alors que Windows → pfSense fonctionne : c'est le pare-feu Windows qui bloque l'ICMP entrant. Voir Phase 1.4 et la section **Erreurs connues**.

### 5.4 — Accès au webGUI

1. Ouvrir un navigateur
2. Aller sur `https://<IP_LAN_PFSENSE>` (ex: `https://10.20.40.1`)
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
| **Hostname** | `<HOSTNAME>` (ex: `pfsense-agence01`) |
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

Tu arrives sur le **Dashboard pfSense** : install terminée. 

---

## Phase 7 — Point de contrôle de sécurité

Hyper-V appelle "point de contrôle" ce que VMware appelle "snapshot".

1. Hyper-V Manager → clic droit sur `<NOM_VM>` → **Point de contrôle**
2. Le checkpoint est créé instantanément (visible dans la section "Points de contrôle" en bas)
3. Clic droit sur le checkpoint créé → **Renommer**
4. **Nom** : `pfSense X.Y.Z - wizard configuré (Hyper-V)`

---

## Validation finale (Definition of Done)

L'install est considérée terminée quand **tous** ces critères sont validés :

- [ ] VM démarrée et stable, menu CLI accessible
- [ ] WAN actif avec une IP de la box (visible dans menu CLI : `v4/DHCP4: 192.168.X.Y/24`)
- [ ] LAN actif avec l'IP cible (visible dans menu CLI : `v4: <IP_LAN_PFSENSE>/24`)
- [ ] VM apparaît dans l'admin de la box ISP (clients DHCP)
- [ ] Ping pfSense → 8.8.8.8 réussi (option 7 du menu CLI)
- [ ] Ping Windows → pfSense LAN réussi
- [ ] **Ping pfSense → Windows hôte réussi** (option 7 du menu CLI vers `<IP_WINDOWS_VETH>`)
- [ ] Accès webGUI fonctionnel via `https://<IP_LAN_PFSENSE>`
- [ ] Login admin avec **nouveau** mot de passe (pas le défaut)
- [ ] Wizard initial complété (hostname, domaine, DNS, time, WAN, LAN, password)
- [ ] Block RFC1918 décoché côté WAN (vérifier dans Interfaces → WAN)
- [ ] Point de contrôle Hyper-V pris ("install propre")

---

## Erreurs connues et solutions

### Boot failure après l'install ("Reboot and Select proper Boot device")

**Symptôme** : après le premier reboot post-install, message "Boot failure. Reboot and Select proper Boot device or Insert Boot Media".

**Causes possibles** :
1. ISO toujours montée et CD en premier dans l'ordre de boot → la VM reboote sur l'ISO
2. IDE pas en premier dans l'ordre de boot → la VM ne trouve pas le système installé
3. Install pfSense interrompue avant la fin

**Solutions** :
1. Désactiver la VM (clic droit → Désactiver, équivalent power off forcé)
2. Paramètres → Lecteur DVD → Aucun
3. Paramètres → BIOS → IDE en premier
4. Démarrer la VM

Si le problème persiste : refaire l'install depuis zéro.

### LAN apparaît "(not assigned)" à l'Interface Assignment

**Symptôme** : à l'écran "Interface Assignment" de l'installer, seul WAN est listé avec une carte. LAN dit `(not assigned)`.

**Cause** : une seule carte réseau ajoutée au moment du wizard de création de VM. Hyper-V ne permet pas d'ajouter plusieurs cartes via le wizard initial.

**Solution** : Désactiver la VM, ajouter une 2ème carte réseau via Paramètres → Ajouter un matériel → Carte réseau, redémarrer l'install.

### Échec de l'arrêt de la VM ("Le périphérique n'est pas prêt à être utilisé")

**Symptôme** : la commande Arrêter sur la VM échoue avec une erreur 0x800710DF.

**Cause** : la VM est dans un état "coincé" (BIOS qui boot loop, OS pas chargé, etc.). Arrêter demande un shutdown propre par l'OS invité, ce qui n'est pas possible dans cet état.

**Solution** : utiliser **Désactiver** au lieu d'Arrêter (équivalent power off forcé). Aucun risque tant qu'aucun OS n'est chargé.

Si Désactiver échoue aussi, en PowerShell admin :
```powershell
Stop-VM -Name "<NOM_VM>" -TurnOff -Force
```

### "Pas d'accès Internet" sur Windows après création du vSwitch External

**Symptôme** : icône réseau Windows affiche "Pas d'accès Internet" après création du vSwitch External.

**Cause** : artefact d'affichage Windows (NCSI - Network Connectivity Status Indicator). La connectivité réelle est OK.

**Solution** : ignorer l'icône, vérifier avec `ping 8.8.8.8`. Si ping répond, tout va bien. L'icône se met à jour dans 1-2 minutes.

### Ping pfSense → Windows hôte ne passe pas

**Symptôme** : depuis le menu CLI pfSense, option 7 (Ping host), le ping vers `<IP_WINDOWS_VETH>` retourne 100% packet loss. Cependant, l'accès à la webGUI depuis Windows fonctionne (login pfSense réussi visible dans les logs pfSense).

**Cause** : le pare-feu Windows bloque l'ICMP entrant sur la carte vEthernet, qui est classée par défaut comme "Réseau public". Le trafic Windows → pfSense fonctionne (stateful inspection autorise les réponses), mais pfSense → Windows est bloqué.

**Solution étape 1** : passer la carte en profil Private.

En PowerShell admin :

```powershell
# Vérifier l'état actuel
Get-NetConnectionProfile

# Changer le profil de la carte LAN en Privé
Set-NetConnectionProfile -InterfaceAlias "vEthernet (<VSWITCH_LAN>)" -NetworkCategory Private
```

Re-tester le ping. Sur la majorité des configs, l'étape 1 suffit.

**Solution étape 2 (si étape 1 ne suffit pas)** : créer une règle pare-feu explicite autorisant l'ICMP entrant.

Sur certaines configs Windows (antivirus tiers avec firewall intégré, GPO custom, règles Windows par défaut plus restrictives), le profil Private ne suffit pas. Dans ce cas :

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In Lab" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

Cette règle autorise spécifiquement les "Echo Request" entrants (type 8 d'ICMPv4 = ping) sur tous les profils.

**Ne pas désactiver le pare-feu Windows entièrement** comme solution. Ça réduit drastiquement la sécurité du PC hôte.

### Connectivity Check Netgate Installer plante

**Symptôme** : "Cannot assign the network interface" pendant l'install.

**Cause** : la carte réseau classique (driver `hn`) n'est pas reconnue par l'installer FreeBSD (rare sur pfSense 2.7+, plus fréquent sur versions antérieures).

**Solution** :
1. Désactiver la VM
2. Paramètres → supprimer les Cartes réseau classiques
3. Ajouter du matériel → **Cartes réseau héritées** (une par vSwitch, comme avant)
4. Reprendre l'install

### Conflit IP Windows ↔ pfSense

**Symptôme** : `ERR_CONNECTION_REFUSED` sur `https://<IP_LAN_PFSENSE>`.

**Cause** : la carte vEthernet Windows a auto-attribué `<IP_LAN_PFSENSE>` (la même IP que pfSense).

**Solution** : forcer une IP statique différente sur la carte vEthernet (`<IP_WINDOWS_VETH>`, ex: `.2` au lieu de `.1`) — cf Phase 5.2.

### Block RFC1918 coupe internet pfSense

**Symptôme** : pas d'internet via pfSense, ping vers 8.8.8.8 échoue.

**Cause** : "Block RFC1918 Private Networks" coché sur le WAN, mais le WAN est sur sous-réseau privé (192.168.1.X de la box).

**Solution** : décocher "Block RFC1918 Private Networks" dans le wizard ou via Interfaces → WAN après le wizard — cf Phase 6.3.

### Plus de mot de passe admin

**Symptôme** : impossible de se logger sur le webGUI.

**Solution** :
1. Dans le menu CLI pfSense, option `3` : Reset webConfigurator password
2. Suivre les instructions, le mot de passe redevient `pfsense`
3. Se reconnecter et **changer immédiatement** le mot de passe

---

## Liens et références

- [Site officiel pfSense](https://www.pfsense.org/)
- [Documentation pfSense sur Hyper-V](https://docs.netgate.com/pfsense/en/latest/virtualization/hyper-v.html)
- [Plan d'adressage du projet](../addressing-plan.md)
- [Procédure d'installation pfSense pour VMware Workstation](pfsense-vmware.md)
- [ADR-XXX : Choix d'hyperviseur (Hyper-V) — à venir](../adr/) (rédaction prévue en fin de Sprint 1)

---

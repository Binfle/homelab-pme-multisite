# Ansible — homelab-pme-multisite

> Configuration des machines Linux via Ansible. Sprint 1 — rôle `common` (bootstrap minimal).

## Structure

ansible/
├── ansible.cfg           # Configuration Ansible (chemins, callbacks, SSH)
├── inventory/
│   └── hosts.yml         # Inventaire (groupes siege01 + agence01)
├── playbooks/
│   └── site.yml          # Playbook principal — applique le rôle common
└── roles/
    └── common/           # Rôle de base, appliqué à toutes les VMs Linux

## Pré-requis (par contrôleur Ansible)

- Debian 13 ou équivalent, Ansible installé (`apt install ansible`)
- Clé SSH ed25519 du compte `sysadmin` du contrôleur (`~/.ssh/id_ed25519`), avec passphrase
- `ssh-agent` chargé avec cette clé avant chaque session :
```bash
  eval "$(ssh-agent -s)"
  ssh-add ~/.ssh/id_ed25519
```
- Accès SSH au moins une fois (pour valider la fingerprint) vers chaque cible

## Exécution

Toutes les commandes se lancent **depuis ce dossier `ansible/`** (à cause de la résolution du `ansible.cfg`).

### Tester l'inventaire
```bash
ansible-inventory --list
ansible all -m ping
```

### Lancer le playbook complet
```bash
ansible-playbook playbooks/site.yml
```

### Cibler un site seulement
```bash
ansible-playbook playbooks/site.yml -l agence01
ansible-playbook playbooks/site.yml -l siege01
```

## Rôle `common`

Bootstrap de base appliqué à toutes les VMs Linux. Au minimum :

- Mise à jour du cache `apt`
- Installation de paquets de base (vim, htop, curl, wget, git, net-tools, dnsutils)
- Définition du hostname (à partir de `inventory_hostname`)
- Création du compte `sysadmin` (groupe sudo, shell bash)
- Sudo NOPASSWD pour `sysadmin` (via `/etc/sudoers.d/sysadmin`, validé par `visudo -cf`)
- Dépôt de la clé SSH publique du contrôleur dans `~sysadmin/.ssh/authorized_keys`

### Variables par défaut

Voir `roles/common/defaults/main.yml`. Surchargeables via `group_vars/` ou `host_vars/`.

## Bootstrap initial

À la création d'une nouvelle VM, le user `sysadmin` n'existe pas encore. Pour le premier passage du rôle `common` :

1. La VM doit avoir un compte avec sudo (`zafl`, `debian`, etc. selon l'install) et SSH activé
2. Copier ta clé publique sur ce compte temporaire : `ssh-copy-id <user_initial>@<ip>`
3. Renseigner ce compte dans l'inventaire (`ansible_user: <user_initial>`)
4. Lancer **avec `-K`** pour demander le mdp sudo : `ansible-playbook playbooks/site.yml -K`
5. Une fois le rôle appliqué (`sysadmin` créé, NOPASSWD, clé déposée), basculer dans l'inventaire : `ansible_user: sysadmin`
6. Les runs suivants se font **sans `-K`**

## Modèle de sécurité — limites assumées

Le rôle `common` configure le compte `sysadmin` en sudo **NOPASSWD**. Ce choix est un compromis assumé entre :
- **Automatisation** : Ansible peut tourner sans intervention humaine, idempotence facile
- **Risque** : si le contrôleur Ansible est compromis pendant qu'une session ssh-agent tourne, l'attaquant a root sur toutes les cibles

Mesures compensatoires retenues :
- Contrôleur sur VLAN MGMT isolé (cf ADR-005)
- Clé SSH du contrôleur chiffrée par passphrase
- Connexion SSH au contrôleur lui-même par clé ed25519 + passphrase
- Snapshots Hyper-V réguliers

Améliorations possibles (phase 2 / bonus) :
- `auditd` côté cibles pour logger toutes les commandes sudo
- Restriction du NOPASSWD à des commandes spécifiques (complexe avec Ansible générique)
- Bascule sur Ansible Tower / AWX pour centraliser les credentials

## TODO

- [ ] Augustin : renseigner `client-debian-siege01-01` dans `inventory/hosts.yml`
- [ ] Compléter `roles/common/` v2 : SSH durci, fail2ban, timezone, locale
- [ ] Créer un rôle dédié `apt-upgrade` pour les mises à jour système
- [ ] Migrer le contrôleur Ansible d'agence01 de `.100` à `.10` après acceptation de la classification IPs MGMT par dizaines
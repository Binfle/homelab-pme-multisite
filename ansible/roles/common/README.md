# Rôle Ansible — common

## Description

Bootstrap minimal appliqué à toutes les VMs Linux du projet `homelab-pme-multisite`. Configure le hostname, crée le compte d'administration `sysadmin` avec sudo NOPASSWD et clé SSH, et installe quelques paquets de base.

## Variables (voir `defaults/main.yml`)

| Variable | Valeur par défaut | Description |
|---|---|---|
| `common_admin_user` | `sysadmin` | Nom du compte admin créé |
| `common_admin_groups` | `[sudo]` | Groupes auxquels le user est ajouté |
| `common_admin_shell` | `/bin/bash` | Shell par défaut |
| `common_admin_authorized_key` | `lookup ~/.ssh/id_ed25519.pub` | Clé SSH publique du contrôleur |
| `common_base_packages` | `vim, htop, curl, wget, git, net-tools, dnsutils` | Paquets installés |

## Idempotence

Toutes les tâches sont idempotentes (validé Sprint 1). Un second run consécutif retourne `changed=0`.

## Exemple d'usage

```yaml
- hosts: all
  become: true
  roles:
    - common
```

## Dépendances

Aucune.

## Limites (v1, Sprint 1)

- Pas de SSH durci (`PermitRootLogin`, `PasswordAuthentication` restent par défaut)
- Pas de fail2ban
- Pas de timezone / locale
- Voir `ansible/README.md` § "Modèle de sécurité" pour le compromis NOPASSWD assumé
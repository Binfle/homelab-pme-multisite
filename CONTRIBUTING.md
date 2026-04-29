# Contribuer

## Workflow Git
- Pas de commit direct sur `main`. Toujours via PR.
- Une branche par feature : `feat/wireguard-site-to-site`, `fix/dns-agence`, `docs/adr-004`.
- Review obligatoire avant merge.
- Squash & merge à la fusion.

## Conventions de commit
Format Conventional Commits :
```
<type>(<scope>): <description>
```
Types : `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`.

Exemples :
- `feat(ansible): rôle bind9 avec config split-horizon`
- `fix(pfsense): NAT sortante manquait pour VLAN serveurs`
- `docs(adr): ADR-004 sur le choix de la PKI`

## Pull Requests
- Titre préfixé : `[site-a] feat: ...`
- Description : quoi, pourquoi, comment tester
- Une PR = un sujet
- Touche un autre site → review du propriétaire de cet autre site

## Documentation
- Décision d'archi → ADR dans `docs/adr/`
- Procédure ops → runbook dans `docs/runbooks/`
- Cadre méthodologique général → [docs/process.md](docs/process.md)

## Secrets
Aucun secret en clair. Ansible Vault uniquement.
```
ansible-vault encrypt_string 'pass' --name 'admin_password'
```
`.vault_pass` est gitignoré, partagé via Bitwarden/KeePass.

## Réunions
Modèle dans [docs/meetings/template.md](docs/meetings/template.md).

# kiwinet-infra-ansible

Provisioning Ansible de l'infrastructure Kiwinet — deux environnements cibles.

Ce repo contient les playbooks et les rôles permettant de reproduire l'état cible
des machines depuis zéro : packages système, hardening SSH, firewall UFW,
installation Docker CE, montages réseau NAS, structure des répertoires et
démarrage des stacks.

## Contexte

Composant IaC du projet [Kiwinet](https://kiwinet.me) — infrastructure DevOps auto-hébergée.  
Voir [kiwinet-docs](https://github.com/Rookain-Kiwi/kiwinet-docs) pour l'ADR-001 et la stack technique.

## Cibles

| Playbook | Cible | Architecture | Rôles |
|---|---|---|---|
| `playbook-freebox.yml` | VM Freebox Delta | ARM64 | base, ssh, ufw, docker, storage, kiwinet |
| `playbook-cloud.yml` | VPS Scaleway | x86_64 | base, user, ssh, ufw, docker, kiwinet-web |

## Structure

```
kiwinet-infra-ansible/
├── ansible.cfg                  # Configuration Ansible locale (défaut : Freebox)
├── inventory.ini                # Groupes [freebox] et [cloud]
├── playbook-freebox.yml         # Provisioning VM Freebox Delta (ARM64)
├── playbook-cloud.yml           # Provisioning VPS Scaleway (x86_64)
├── roles/
│   ├── base/                    # Packages système, locale, timezone
│   ├── user/                    # Création utilisateur rookain (cloud uniquement)
│   ├── ssh/                     # Hardening sshd_config, clés autorisées, port SSH
│   ├── ufw/                     # Firewall — règles paramétrables par playbook
│   ├── docker/                  # Docker CE (ARM64 ou AMD64) + plugin Compose v2
│   ├── storage/                 # Montages CIFS NAS Freebox (freebox uniquement)
│   ├── kiwinet/                 # Répertoires /opt, clone repos, démarrage stacks
│   └── kiwinet-web/             # Déploiement container kiwinet-web (cloud uniquement)
└── docs/
    └── usage.md                 # Prérequis et commandes
```

## Lancement rapide

```bash
# Prérequis
pip install ansible
ansible-galaxy collection install community.general community.docker ansible.posix

# Freebox
ansible-playbook playbook-freebox.yml

# VPS Scaleway (après terraform apply dans kiwinet-infra-cloud)
ansible-playbook playbook-cloud.yml
```

Voir [docs/usage.md](docs/usage.md) pour le détail des opérations et la gestion des secrets.

## Ce repo ne gère pas

- Les fichiers de secrets (`.env`, `acme.json`) — déployés manuellement
- La configuration fine des services (Traefik, Grafana, HA) — dans `kiwinet-services` et `kiwinet-observability`
- L'infrastructure cloud (provisioning VM) — dans `kiwinet-infra-cloud` (Terraform + Scaleway)
# kiwinet-infra-vm

Provisioning Ansible de la VM Kiwinet — Debian ARM64 (Freebox Delta).

Ce repo contient le playbook et les rôles permettant de reproduire l'état cible de la VM depuis zéro : packages système, hardening SSH, firewall UFW, installation Docker CE, structure des répertoires et démarrage des stacks.

## Contexte

Composant IaC du projet [Kiwinet](https://kiwinet.me) — infrastructure DevOps auto-hébergée.  
Voir [kiwinet-docs](https://github.com/Rookain-Kiwi/kiwinet-docs) pour l'ADR-001 et la stack technique.

## Structure

```
kiwinet-infra-vm/
├── ansible.cfg          # Configuration Ansible locale
├── inventory.ini        # Cible : VM Kiwinet (IP à renseigner localement)
├── playbook.yml         # Playbook principal
├── roles/
│   ├── base/            # Packages système, locale, timezone
│   ├── ssh/             # Hardening sshd_config, clés autorisées
│   ├── ufw/             # Firewall (22, 80, 443, 25565, WireGuard UDP)
│   ├── docker/          # Docker CE ARM64 + plugin Compose v2
│   └── kiwinet/         # Répertoires /opt, clone repos, docker compose up
└── docs/
    └── usage.md         # Prérequis et commandes
```

## Lancement rapide

```bash
# Prérequis
pip install ansible
ansible-galaxy collection install community.general community.docker

# Exécution
ansible-playbook playbook.yml
```

Voir [docs/usage.md](docs/usage.md) pour le détail des options et la gestion des secrets.

## Ce repo ne gère pas

- Les fichiers de secrets (`.env`, `acme.json`) — déployés manuellement
- La configuration fine des services (Traefik, Grafana, HA) — dans `kiwinet-services` et `kiwinet-observability`
- L'infrastructure cloud — dans `kiwinet-infra-cloud` (Terraform + Scaleway)

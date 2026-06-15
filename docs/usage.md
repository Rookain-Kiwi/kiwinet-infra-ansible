# Guide opérationnel — kiwinet-infra-ansible

Ce document décrit les étapes complètes pour provisionner les machines Kiwinet
depuis zéro, ainsi que les opérations de maintenance courantes.

---

## Environnement Freebox Delta (ARM64)

### Création de la VM (Freebox OS)

La Freebox Delta injecte la clé SSH via cloud-init au premier démarrage — aucune
manipulation manuelle sur la VM n'est nécessaire avant de lancer Ansible.

**Étapes dans Freebox OS :**

1. Machines virtuelles → Créer une VM
2. Système pré-installé : `Debian 12 (Bookworm)`
3. Utilisateur par défaut : `rookain`
4. Clé SSH : coller le contenu de `~/.ssh/kiwinet.pub`
   ```bash
   cat ~/.ssh/kiwinet.pub
   ```
5. Mot de passe : laisser vide (authentification par clé uniquement)
6. **Accès aux disques Freebox : laisser décoché** — les montages CIFS sont gérés
   par le rôle `storage`, pas par cloud-init
7. Suivant → choisir la taille du disque → Terminer
8. Allumer la VM, attendre qu'une IP apparaisse dans l'interface

**Optionnel mais recommandé :** fixer une IP statique dans Freebox OS
(Paramètres DHCP → Baux statiques) pour éviter un changement d'IP après redémarrage.

### Prérequis manuels avant le playbook

Certains fichiers de secrets ne sont pas versionnés et doivent être déposés
manuellement sur la VM avant de lancer le playbook, ou immédiatement après
selon le service.

**Rôle `db` (PostgreSQL + PgAdmin) :**

```bash
# Créer les fichiers .env à partir des exemples versionnés
cp /opt/kiwinet-services/postgres/.env.example /opt/kiwinet-services/postgres/.env
cp /opt/kiwinet-services/pgadmin/.env.example  /opt/kiwinet-services/pgadmin/.env

# Éditer avec les valeurs réelles
nano /opt/kiwinet-services/postgres/.env   # POSTGRES_USER, POSTGRES_PASSWORD
nano /opt/kiwinet-services/pgadmin/.env    # PGADMIN_EMAIL, PGADMIN_PASSWORD
```

**Rôle `kiwinet` (autres services) :**

```bash
# Traefik
nano /opt/kiwinet-services/traefik/.env

# Komf (credentials Komga + clé ComicVine — non versionné)
# Template disponible dans komf/config/application.yml.template
nano /opt/kiwinet-services/komf/config/application.yml

# acme.json — doit être un fichier (pas un répertoire)
touch /opt/kiwinet-services/traefik/acme.json
chmod 600 /opt/kiwinet-services/traefik/acme.json
```

### Première installation

```bash
# Installer Ansible
pip install ansible
ansible-galaxy collection install community.general community.docker ansible.posix

# Cloner le repo
git clone git@github.com:Rookain-Kiwi/kiwinet-infra-ansible.git
cd kiwinet-infra-ansible

# Copier la clé CI/CD (répertoire gitignored)
cp ~/.ssh/kiwinet_deploy.pub roles/ssh/files/kiwinet_deploy.pub
```

> `kiwinet.pub` n'a pas besoin d'être dans `roles/ssh/files/` — elle est déjà
> présente sur la VM depuis la création. Seule `kiwinet_deploy.pub` est ajoutée par Ansible.

### Vérifier la connectivité

```bash
ansible freebox -m ping
# Attendu : kiwinet-vm | SUCCESS => { "ping": "pong" }
```

### Lancer le playbook

```bash
ansible-playbook playbook-freebox.yml
```

Ordre d'exécution : `base → ssh → ufw → docker → storage → db → kiwinet`.

### Après le déploiement — créer la base Synapse

Une fois PostgreSQL démarré, créer la base dédiée à Synapse :

```bash
docker exec -it postgres psql -U <POSTGRES_USER>
```

```sql
CREATE DATABASE synapse;
CREATE USER synapse WITH PASSWORD 'motdepasse';
GRANT ALL PRIVILEGES ON DATABASE synapse TO synapse;
\q
```

---

## Environnement VPS Scaleway (x86_64)

### Prérequis

Le VPS doit être provisionné via Terraform avant de lancer Ansible :

```bash
cd ~/GIT/kiwinet-infra-cloud
terraform apply
# Noter l'IP flexible en sortie (instance_public_ip)
```

Mettre à jour `inventory.ini` avec l'IP retournée :

```ini
[cloud]
kiwinet-cloud ansible_host=<instance_public_ip> ansible_port=22
```

### Vérifier la connectivité

```bash
ansible cloud -m ping
# Attendu : kiwinet-cloud | SUCCESS => { "ping": "pong" }
```

### Lancer le playbook

```bash
ansible-playbook playbook-cloud.yml
```

Ordre d'exécution : `base → user → ssh → ufw → docker → kiwinet-web`.

### Après le playbook — bascule port SSH

Le rôle `ssh` configure le port 2222. Mettre à jour `inventory.ini` :

```ini
[cloud]
kiwinet-cloud ansible_host=<instance_public_ip> ansible_port=2222
```

### Après le playbook — mise à jour DNS et CI/CD

1. **Bluehost** — mettre à jour l'enregistrement A de `kiwinet.me` avec l'IP flexible
2. **GitHub Secrets** sur `kiwinet-web` — mettre à jour `DEPLOY_HOST` avec l'IP flexible
3. **`firewall.tf`** dans `kiwinet-infra-cloud` — supprimer la règle port 22 temporaire

---

## Opérations de maintenance

### Freebox uniquement

```bash
# Mettre à jour les règles firewall
ansible-playbook playbook-freebox.yml --tags ufw

# Re-appliquer le hardening SSH
ansible-playbook playbook-freebox.yml --tags ssh

# Mettre à jour les packages système
ansible-playbook playbook-freebox.yml --tags base

# Re-appliquer les montages CIFS
ansible-playbook playbook-freebox.yml --tags storage

# Re-appliquer uniquement la couche db (réseau, volumes, stacks)
ansible-playbook playbook-freebox.yml --tags db

# Dry-run
ansible-playbook playbook-freebox.yml --check --diff
```

### Cloud uniquement

```bash
# Mettre à jour les règles firewall
ansible-playbook playbook-cloud.yml --tags ufw

# Re-appliquer le hardening SSH
ansible-playbook playbook-cloud.yml --tags ssh

# Dry-run
ansible-playbook playbook-cloud.yml --check --diff
```

---

## Dépannage

### Erreur "unreachable" — Freebox

```bash
ssh -i ~/.ssh/kiwinet rookain@kiwinet.me
```

### Erreur "unreachable" — Cloud

```bash
ssh -i ~/.ssh/kiwinet_deploy root@<instance_public_ip>
```

### Erreur sur le rôle docker (clé GPG)

```bash
# Freebox
ssh rookain@kiwinet.me "sudo rm /etc/apt/keyrings/docker.gpg"
ansible-playbook playbook-freebox.yml --tags docker

# Cloud
ssh -i ~/.ssh/kiwinet_deploy root@<instance_public_ip> "rm /etc/apt/keyrings/docker.gpg"
ansible-playbook playbook-cloud.yml --tags docker
```

### PostgreSQL refuse de démarrer

Cause la plus fréquente : permissions incorrectes sur `/var/lib/postgresql/data`.

```bash
# Vérifier le propriétaire (attendu : 999)
ls -la /var/lib/postgresql/

# Corriger si nécessaire
sudo chown -R 999:999 /var/lib/postgresql/data

# Relancer
cd /opt/kiwinet-services/postgres && docker compose up -d --force-recreate
```

### PgAdmin inaccessible après démarrage

Vérifier que PgAdmin est sur les deux réseaux `proxy` et `db` :

```bash
docker inspect pgadmin | grep -A 20 '"Networks"'
```

Les deux réseaux doivent apparaître. Si `db` est absent, vérifier que le réseau
existe et relancer la stack :

```bash
docker network ls | grep db
cd /opt/kiwinet-services/pgadmin && docker compose up -d --force-recreate
```

### Erreur architecture

```bash
# Freebox — attendu : aarch64
uname -m

# Cloud — attendu : x86_64
uname -m
```

### VM test vs VM prod (Freebox)

1. Éteindre la VM prod
2. Créer et provisionner la VM test (2 CPU, RAM disponible)
3. Valider le playbook sur la VM test
4. Éteindre la VM test, rallumer la VM prod

### Faux positifs DNS dans Uptime Kuma

**Symptôme :** Uptime Kuma déclare des services down (erreur `ENOTFOUND`) alors qu'ils
sont accessibles par IP.

**Cause :** Le resolver DNS de la Freebox (`192.168.1.254`) est instable par intermittence.
Les containers Docker héritent de ce resolver seul au démarrage et n'ont aucun fallback.

**Correction Ansible (déjà appliquée) :**

- Rôle `base` : déploie `/etc/systemd/resolved.conf.d/dns-upstream.conf`
  — fallback `1.1.1.1` / `9.9.9.9` au niveau système (VM)
- Rôle `docker` : déploie `/etc/docker/daemon.json`
  — fallback `1.1.1.1` / `9.9.9.9` pour tous les containers Docker

**Correction manuelle Uptime Kuma (hors Ansible) :**

- Directive `dns:` ajoutée dans `kiwinet-observability/uptime/docker-compose.yml`
- Monitors : Essais = 2, Timeout = 10s, méthode GET (Home Assistant rejette HEAD avec 405)

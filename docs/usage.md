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

Ordre d'exécution : `base → ssh → ufw → docker → storage → kiwinet`.

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
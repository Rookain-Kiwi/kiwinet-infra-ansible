# Guide opérationnel — kiwinet-infra-vm

Ce document décrit les étapes complètes pour provisionner la VM Kiwinet depuis un Debian minimal, ainsi que les opérations de maintenance courantes.

---

## Création de la VM (Freebox OS)

La Freebox Delta injecte la clé SSH via cloud-init au premier démarrage — aucune manipulation manuelle sur la VM n'est nécessaire avant de lancer Ansible.

**Étapes dans Freebox OS :**

1. Machines virtuelles → Créer une VM
2. Système pré-installé : `Debian 12 (Bookworm)`
3. Utilisateur par défaut : `rookain`
4. Clé SSH : coller le contenu de `~/.ssh/kiwinet.pub`
   ```bash
   cat ~/.ssh/kiwinet.pub
   ```
5. Mot de passe : laisser vide (authentification par clé uniquement)
6. **Accès aux disques Freebox : laisser décoché** — les montages CIFS sont gérés par le rôle `storage`, pas par cloud-init
7. Suivant → choisir la taille du disque → Terminer
8. Allumer la VM, attendre qu'une IP apparaisse dans l'interface

**Optionnel mais recommandé :** fixer une IP statique dans Freebox OS (Paramètres DHCP → Baux statiques) pour éviter un changement d'IP après redémarrage.

---

## Première installation (from scratch)

### Étape 1 : Préparer la machine locale

```bash
# Installer Ansible
pip install ansible

# Installer les collections requises
ansible-galaxy collection install community.general community.docker ansible.posix
```

### Étape 2 : Configurer le repo

```bash
git clone git@github.com:Rookain-Kiwi/kiwinet-infra-vm.git
cd kiwinet-infra-vm

# Copier la clé CI/CD dans roles/ssh/files/ (répertoire gitignored)
# La clé kiwinet.pub est déjà injectée par Freebox OS à la création de la VM
cp ~/.ssh/kiwinet_deploy.pub roles/ssh/files/kiwinet_deploy.pub
```

> `kiwinet.pub` n'a pas besoin d'être dans `roles/ssh/files/` — elle est déjà présente
> sur la VM depuis la création. Seule `kiwinet_deploy.pub` est ajoutée par Ansible.

### Étape 3 : Vérifier la connectivité

```bash
ansible kiwinet -m ping
# Attendu : kiwinet-vm | SUCCESS => { "ping": "pong" }
```

### Étape 4 : Lancer le playbook

```bash
ansible-playbook playbook.yml
```

Le playbook s'exécute dans l'ordre : `base → ssh → ufw → docker → storage → kiwinet`.

---

## Étapes manuelles post-playbook

Ces étapes ne peuvent pas être automatisées (secrets, fichiers sensibles) :

### 1. Déposer les fichiers .env

Chaque stack a besoin de son fichier `.env` :

```bash
# Exemple pour Traefik
scp .env.traefik rookain@kiwinet.me:/opt/kiwinet-services/traefik/.env
```

### 2. Vérifier acme.json

```bash
ssh rookain@kiwinet.me
ls -la /opt/kiwinet-services/traefik/acme.json
# Attendu : -rw------- 1 rookain rookain
```

### 3. Démarrer les stacks manuellement

Après dépôt des `.env` :

```bash
ssh rookain@kiwinet.me

# Traefik en premier (crée le réseau proxy)
cd /opt/kiwinet-services/traefik && docker compose up -d

# Puis les autres stacks
cd /opt/kiwinet-services && docker compose up -d
cd /opt/kiwinet-observability && docker compose up -d
```

### 4. Configurer Plex (connexion locale)

Dans Plex Web → **Paramètres → Réseau → "URL personnalisées pour accéder au serveur"** :

```
https://plex.kiwinet.me,http://192.168.1.33:32400
```

Sans cette configuration, les clients Android TV transitent par Traefik et subissent des coupures audio.

---

## Opérations de maintenance

### Mettre à jour les règles firewall uniquement

```bash
ansible-playbook playbook.yml --tags ufw
```

### Re-appliquer le hardening SSH

```bash
ansible-playbook playbook.yml --tags ssh
```

### Mettre à jour les packages système

```bash
ansible-playbook playbook.yml --tags base
```

### Re-appliquer les montages CIFS uniquement

```bash
ansible-playbook playbook.yml --tags storage
```

### Dry-run (simulation)

```bash
ansible-playbook playbook.yml --check --diff
```

---

## Dépannage

### Erreur "unreachable"

Vérifier que la VM est démarrée et que la clé SSH est bien celle utilisée dans `ansible.cfg` :

```bash
ssh -i ~/.ssh/kiwinet rookain@kiwinet.me
```

### Erreur sur le rôle docker (clé GPG)

Si la clé GPG Docker échoue, supprimer le fichier et relancer :

```bash
ssh rookain@kiwinet.me "sudo rm /etc/apt/keyrings/docker.gpg"
ansible-playbook playbook.yml --tags docker
```

### Erreur architecture

Le playbook refuse de s'exécuter si l'architecture n'est pas AArch64. Vérifier sur la VM :

```bash
uname -m
# Attendu : aarch64
```

### VM test vs VM prod

Ce playbook est conçu pour être exécuté sur une VM à la fois.
La stratégie de test recommandée est de séquencer, pas de paralléliser :

1. Éteindre la VM prod
2. Créer et provisionner la VM test (2 CPU, RAM disponible)
3. Valider le playbook sur la VM test
4. Éteindre la VM test, rallumer la VM prod

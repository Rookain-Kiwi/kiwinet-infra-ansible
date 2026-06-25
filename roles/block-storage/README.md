# Rôle : `block-storage`

Formatage et montage du block storage Scaleway pour PostgreSQL.

## Responsabilités

1. **Vérification** : confirme que le device existe (`/dev/sdb`)
2. **Formatage** : formate en ext4 si vierge (destructif ⚠️)
3. **Point de montage** : crée `/var/lib/postgresql` avec les bonnes permissions
4. **Montage persistant** : ajoute une entrée dans `/etc/fstab`
5. **Vérification** : confirme que le montage fonctionne

## Prérequis

- Device attaché physiquement (Terraform + Scaleway)
- Pas de données critiques dessus (DANGER: formatage destructif)
- Ansible `community.general` (pour `filesystem` module)

## Variables

```yaml
block_storage_device: /dev/sdb                    # Device à monter
block_storage_mount_path: /var/lib/postgresql     # Point de montage
block_storage_fstype: ext4                        # Type filesystem
block_storage_mount_opts: defaults,noatime,...    # Options fstab
block_storage_owner: "999"                        # UID postgres (Alpine)
block_storage_group: "999"                        # GID postgres
block_storage_mode: "0700"                        # Permissions
```

## Utilisation dans playbook

```yaml
roles:
  - block-storage  # Avant le rôle `db`
  - db
```

## Flux d'exécution

1. ✅ Vérifie `/dev/sdb` existe
2. ✅ Vérifie s'il est déjà formaté (`blkid`)
3. ✅ Formate en ext4 (si vierge)
4. ✅ Crée `/var/lib/postgresql` (owner: 999:999, mode: 0700)
5. ✅ Monte dans `/etc/fstab` (options systemd automount)
6. ✅ Vérifie que le montage fonctionne

## Idempotence

✅ **Sûr à relancer** :
- Si déjà formaté → passe le formatage
- Si déjà monté → confirme le montage dans fstab
- Si permissions OK → ne change rien

## Risques

⚠️ **DANGER** : Le rôle formate le device sans demander confirmation.
- Vérifier le device correct avant de lancer (`block_storage_device`)
- Pas de données critiques sur `/dev/sdb`

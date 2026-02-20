# ansible-playbook-wordpress-ops

> Repo playbook – Routines opérationnelles d'un WordPress dockerisé.
> Ce repo est **indépendant du rôle**. Il l'installe via `requirements.yml`.

---

## Architecture – 2 repos séparés

```
┌─────────────────────────────────────────────────────────────────┐
│  ansible-role-wordpress-ops   (repo rôle – réutilisable)        │
│  ── defaults/  handlers/  tasks/  templates/  vars/  meta/      │
└─────────────────────────────┬───────────────────────────────────┘
                              │  référencé dans requirements.yml
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  ansible-playbook-wordpress-ops   (ce repo – spécifique env)    │
│  ── site.yml                                                     │
│  ── requirements.yml                                            │
│  ── ansible.cfg                                                  │
│  ── inventory/                                                   │
│     ├── hosts.yml              ← hosts et groupes               │
│     └── group_vars/                                             │
│         ├── all.yml            ← vars communes                  │
│         ├── wordpress_servers.yml  ← vars du groupe             │
│         └── vault.yml          ← secrets chiffrés (ansible-vault)│
└─────────────────────────────────────────────────────────────────┘
```

---

## Groupes et hosts (inventory/hosts.yml)

```
all
└── wordpress_servers
    ├── prod
    │   └── wp-prod-01      (1.2.3.4)   s3_prefix: wordpress/prod
    └── staging
        └── wp-staging-01   (5.6.7.8)   s3_prefix: wordpress/staging
```

---

## Installation & premier lancement

### 1. Cloner ce repo
```bash
git clone https://github.com/votre-org/ansible-playbook-wordpress-ops.git
cd ansible-playbook-wordpress-ops
```

### 2. Installer le rôle et les collections
```bash
ansible-galaxy role install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

### 3. Configurer le vault (secrets)
```bash
# Créer le fichier vault
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml

# Éditer les secrets
vi inventory/group_vars/vault.yml

# Chiffrer le fichier
ansible-vault encrypt inventory/group_vars/vault.yml

# Créer le fichier de mot de passe vault (ne pas commiter)
echo "votre_mot_de_passe_vault" > .vault_pass
chmod 600 .vault_pass
```

### 4. Adapter l'inventaire
```bash
vi inventory/hosts.yml                       # Renseigner les IPs
vi inventory/group_vars/wordpress_servers.yml  # Adapter s3_bucket, région, etc.
```

---

## Flux S3 – Comment ça marche

```
backup_db   ──►  dump MySQL local  ──►  s3://bucket/wordpress/database/db_YYYY-MM-DD_HHMMSS.sql
backup_files ──► archive tar.gz local ──► s3://bucket/wordpress/files/files_YYYY-MM-DD_HHMMSS.tar.gz
                 (+ dump DB via handler, uploadé aussi)
```

Pour activer l'upload S3 :
```yaml
# inventory/group_vars/wordpress_servers.yml
s3_enabled: true
s3_bucket: "mon-bucket-wordpress-backup"
s3_region: "eu-west-1"
```

---

## Commandes par routine

> **Légende**
> - `--vault-password-file .vault_pass` : déchiffre les secrets automatiquement
> - `-l prod` : limite l'exécution au groupe `prod` (ou `staging`, ou un host précis)
> - `-e "var=val"` : surcharger une variable au runtime

---

### 🗄️ Routine 1 – Backup de la base de données

```bash
# Backup DB sur TOUS les serveurs WordPress
ansible-playbook site.yml \
  --tags backup_db \
  --vault-password-file .vault_pass

# Backup DB uniquement sur prod
ansible-playbook site.yml \
  --tags backup_db \
  --limit prod \
  --vault-password-file .vault_pass

# Backup DB sur staging
ansible-playbook site.yml \
  --tags backup_db \
  --limit staging \
  --vault-password-file .vault_pass

# Backup DB sur un seul host
ansible-playbook site.yml \
  --tags backup_db \
  --limit wp-prod-01 \
  --vault-password-file .vault_pass

# Backup DB SANS upload S3 (même si s3_enabled: true dans group_vars)
ansible-playbook site.yml \
  --tags backup_db \
  --limit prod \
  -e "s3_enabled=false" \
  --vault-password-file .vault_pass

# Backup DB avec rétention personnalisée
ansible-playbook site.yml \
  --tags backup_db \
  --limit prod \
  -e "backup_retention_days=30" \
  --vault-password-file .vault_pass
```

---

### 📁 Routine 2 – Backup des fichiers du site

> Le backup DB est déclenché **automatiquement en amont** via le handler.
> Les deux fichiers (DB + fichiers) sont uploadés sur S3.

```bash
# Backup fichiers (+ DB via handler) sur prod
ansible-playbook site.yml \
  --tags backup_files \
  --limit prod \
  --vault-password-file .vault_pass

# Backup fichiers sur staging
ansible-playbook site.yml \
  --tags backup_files \
  --limit staging \
  --vault-password-file .vault_pass

# Backup fichiers avec répertoire de backup personnalisé
ansible-playbook site.yml \
  --tags backup_files \
  --limit prod \
  -e "backup_base_dir=/mnt/data/backups" \
  --vault-password-file .vault_pass
```

---

### 🔄 Routine 3 – Restauration de la base de données

> Requiert : `restore_db_file` (chemin absolu vers le .sql sur le host cible)

```bash
# Restaurer la DB depuis un fichier local sur le serveur
ansible-playbook site.yml \
  --tags restore_db \
  --limit prod \
  -e "restore_db_file=/opt/backups/wordpress/database/db_2024-01-15_120000.sql" \
  --vault-password-file .vault_pass

# Restaurer la DB depuis un backup S3 (télécharger d'abord manuellement)
# Étape 1 : télécharger depuis S3 sur le serveur cible
ssh ubuntu@1.2.3.4 "aws s3 cp s3://mon-bucket/wordpress/database/db_2024-01-15_120000.sql /tmp/"

# Étape 2 : lancer la restauration
ansible-playbook site.yml \
  --tags restore_db \
  --limit prod \
  -e "restore_db_file=/tmp/db_2024-01-15_120000.sql" \
  --vault-password-file .vault_pass
```

---

### 🔄 Routine 4 – Restauration des fichiers du site

> Requiert les deux : `restore_files_archive` ET `restore_db_file`
> La DB est restaurée **automatiquement en amont** via le handler.

```bash
# Restauration complète (fichiers + DB via handler)
ansible-playbook site.yml \
  --tags restore_files \
  --limit prod \
  -e "restore_files_archive=/opt/backups/wordpress/files/files_2024-01-15_120000.tar.gz" \
  -e "restore_db_file=/opt/backups/wordpress/database/db_2024-01-15_120000.sql" \
  --vault-password-file .vault_pass

# Restauration sur staging depuis des fichiers S3 (après téléchargement)
ansible-playbook site.yml \
  --tags restore_files \
  --limit staging \
  -e "restore_files_archive=/tmp/files_2024-01-15_120000.tar.gz" \
  -e "restore_db_file=/tmp/db_2024-01-15_120000.sql" \
  --vault-password-file .vault_pass
```

---

### 🔌 Routine 5 – Mise à jour de tous les plugins

```bash
# Mettre à jour tous les plugins sur prod
ansible-playbook site.yml \
  --tags update_plugins \
  --limit prod \
  --vault-password-file .vault_pass

# Mettre à jour tous les plugins sur staging
ansible-playbook site.yml \
  --tags update_plugins \
  --limit staging \
  --vault-password-file .vault_pass

# Dry-run : voir sans appliquer (Ansible check mode)
ansible-playbook site.yml \
  --tags update_plugins \
  --limit prod \
  --check \
  --vault-password-file .vault_pass
```

---

### 🔧 Routine 6 – Activation / désactivation d'un plugin

```bash
# Activer un plugin sur prod
ansible-playbook site.yml \
  --tags manage_plugin \
  --limit prod \
  -e "plugin_name=woocommerce plugin_action=activate" \
  --vault-password-file .vault_pass

# Désactiver un plugin sur prod
ansible-playbook site.yml \
  --tags manage_plugin \
  --limit prod \
  -e "plugin_name=woocommerce plugin_action=deactivate" \
  --vault-password-file .vault_pass

# Activer un plugin sur tous les serveurs WordPress
ansible-playbook site.yml \
  --tags manage_plugin \
  -e "plugin_name=akismet plugin_action=activate" \
  --vault-password-file .vault_pass

# Activer sur staging uniquement
ansible-playbook site.yml \
  --tags manage_plugin \
  --limit staging \
  -e "plugin_name=debug-bar plugin_action=activate" \
  --vault-password-file .vault_pass
```

---

### 🧹 Routine 7 – Nettoyage des anciens backups

```bash
# Nettoyer les backups > 7 jours (valeur par défaut)
ansible-playbook site.yml \
  --tags cleanup \
  --limit prod \
  --vault-password-file .vault_pass

# Nettoyer avec rétention personnalisée (3 jours)
ansible-playbook site.yml \
  --tags cleanup \
  --limit prod \
  -e "backup_retention_days=3" \
  --vault-password-file .vault_pass

# Nettoyer sur tous les serveurs
ansible-playbook site.yml \
  --tags cleanup \
  --vault-password-file .vault_pass
```

---

## Commandes utiles pour déboguer

```bash
# Lister les tâches sans les exécuter
ansible-playbook site.yml --tags backup_db --list-tasks

# Lister les hosts ciblés sans exécuter
ansible-playbook site.yml --tags backup_db --limit prod --list-hosts

# Vérifier la syntaxe du playbook
ansible-playbook site.yml --syntax-check

# Exécution en mode verbeux (voir les détails)
ansible-playbook site.yml --tags backup_db --limit prod -vvv \
  --vault-password-file .vault_pass

# Tester la connectivité aux serveurs
ansible wordpress_servers -m ping --vault-password-file .vault_pass

# Voir les variables résolues pour un host
ansible wp-prod-01 -m debug -a "var=hostvars[inventory_hostname]" \
  --vault-password-file .vault_pass
```

---

## Mise à jour du rôle

```bash
# Mettre à jour vers la dernière version du rôle
ansible-galaxy role install -r requirements.yml --force

# Mettre à jour vers une version spécifique
# → Modifier requirements.yml : version: v1.2.0
# → Puis :
ansible-galaxy role install -r requirements.yml --force
```

---

## Structure du repo

```
ansible-playbook-wordpress-ops/
├── site.yml                              ← Playbook principal
├── requirements.yml                      ← Référence le rôle wordpress_ops
├── ansible.cfg                           ← Configuration Ansible
├── .gitignore
├── .vault_pass                           ← NE PAS COMMITER (dans .gitignore)
└── inventory/
    ├── hosts.yml                         ← Hosts et groupes (prod, staging)
    └── group_vars/
        ├── all.yml                       ← Variables communes
        ├── wordpress_servers.yml         ← Variables du groupe (S3, Docker…)
        └── vault.yml                     ← Secrets chiffrés (ansible-vault)
```

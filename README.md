# PowerDNS - Déploiement automatisé avec Ansible et Docker

Déploiement d'une infrastructure DNS complète (Master, Slave, Récurseur) avec PowerDNS, automatisé via Ansible et conteneurisé avec Docker.

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Serveur: 192.168.11.141                      │
│                    Réseau Docker: 172.5.0.0/16                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Master    │  │    Slave    │  │       Récurseur         │  │
│  │  (172.5.0.20)│  │ (172.5.0.21)│  │    (forward-zones)      │  │
│  │   Port 53   │◄─┤  Port 5454  │  │      Port 5354          │  │
│  └──────┬──────┘  └─────────────┘  └─────────────────────────┘  │
│         │                                                       │
│  ┌──────┴──────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  MariaDB    │  │ phpMyAdmin  │  │     PowerDNS-Admin      │  │
│  │             │  │  Port 8888  │  │       Port 8889         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 📋 Composants

| Service | Port | Description |
|---------|------|-------------|
| **PowerDNS Master** | 53 | Serveur DNS autoritaire principal |
| **PowerDNS Slave** | 5454 | Serveur DNS autoritaire secondaire (réplication) |
| **PowerDNS Récurseur** | 5354 | Résolveur DNS avec cache |
| **PowerDNS-Admin** | 8889 | Interface web d'administration |
| **phpMyAdmin** | 8888 | Gestion de la base de données |
| **MariaDB** | - | Base de données (interne) |

## 🚀 Déploiement

### Prérequis

- Ansible installé sur la machine de contrôle
- Docker installé sur le serveur cible
- Accès SSH au serveur cible

### Configuration

1. **Inventaire** - Modifier `hosts` :
```ini
[pdns_servers]
192.168.11.141 ansible_user=votre_user ansible_ssh_private_key_file=~/.ssh/id_rsa
```

2. **Variables** - Modifier `roles/powerdns/defaults/main.yml` :
```yaml
pdns_master_ip: 172.5.0.20
pdns_slave_ip: 172.5.0.21
dns_local_suffix: uauben    # Suffixe pour vos zones locales
node_user: votre_user
```

### Exécution

```bash
# Déploiement complet
ansible-playbook -i hosts ansible-playbook-mysql.yml

# Déploiement par composant
ansible-playbook -i hosts ansible-playbook-mysql.yml --tags pdns          # Master + Slave
ansible-playbook -i hosts ansible-playbook-mysql.yml --tags pdns-recursor # Récurseur
ansible-playbook -i hosts ansible-playbook-mysql.yml --tags pdns-admin    # Interface web
ansible-playbook -i hosts ansible-playbook-mysql.yml --tags db            # MariaDB + phpMyAdmin
```

### Suppression

```bash
ansible-playbook -i hosts ansible-playbook-mysql.yml -e "wipe=true"
```

## 🔧 Configuration client DNS

Pour utiliser votre serveur DNS sur un client Linux :

```bash
# Créer le fichier de configuration
sudo tee /etc/systemd/resolved.conf.d/powerdns.conf << 'EOF'
[Resolve]
DNS=192.168.11.141
FallbackDNS=8.8.8.8
Domains=~uauben
EOF

# Appliquer
sudo systemctl restart systemd-resolved
```

## 🧪 Tests

```bash
# Tester le Master
dig @192.168.11.141 -p 53 votre-zone.uauben A

# Tester le Slave
dig @192.168.11.141 -p 5454 votre-zone.uauben A

# Tester le Récurseur (zones locales)
dig @192.168.11.141 -p 5354 votre-zone.uauben A

# Tester le Récurseur (Internet)
dig @192.168.11.141 -p 5354 google.com A
```

## 🌐 Accès aux interfaces

| Interface | URL |
|-----------|-----|
| PowerDNS-Admin | http://192.168.11.141:8889 |
| phpMyAdmin | http://192.168.11.141:8888 |

**Identifiants phpMyAdmin :** `root` / `my-secret-pw`

**PowerDNS-Admin :** Créer un compte lors de la première connexion (le premier utilisateur devient admin).

## 📁 Structure du projet

```
.
├── ansible-playbook-mysql.yml    # Playbook principal
├── hosts                         # Inventaire Ansible
├── docker/
│   ├── pdns-mysql/              # Image PowerDNS (Master/Slave)
│   ├── pdns-recursor/           # Image Récurseur
│   └── pdns-admin/              # Image interface web
└── roles/
    └── powerdns/
        ├── defaults/main.yml    # Variables par défaut
        └── tasks/main.yml       # Tâches de déploiement
```

## 📝 Notes

- **Synchronisation Master/Slave** : Automatique via AXFR/NOTIFY
- **Récurseur** : Configuré avec `forward-zones` pour router les zones locales vers le Master
- **En production** : Déployer Master, Slave et Récurseur sur des serveurs séparés

## 📄 Licence

MIT

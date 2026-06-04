# 📡 Documentation Zabbix 7.4

> Supervision de parc avec Zabbix Server, Agent, et déploiement automatique via Puppet.

---

## Table des matières

- [Installation du serveur Zabbix](#installation-du-serveur-zabbix)
  - [Prérequis](#prérequis)
  - [Installation du dépôt Zabbix](#installation-du-dépôt-zabbix)
  - [Installation des composants](#installation-des-composants)
  - [Configuration de la base de données](#configuration-de-la-base-de-données)
  - [Configuration et démarrage des services](#configuration-et-démarrage-des-services)
  - [Installation Web](#installation-web)
- [Configuration des clients Zabbix](#configuration-des-clients-zabbix)
  - [Installation de l'agent côté client](#installation-de-lagent-côté-client)
  - [Configuration de l'agent côté client](#configuration-de-lagent-côté-client)
  - [Configuration de l'auto-enregistrement côté serveur](#configuration-de-lauto-enregistrement-côté-serveur)
- [Ajout d'Items personnalisés](#ajout-ditems-personnalisés)
- [Déploiement automatique via Puppet](#déploiement-automatique-via-puppet)

---

## Installation du serveur Zabbix

### Prérequis

| Composant | Valeur |
|-----------|--------|
| OS | Debian Bookworm (12) |
| Composants | Server, Frontend, Agent |
| Base de données | MySQL / MariaDB |
| Serveur web | Apache |
| Port serveur Zabbix | `10051` |
| Port agent Zabbix | `10050` |

> 🔗 **Liens utiles :**
> - [Page de téléchargement officielle](https://www.zabbix.com/download)
> - [Documentation officielle Zabbix 7.4](https://www.zabbix.com/documentation/7.4/en/manual)

---

### Installation du dépôt Zabbix

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian12_all.deb
dpkg -i zabbix-release_latest_7.4+debian12_all.deb
apt update
```

---

### Installation des composants

```bash
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

---

### Configuration de la base de données

#### 1. Installer MariaDB

```bash
apt install mariadb-server
```

#### 2. Créer la base et l'utilisateur

```bash
mysql -u root -p
# ou : sudo mysql
```

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
SET GLOBAL log_bin_trust_function_creators = 1;
QUIT;
```

#### 3. Importer le schéma SQL

```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

#### 4. Désactiver l'option temporaire

```bash
mysql -u root -p
# ou : sudo mysql
```

```sql
SET GLOBAL log_bin_trust_function_creators = 0;
QUIT;
```

#### 5. Configurer le mot de passe dans Zabbix

Modifier le fichier `/etc/zabbix/zabbix_server.conf` :

```ini
DBPassword=password
```

---

### Configuration et démarrage des services

```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```

---

### Installation Web

1. Ouvrir un navigateur et aller sur : `http://IP_SERVEUR/zabbix`
2. Suivre l'assistant d'installation web
3. Se connecter avec les identifiants par défaut :

| Champ | Valeur |
|-------|--------|
| Login | `Admin` |
| Mot de passe | `zabbix` |

> ⚠️ **Penser à changer le mot de passe par défaut après la première connexion.**

> 🔗 [Créer un utilisateur normal dans Zabbix](https://www.zabbix.com/documentation/7.4/en/manual/quickstart/login)

---

## Configuration des clients Zabbix

L'objectif est que chaque client s'enregistre **automatiquement** auprès du serveur Zabbix.

> 🔗 [Documentation auto-enregistrement Zabbix](https://www.zabbix.com/documentation/7.4/en/manual/discovery/auto_registration?hl=agent%2Cautoregistration)

---

### Installation de l'agent côté client

> 🔗 [Télécharger Zabbix Agent pour Debian 13](https://www.zabbix.com/download?zabbix=7.4&os_distribution=debian&os_version=13&components=agent&db=&ws=)

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian13_all.deb
dpkg -i zabbix-release_latest_7.4+debian13_all.deb
apt update
apt install zabbix-agent
```

---

### Configuration de l'agent côté client

Modifier le fichier `/etc/zabbix/zabbix_agentd.conf` :

```ini
Server=IP_SERVEUR
ServerActive=IP_SERVEUR
# Supprimer ou commenter la ligne Hostname=
```

Puis redémarrer l'agent :

```bash
systemctl restart zabbix-agent
```

---

### Configuration de l'auto-enregistrement côté serveur

Dans l'interface web du serveur, suivre ce chemin :

```
Alertes → Actions → Actions d'enregistrement automatique → Créer une action
```

Paramètres recommandés :

| Étape | Action |
|-------|--------|
| 1 | Attribuer un nom à l'action |
| 2 | Cocher **Activé** |
| 3 | Onglet **Opérations** → Ajouter les opérations |

Opérations suggérées :

- ✅ Ajouter hôte
- ✅ Ajouter au groupe d'hôtes : `Discovery hosts`
- ✅ Lier le modèle : `Linux by Zabbix agent active`

---

## Ajout d'Items personnalisés

Les **Items** sont des métriques collectées sur les postes supervisés.

> 🔗 [Liste complète des items disponibles](https://www.zabbix.com/documentation/7.2/en/manual/config/items/itemtypes/zabbix_agent)

### Ajouter un item dans un modèle existant

```
Collecte de données → Modèles → Sélectionner le modèle → Créer un élément
```

### Exemple : lister les paquets installés

Item ajouté dans le modèle `Linux by Zabbix agent active` pour récupérer la liste des paquets installés sur chaque poste :

> 🔗 [Documentation system.sw.packages](https://www.zabbix.com/documentation/7.2/en/manual/config/items/itemtypes/zabbix_agent#system.sw.packages)

> 💡 Il est également possible de **créer un modèle personnalisé** pour choisir précisément les informations à superviser.

---

## Déploiement automatique via Puppet

Pour déployer l'agent Zabbix automatiquement sur tous les clients, ajouter le bloc suivant dans le fichier `site.pp` de Puppet :

```puppet
# Zabbix agent — Managed by Puppet

exec { 'download-zabbix-repo':
  command => '/usr/bin/wget -q https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian13_all.deb -O /tmp/zabbix-release.deb',
  creates => '/tmp/zabbix-release.deb',
}

exec { 'install-zabbix-repo':
  command => '/usr/bin/dpkg -i /tmp/zabbix-release.deb',
  creates => '/etc/apt/sources.list.d/zabbix.list',
  require => Exec['download-zabbix-repo'],
}

exec { 'apt-update-zabbix':
  command     => '/usr/bin/apt-get update',
  refreshonly => true,
  subscribe   => Exec['install-zabbix-repo'],
}

package { 'zabbix-agent':
  ensure  => present,
  require => Exec['apt-update-zabbix'],
}

file { '/etc/zabbix/zabbix_agentd.conf':
  ensure  => present,
  owner   => 'root',
  group   => 'zabbix',
  mode    => '0640',
  content => "# Managed by Puppet - do not edit manually

PidFile=/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0

Server=IP_SERVEUR_ZABBIX
ServerActive=IP_SERVEUR_ZABBIX

Include=/etc/zabbix/zabbix_agentd.d/*.conf
",
  require => Package['zabbix-agent'],
  notify  => Service['zabbix-agent'],
}

service { 'zabbix-agent':
  ensure  => running,
  enable  => true,
  require => Package['zabbix-agent'],
}
```

> ⚠️ Remplacer `IP_SERVEUR_ZABBIX` par l'adresse IP réelle de votre serveur Zabbix.

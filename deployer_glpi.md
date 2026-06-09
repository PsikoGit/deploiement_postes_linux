# 📦 Documentation GLPI

> Guide d'installation et de configuration de GLPI avec automatisation via Puppet.

---

## 📋 Sommaire

1. [Installation de GLPI](#1-installation-de-glpi)
   - [Préparer le répertoire](#11-préparer-le-répertoire)
   - [Télécharger l'archive](#12-télécharger-larchive)
   - [Configurer le VirtualHost Apache](#13-configurer-le-virtualhost-apache)
   - [Activer le site et redémarrer Apache](#14-activer-le-site-et-redémarrer-apache)
   - [Configurer la résolution DNS locale (client)](#15-configurer-la-résolution-dns-locale-client)
2. [Installation via l'interface web](#2-installation-via-linterface-web)
   - [Accéder à l'installateur](#21-accéder-à-linstallateur)
   - [Configurer la base de données](#22-configurer-la-base-de-données)
3. [Activer l'inventaire GLPI](#3-activer-linventaire-glpi)
4. [Installation de l'agent GLPI](#4-installation-de-lagent-glpi)
   - [Installation manuelle](#41-installation-manuelle)
   - [Vérification de l'agent](#42-vérification-de-lagent)
   - [Enregistrement dans l'inventaire](#43-enregistrement-dans-linventaire)
5. [Automatisation via Puppet](#5-automatisation-via-puppet)

---

## 1. Installation de GLPI

> 📖 [Documentation officielle GLPI](https://glpi-install.readthedocs.io/en/latest/)

### 1.1 Préparer le répertoire

```bash
mkdir /var/www/html/glpi
```

### 1.2 Télécharger l'archive

```bash
cd /var/www/html/glpi
wget https://github.com/glpi-project/glpi/releases/download/11.0.7/glpi-11.0.7.tgz
tar -xvzf glpi-11.0.7.tgz
```

### 1.3 Configurer le VirtualHost Apache

Créer le fichier `/etc/apache2/sites-available/glpi.conf` :

```apache
<VirtualHost *:80>
    ServerName rechidov.glpi

    DocumentRoot /var/www/html/glpi/glpi/public

    <Directory /var/www/html/glpi/glpi/public>
        Require all granted

        RewriteEngine On

        # Ensure authorization headers are passed to PHP.
        # Some Apache configurations may filter them and break usage of API, CalDAV, ...
        RewriteCond %{HTTP:Authorization} ^(.+)$
        RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

        # Redirect all requests to GLPI router, unless file exists.
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>
</VirtualHost>
```

### 1.4 Activer le site et redémarrer Apache

```bash
a2ensite glpi
a2enmod rewrite
systemctl restart apache2
```

> ⚠️ Si des extensions PHP manquantes sont signalées lors de l'installation, les installer via :
> ```bash
> apt install php-<nom_de_lextension>
> systemctl restart apache2
> ```

### 1.5 Configurer la résolution DNS locale (client)

Sur le poste client qui accède à l'interface web, ajouter dans `/etc/hosts` :

```
IP_SERVEUR_WEB    rechidov.glpi
```

---

## 2. Installation via l'interface web

### 2.1 Accéder à l'installateur

Ouvrir un navigateur et aller sur :

- `http://rechidov.glpi`
- ou en fallback : `http://ip_serveur/glpi/glpi/public`

### 2.2 Configurer la base de données

Pendant l'installation, renseigner les informations de connexion à la base de données.

Créer au préalable un utilisateur MySQL/MariaDB dédié :

```sql
CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';
GRANT ALL ON glpi.* TO 'user'@'localhost';
```

Puis créer la base de données `glpi` depuis l'interface web d'installation.

---

## 3. Activer l'inventaire GLPI

Une fois GLPI installé, activer la fonctionnalité d'inventaire :

```
Administration → Inventaire → Activer l'inventaire
```

---

## 4. Installation de l'agent GLPI

> 📖 [Documentation officielle GLPI Agent](https://glpi-agent.readthedocs.io/en/latest/installation/)

### 4.1 Installation manuelle

```bash
wget https://github.com/glpi-project/glpi-agent/releases/download/1.17/glpi-agent-1.17-linux-installer.pl
apt install -y perl
```

Pour consulter les options disponibles :

```bash
perl glpi-agent-1.17-linux-installer.pl --help
```

Lancer l'installation :

```bash
perl glpi-agent-1.17-linux-installer.pl
```

Lors de la question **"Provide an URL..."**, saisir :

```
rechidov.glpi/glpi
```

> 💡 Il s'agit du `ServerName` défini dans le VirtualHost Apache de GLPI.

### 4.2 Vérification de l'agent

```bash
curl http://localhost:62354
```

Ou depuis un navigateur : `http://localhost:62354`

### 4.3 Enregistrement dans l'inventaire

Sur le serveur GLPI, vérifier que la machine apparaît dans l'inventaire.

Si elle n'y est pas (l'agent se présente toutes les heures environ), forcer l'envoi manuellement :

```bash
glpi-agent --force --debug
```

---

## 5. Automatisation via Puppet

Ajouter les blocs suivants dans le fichier `site.pp` :

```puppet
#---------------------------------------------------------
# GLPI AGENT
#---------------------------------------------------------

# Téléchargement de l'installateur
exec { 'download-glpi-agent':
  command => '/usr/bin/wget -q https://github.com/glpi-project/glpi-agent/releases/download/1.17/glpi-agent-1.17-linux-installer.pl -O /tmp/glpi-agent-installer.pl',
  creates => '/tmp/glpi-agent-installer.pl',
}

# Installation de Perl (dépendance)
package { 'perl':
  ensure => present,
}

# Installation de l'agent GLPI
exec { 'install-glpi-agent':
  command => '/usr/bin/perl /tmp/glpi-agent-installer.pl --server=rechidov.glpi --no-question',
  creates => '/usr/bin/glpi-agent',
  require => [Exec['download-glpi-agent'], Package['perl'], Host['rechidov.glpi']],
}

# Démarrage et activation du service
service { 'glpi-agent':
  ensure  => running,
  enable  => true,
  require => Exec['install-glpi-agent'],
}

# Résolution locale du serveur GLPI
host { 'rechidov.glpi':
  ensure       => present,
  ip           => 'IP_SERVEUR',   # ← Remplacer par l'IP réelle du serveur GLPI
  host_aliases => [],
}

# Forcer l'inventaire immédiatement après l'installation
# (évite d'attendre ~1h) — optionnel
exec { 'glpi-agent-force-inventory':
  command     => '/usr/bin/glpi-agent --force',
  refreshonly => true,
  subscribe   => Exec['install-glpi-agent'],
  require     => Service['glpi-agent'],
}
```

> 🔁 Le bloc `glpi-agent-force-inventory` est **optionnel** mais recommandé : il enregistre le poste dans l'inventaire GLPI immédiatement après l'installation, sans attendre le délai d'une heure.

---

*Documentation rédigée pour GLPI 11.0.7 et GLPI Agent 1.17.*

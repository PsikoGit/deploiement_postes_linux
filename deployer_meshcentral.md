# MeshCentral — Installation et configuration

> Supervision et bureau à distance, avec automatisation Puppet.

---

## Sommaire

- [Installation du serveur](#installation-du-serveur)
- [Configuration](#configuration)
- [Accès à l'interface web](#accès-à-linterface-web)
- [Ajout d'un poste client](#ajout-dun-poste-client)
- [Bureau à distance — Passer de Wayland à Xorg](#bureau-à-distance--passer-de-wayland-à-xorg)
- [Automatisation via Puppet](#automatisation-via-puppet)

---

## Installation du serveur

### 1. Installer Node.js

```bash
curl -sL https://deb.nodesource.com/setup_20.x | bash -
apt install nodejs -y

# Vérifier les versions
node -v
npm -v
```

### 2. Installer MeshCentral

```bash
mkdir /opt/meshcentral
cd /opt/meshcentral

npm install meshcentral
node node_modules/meshcentral --install
```

### 3. Vérifier que le service est actif

```bash
systemctl status meshcentral
```

---

## Configuration

Modifier le fichier `/opt/meshcentral/meshcentral-data/config.json` pour éviter les conflits de ports avec Apache et renseigner l'IP du serveur :

```json
{
  "settings": {
    "port": 8086,
    "redirport": 8080,
    "cert": "IP_SERVEUR"
  }
}
```

> `port` et `redirport` sont changés car les ports par défaut (80/443) sont déjà utilisés par Apache.
> `cert` doit correspondre exactement à l'IP joignable par les agents, sinon le certificat TLS sera généré avec `localhost` et les agents ne pourront pas se connecter.

Puis redémarrer MeshCentral :

```bash
systemctl restart meshcentral
```

---

## Accès à l'interface web

Ouvrir un navigateur et accéder à :

```
https://IP_SERVEUR:8086
```

Accepter le certificat auto-signé au premier accès.

### Créer le compte administrateur

Créer le premier compte utilisateur — il sera automatiquement promu administrateur.

Une fois le compte créé, bloquer les nouvelles inscriptions dans `config.json` :

```json
{
  "settings": {
    "port": 8086,
    "redirport": 8080,
    "cert": "IP_SERVEUR",
    "newAccounts": false
  }
}
```

```bash
systemctl restart meshcentral
```

---

## Ajout d'un poste client

Dans l'interface web :

1. Créer un **groupe d'appareils**, type : *Gérer à l'aide d'un agent logiciel*
2. Récupérer la commande d'installation générée
3. Exécuter cette commande sur le poste client

Une fois installé, le poste apparaît dans l'interface et donne accès à :

| Fonctionnalité | Détail |
|---|---|
| **Processus** | Liste des processus en cours |
| **Fichiers / répertoires** | Accès complet, création, suppression |
| **Terminal distant** | Shell root à distance |
| **Bureau à distance** | Voir section suivante |

> ⚠️ L'accès est en **root total** sur les machines supervisés. Le serveur MeshCentral doit être correctement sécurisé et l'accès très restreint !.

---

## Bureau à distance — Passer de Wayland à Xorg

MeshCentral ne supporte pas Wayland. Il faut basculer la session sur Xorg.

### 1. Modifier la configuration GDM3

```bash
nano /etc/gdm3/daemon.conf
```

Décommenter la ligne suivante dans la section `[daemon]` :

```ini
[daemon]
WaylandEnable=false
```

### 2. Redémarrer GDM3

```bash
systemctl restart gdm3
```

### 3. Vérifier

```bash
echo $XDG_SESSION_TYPE
```

Doit retourner `x11`.

Le bureau à distance est maintenant accessible depuis l'interface web MeshCentral.

---

## Automatisation via Puppet

Dans le fichier `site.pp`, ajouter les blocs suivants :

### Installation de l'agent MeshCentral

```puppet
#---------------------------------------------------------
# MESHCENTRAL AGENT
#---------------------------------------------------------

exec { 'download-meshinstall':
    command => '/usr/bin/wget "https://IP_SERVEUR:8086/meshagents?script=1" --no-check-certificate -O /tmp/meshinstall.sh',
    creates => '/tmp/meshinstall.sh',
}

exec { 'chmod-meshinstall':
    command => '/bin/chmod 755 /tmp/meshinstall.sh',
    require => Exec['download-meshinstall'],
}

exec { 'install-meshagent':
    command => '/tmp/meshinstall.sh https://IP_SERVEUR:8086 \'<chaine_meshid>\'',
    creates => '/usr/sbin/meshagent',
    require => Exec['chmod-meshinstall'],
}

service { 'meshagent':
    ensure  => running,
    enable  => true,
    require => Exec['install-meshagent'],
}
```

> Remplacer `<chaine_meshid>` par le deuxième argument de la commande générée dans l'interface web MeshCentral (longue chaîne de caractères propre à ton groupe d'appareils).

### Désactivation de Wayland

```puppet
#---------------------------------------------------------
# DESACTIVER WAYLAND -> XORG
#---------------------------------------------------------

exec { 'disable-wayland':
    command => '/bin/sed -i "s/#WaylandEnable=false/WaylandEnable=false/" /etc/gdm3/daemon.conf',
    onlyif  => '/bin/grep -q "#WaylandEnable=false" /etc/gdm3/daemon.conf',
    notify  => Exec['restart-gdm3'],
}

exec { 'restart-gdm3':
    command     => '/bin/systemctl restart gdm3',
    refreshonly => true,
}
```

> `onlyif` évite de rejouer la commande si Wayland est déjà désactivé. `notify` redémarre GDM3 uniquement si le fichier a été modifié.

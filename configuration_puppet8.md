# Guide d'installation et de configuration — Puppet 8

> ⚠️ Le serveur Puppet sera sur Debian 12 et les ordinateurs que je déploie seront sur Debian 13 ⚠️

> 📄 **Documentation officielle :** [puppet-8.10.0.pdf](https://help.puppet.com/core/media/pdf/puppet-8.10.0.pdf)

---

## Sommaire

1. [Installation du PuppetServer](#1-installation-du-puppetserver)
2. [Configuration du PuppetServer](#2-configuration-du-puppetserver)
3. [Gestion des certificats (autosign)](#3-gestion-des-certificats-autosign)
4. [Script d'installation des agents](#4-script-dinstallation-des-agents)
5. [Vérification côté client](#5-vérification-côté-client)
6. [Commandes utiles](#6-commandes-utiles)
7. [Debug](#7-debug)

---

## 1. Installation du PuppetServer

> ⚠️ Ici j'installe la version opensource de PuppetServer, il existe aussi une version Puppet Core dont les paquets auront pour racine dans l'URL `https://apt-puppetcore.puppet.com/public/`, ce n'est pas ce qu'on utilisera
>
> Choisir le bon `.deb` selon la version de Debian ! Ici j'utilise `puppet8-release-bookworm.deb` car le serveur tourne sous **Debian 12 (Bookworm)**, le paquet .deb n'est pas disponible pour **Debian 13 (Trixie)** au moment où j'écris cette documentation (mai 2026).

```bash
wget https://apt.puppet.com/puppet8-release-bookworm.deb
dpkg -i puppet8-release-bookworm.deb
apt-get update
apt-get install puppetserver
```

---

## 2. Configuration du PuppetServer

### Ajustement de la mémoire JVM

Par défaut, Puppet alloue 2 Go de RAM à la JVM. Sur une machine modeste, réduire à 1 Go :

```bash
nano /etc/default/puppetserver
```

Modifier la ligne suivante :

```diff
- JAVA_ARGS="-Xms2g -Xmx2g ..."
+ JAVA_ARGS="-Xms1g -Xmx1g ..."
```

### Démarrage et activation du service

```bash
systemctl start puppetserver
systemctl enable puppetserver
```

### Vérification

```bash
systemctl status puppetserver
/opt/puppetlabs/bin/puppet --version
ss -lntp | grep 8140
```

### Configuration `puppet.conf` (côté serveur)

Dans `/etc/puppetlabs/puppet/puppet.conf`, ajouter la section suivante :

```ini
[main]
server   = LE_HOSTNAME_DU_SERVEUR
certname = LE_HOSTAME_DU_SERVEUR
```

vérifiez bien que le serveur sait résoudre son propre hostname : `ping $(hostname -f)`

---

## 3. Gestion des certificats (autosign)

Deux approches possibles :

- **Manuelle** : désactiver l'autosign et signer les certificats à la main depuis le serveur.
- **Automatique** : trois méthodes disponibles, décrites dans la [documentation officielle](https://help.puppet.com/core/current/Content/PuppetCore/ssl_autosign.htm).

On va utiliser l'approche [Policy-based autosigning](https://help.puppet.com/core/current/Content/PuppetCore/ssl_policy_based_autosigning.htm), c'est l'option la plus sécurisée et flexible. On indique au serveur un script (dans n'importe quelle langage de programmation)
pour la signature automatique des certificats, le serveur exécute ce script avec le certname (hostname) du client en argument, et le PEM-encoded CSR sur la sortie standard (stdin). Puis acceptera la signature automatique selon la sortie du script, true -> on signe automatiquement, false -> on ne signe pas automatiquement.

On se contentera d'un script qui vérifie si le certname (hostname) du client commence par `exemple-`.

On modifie le fichier `/etc/puppetlabs/puppet/puppet.conf` et ajouter la ligne _autosign_ sous la section [server], afin d'indiquer le script à exécuter :

```bash
[server]
...
autosign = /etc/puppetlabs/puppet/autosign.sh
```

On crée le fichier `/etc/puppetlabs/puppet/autosign.sh` :

```bash
#!/bin/bash
# =============================================================================
# autosign.sh — Policy-based autosigning Puppet 8
# Le master passe le certname en $1 et la CSR en PEM sur stdin
# exit 0 = approuvé, exit 1 = refusé
# =============================================================================
 
CERTNAME="$1"
 
if [[ "$CERTNAME" == exemple-* ]]; then
    exit 0
else
    exit 1
fi
```

Puis :

```bash
chown puppet:puppet /etc/puppetlabs/puppet/autosign.sh 
chmod 750 /etc/puppetlabs/puppet/autosign.sh 
systemctl restart puppetserver
```

---

## 4. Script d'installation des agents

L'objectif est que les clients iPXE soient automatiquement des agents Puppet afin que le PuppetServer leur envoie la configuration nécessaire. On utilisera l'instruction **late_command** du [fichier preseed](deployer_poste_linux.md#82-automatisation-debian).

On va faire exécuter automatiquement un script sur les clients qui sera récupéré depuis le serveur.

`nano /var/www/html/debian/preseed.cfg`

Rajouter à la fin du fichier

```bash
d-i preseed/late_command string \
    in-target wget -q -O /tmp/puppet-bootstrap.sh http://IP_SERVEUR/scripts/puppet-bootstrap.sh ; \
    in-target chmod +x /tmp/puppet-bootstrap.sh ; \
    in-target /tmp/puppet-bootstrap.sh
```

Le script à exécuter sur chaque machine cliente pour l'enrôler comme agent Puppet automatiquement :

`nano /var/www/html/scripts/puppet-bootstrap.sh`

```bash
#!/bin/bash
set -euo pipefail

PUPPET_SERVER="IP_SERVEUR"
PUPPET_HOSTNAME="HOSTNAME_DU_SERVEUR"

# --- Récupération de l'adresse MAC et construction du hostname ---
MAC=$(ip link show | awk '/ether/ {print $2}' | head -n1 | tr -d ':' | tail -c 9)
HOSTNAME="exemple-${MAC,,}"  # ,, pour mettre en minuscule (Puppet ne supporte pas les majuscules)

# --- Appliquer le hostname ---
echo "${HOSTNAME}" > /etc/hostname
#changer la ligne de l'ancien hostname (celui dans le preseed)
sed -i "s/^127\.0\.1\.1.*/127.0.1.1 ${HOSTNAME}/" /etc/hosts
hostname "${HOSTNAME}"

# --- Permettre aux agents Puppet de résoudre le hostname du PuppetServer ---
if ! grep -q "${PUPPET_HOSTNAME}" /etc/hosts; then
    echo "${PUPPET_SERVER} ${PUPPET_HOSTNAME}" >> /etc/hosts
fi

# --- 1. Ajout du dépôt officiel Puppet 8 ---
wget -q https://apt.puppet.com/puppet8-release-bookworm.deb -O /tmp/puppet8-release-bookworm.deb
dpkg -i /tmp/puppet8-release-bookworm.deb
apt-get update -qq
apt-get install -y puppet-agent

# --- 2. Configuration ---
cat > /etc/puppetlabs/puppet/puppet.conf << PUPPETCONF
[main]
server      = ${PUPPET_HOSTNAME}
certname    = ${HOSTNAME}
runinterval = 30m

PUPPETCONF

# --- 3. Premier run + activation ---
/opt/puppetlabs/bin/puppet agent --test --waitforcert 60 || true
systemctl enable puppet
systemctl start puppet
```

Démarrer le PC client sur le réseau et tester.

---

## 5. Vérification côté client

Une fois le client démarré, tester le bon fonctionnement de l'agent en tant que `root` sur le client :

```bash
/opt/puppetlabs/bin/puppet agent --test
```

Si c'est ok, on va voir comment déployer de la configuration via Puppet, en commençant par le portail Application : [ici !](deployer_portailapplication_via_puppet.md)

---

## 6. Commandes utiles

### Gestion des certificats (depuis le serveur)

| Action | Commande |
|--------|----------|
| Signer tous les certificats en attente | `/opt/puppetlabs/bin/puppetserver ca sign --all` |
| Signer un certificat spécifique | `/opt/puppetlabs/bin/puppetserver ca sign --certname HOSTNAME_DU_CLIENT` |
| Supprimer un certificat | `/opt/puppetlabs/bin/puppetserver ca clean --certname HOSTNAME_DU_CLIENT` |
| Lister tous les certificats (signés + en attente) | `/opt/puppetlabs/bin/puppetserver ca list --all` |

### Tester l'agent (depuis le client)

```bash
/opt/puppetlabs/bin/puppet agent -t
```

### Resoumettre une demande de certificat (depuis le client)

À utiliser en cas de problème de certificat :

```bash
rm -rf /etc/puppetlabs/puppet/ssl
/opt/puppetlabs/bin/puppet agent --test
```

---

## 7. Debug

### Installation Debian bloquée

Si l'installation de Debian se fige, ouvrir un shell de secours :

```
CTRL + ALT + F3
```

Puis consulter les logs en temps réel :

```bash
tail -f /var/log/syslog
```

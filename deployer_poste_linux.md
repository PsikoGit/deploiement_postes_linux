# Configuration PXE / iPXE — Déploiement réseau Ubuntu et Debian

## Sommaire

- [0. Avant-propos](#0-avant-propos)
  - [0.1 Ubuntu](#01-ubuntu)
  - [0.2 Debian](#02-debian)
- [1. Configuration IP statique](#1-ip-statique-sur-le-serveur)
- [2. Installation des paquets](#2-installation-des-paquets)
- [3. Configuration TFTP](#3-configuration-tftp)
- [4. Téléchargement des bootloaders iPXE](#4-téléchargement-des-bootloaders-ipxe)
- [5. Installation du serveur web](#5-serveur-web-pour-ipxe)
- [6. Montage et copie de l'ISO](#6-récupération-de-liso)
- [7. Configuration DHCP](#7-configuration-dhcp-dnsmasq)
- [8. Automatisation des questions](#8-automatisation)
    - [8.1 Automatisation Ubuntu](#81-automatisation-ubuntu)
    - [8.2 Automatisation Debian](#81-automatisation-debian)
- [9. Fichier iPXE](#9-fichier-installipxe)

---

## 0. Avant-propos

### 0.1 Ubuntu
Pour déployer du Ubuntu on peut soit déployer un `Ubuntu Desktop` soit un `Ubuntu Server`, le problème d'Ubuntu Desktop est que l'iso à télécharger fait **5 Go** voir plus selon les versions, c'est beaucoup trop lourd à télécharger. En faisant des recherches il est apparement possible de déployer du Ubuntu Desktop en faisant télécharger aux PC seulement le **kernel**, **l'initrd** et un fichier **squashfs**, le tout pour un total de **1,5 Go** environ ce qui allège beaucoup. J'ai essayer d'utiliser cette solution mais ça n'a pas fonctionner pour moi, j'ai eu des erreurs de type `Unable to find a live system on the network` au moment du poot iPXE client.

Je me suis donc tourner vers `Ubuntu Server` qui est beaucoup plus adapté pour le déploiement automatisé, il faudra cependant construire à la main l'environnement Desktop. Voir plus en détail toute la section 8 [Ici](#8-autoinstall--user-data)

Il existe une solution spécifique à Ubuntu pour le déploiement qui est [MAAS](https://canonical.com/maas). Je n'ai pas eu le temps de l'étudier. 

Il existe des solutions plus généralistes tel que [Foreman](https://theforeman.org/), qui est super bien documenté que j'ai pu tester un peu, il en existe encore d'autres. Ces solutions reposent généralement sur du boot PXE/iPXE combinés avec d'autres outils afin de donner une application tout en un prêt à l'emploi.

Cependant j'ai opté pour une solution manuelle, car il faut passer beaucoup de temps à maitriser les applications comme Foreman, de plus ça permet de mieux comprendre le fonctionnement de l'architecture, d'avoir une solution légère et une maitrise de l'ensemble du processus de déploiement. 

### 0.2 Debian
Debian est bien réputé et facile à configurer pour le déploiement automatisé, il suffit d'extraire le **kernel** et l'**initrd** de l'iso [netboot](http://ftp.debian.org/debian/dists/stable/main/installer-amd64/current/images/netboot/) et de le faire télécharger aux clients PXE/iPXE, pour un total de 50Mo par poste. Cela va drastiquement réduire la charge réseau de la liaison WAN entre les LAN des postes de formation et le serveur qui se trouvera sur le site distant. (En comparaison avec Ubuntu qui devra faire télécharger environ 2.5Go par poste).

De plus, le déploiement automatisé de Debian est beaucoup mieux documenté donc plus facile à mettre en oeuvre. Si on veut éviter une solution manuelle, il faudra se diriger vers des solutions "SaaS" tel que Foreman, qu'on a cité juste avant, et d'autres encore.

## 1. IP statique sur le serveur

J'utilise un Debian 13 pour mon serveur, les versions récentes de Debian utilisent Network Manager par défaut pour la gestion du réseau, voici comment mettre des paramètres IP statiques avec Network Manager. Sinon il est possible de désactiver Network Manager et de modifier le fichier `/etc/network/interfaces`.
```bash
nmcli device show

GENERAL.DEVICE:                         enp0s31f6
GENERAL.TYPE:                           ethernet
GENERAL.HWADDR:                         E4:B9:7A:6E:EC:62
GENERAL.MTU:                            1500
GENERAL.STATE:                          100 (connecté)
GENERAL.CONNECTION:                     Wired connection 1
[...]

nmcli con modify "Wired connection 1" ipv4.addresses 192.168.1.187/24
nmcli con modify "Wired connection 1" ipv4.gateway 192.168.1.254
nmcli con modify "Wired connection 1" ipv4.dns 192.168.1.254
nmcli con modify "Wired connection 1" ipv4.method manual
nmcli con down "Wired connection 1"
nmcli con up "Wired connection 1"
```

---

## 2. Installation des paquets

```bash
apt install tftpd-hpa apache2 wget dnsmasq
```

---

## 3. Configuration TFTP

Les clients PXE récupéreront le bootloader iPXE via TFTP. Rappelons que dans l'architecture finale, ce serveur TFTP se trouvera sur un site distant.

Modifier `/etc/default/tftpd-hpa` :

```ini
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="IP_SERVEUR:69"
TFTP_OPTIONS="--secure"
RUN_DAEMON="yes"
```

```bash
systemctl restart tftpd-hpa.service
```

---

## 4. Téléchargement des bootloaders iPXE

Ces bootloaders permettent de passer de client PXE à client iPXE pour bénéficier des avantages de iPXE.

```bash
cd /srv/tftp/
wget http://boot.ipxe.org/x86_64-efi/ipxe.efi
wget http://boot.ipxe.org/undionly.kpxe
```

> - `ipxe.efi` — pour les PC **UEFI**
> - `undionly.kpxe` — pour les PC **BIOS**
>
> À voir s'il faut prendre `snponly.efi` à la place de `ipxe.efi`, pour l'instant ça fonctionne avec `ipxe.efi`

```bash
ls -lh /srv/tftp/

total 1,2M
-rw-rw-r-- 1 root root 1,1M  7 mai   16:09 ipxe.efi
-rw-rw-r-- 1 root root  70K  7 mai   16:09 undionly.kpxe
```

---

## 5. Serveur web pour iPXE

Créer un lien symbolique entre le répertoire TFTP et Apache :

```bash
mkdir -p /var/www/html/tftpboot
ln -s /srv/tftp/ /var/www/html/tftpboot/
```

Ce qui se trouvera dans le répertoire `/srv/tftp/` sera aussi dans le répertoire `/var/www/html/tftpboot/tftp/`

---

## 6. Récupération de l'ISO

### 6.1 Ubuntu Server

Télécharger l'iso Ubuntu Server
```bash
mkdir /var/www/html/ubuntu
cd /var/www/html/ubuntu
wget https://releases.ubuntu.com/releases/resolute/ubuntu-26.04-live-server-amd64.iso
```
Lien à adapter selon la version souhaitée. Voir [ici](https://releases.ubuntu.com/)

Monter l'ISO et copier les fichiers suivants **initrd** et **kernel** dans le répertoire `/srv/tftp/ubuntu` :
```bash
mount -o loop /var/www/html/ubuntu/ubuntu-26.04-live-server-amd64.iso /mnt/

ls /mnt/
boot  boot.catalog  casper  dists  EFI  md5sum.txt  pool  ubuntu
```

```bash
mkdir /srv/tftp/ubuntu
cp /mnt/casper/{initrd,vmlinuz} /srv/tftp/ubuntu/
umount /mnt/
```

Renommer le fichier iso ubuntu (facultatif)
```bash
mv /var/www/html/ubuntu/ubuntu-26.04-live-server-amd64.iso /var/www/html/ubuntu/ubuntu-server.iso
```

### 6.2 Debian

```bash
mkdir /var/www/html/debian
cd /var/www/html/debian
wget http://ftp.debian.org/debian/dists/stable/main/installer-amd64/current/images/netboot/netboot.tar.gz
tar -xzf netboot.tar.gz
```

Les fichiers qui vont nous intéresser pour la suite sont dans le répertoire `/var/www/html/debian/debian-installer/amd64/`

---

## 7. Configuration DHCP (dnsmasq)

Pour l'infrastructure finale, la configuration devra se faire sur les firewall Stormshield, pour indiquer aux clients qui bootent sur le réseau l'option 066 (next-server) qui sera le serveur TFTP à contacter et l'option 067 (filename) le fichier de boot à récupérer. Il faudra donner le bon fichier de boot selon si l'ordinateur client est en UEFI ou BIOS.

Cependant pour le PoC local c'est le serveur PXE/iPXE lui même qui gère cela via `dnsmasq`, c'est aussi possible avec `isc-dhcp-server`.

Fichier `/etc/dnsmasq.d/pxe.conf` :

```ini
# Interface d'écoute
interface=enp0s31f6

# Pas de DNS
port=0

# Logs pour debug
log-dhcp

# Éviter du trafic réseau inutile
dhcp-option=vendor:PXEClient,6,2b

# ProxyDHCP — on ne distribue pas de paramètres IP car un serveur
# DHCP gère déjà ça
dhcp-range=IP_RESEAU,proxy
#Exemple : dhcp-range=192.168.1.0,proxy

# Éviter des erreurs avec certains vieux clients DHCP
dhcp-no-override

# Détecter si le client est déjà iPXE
dhcp-match=set:ipxe-http,175,19
dhcp-match=set:ipxe-menu,175,39

# On crée le tag ipxe-ok si le client supporte les options iPXE http et menu
tag-if=set:ipxe-ok,tag:ipxe-http,tag:ipxe-menu

# Si le client n'est pas iPXE, on lui envoie le bootloader iPXE

# Client en Legacy BIOS
pxe-service=tag:!ipxe-ok,X86PC,PXE,undionly.kpxe,IP_SERVEUR
# Client en UEFI 32 bit (rare mais sait-on jamais !)
pxe-service=tag:!ipxe-ok,IA32_EFI,PXE,ipxe.efi,IP_SERVEUR
# Client en BC_EFI (carte réseau rare)
pxe-service=tag:!ipxe-ok,BC_EFI,PXE,ipxe.efi,IP_SERVEUR
# Client en UEFI 64 bit (tous les PCs modernes)
pxe-service=tag:!ipxe-ok,X86-64_EFI,PXE,ipxe.efi,IP_SERVEUR

# Si le client est iPXE, on passe à la suite
dhcp-boot=tag:ipxe-ok,http://IP_SERVEUR/install.ipxe
```

```bash
systemctl restart dnsmasq.service
```
Il y a plusieurs manières et syntaxes d'écrire le fichier pour reproduire le même fonctionnement, notamment sur la détection iPXE.

---

## 8. Automatisation

### 8.1 Automatisation Ubuntu

Comme l'idée est d'avoir une installation automatique, on va utiliser le mécanisme de cloud-config, spécifique aux distributions Ubuntu. Le fichier `user-data` permet de répondre automatiquement aux questions normalement posés lors de l'installation Ubuntu. C'est le même principe que `preseed` pour Debian.

```bash
mkdir -p /var/www/html/autoinstall
nano /var/www/html/autoinstall/user-data
```

```yaml
#cloud-config
autoinstall:
  version: 1
  locale: fr_FR
  keyboard:
    layout: fr
    variant: ''
  storage:
    layout:
      name: direct
  identity:
    hostname: ubuntu-poste
    username: soulim
    password: 'HASH_A_GÉNÉRÉ'
  ssh:
    install-server: true
    allow-pw: true
  packages:
    - mate-desktop-environment-core
    - mate-desktop-environment
    - lightdm
    - network-manager
    - network-manager-gnome
    - caja
    - mate-terminal
    - pluma
  updates: security
  shutdown: reboot
```

> **Générer le hash du mot de passe :**
> ```bash
> openssl passwd -6 -salt xyz MOT_DE_PASSE
> ```

Ce fichier comporte énorméments d'options et il est possible de faire beaucoup de choses, comme exécuter des commandes Linux après installation (documentation:https://blog.stephane-robert.info/docs/cloud/cloud-init/). Au niveau des packages on peut installer ce qu'on veut et adapter, cependant, j'ai essayé d'installer des paquets comme `ubuntu-mate-desktop`, `ubuntu-desktop` et `ubuntu-desktop-minimal` mais ça ne fonctionnait pas. 

L'hypothèse est que ces paquets sont beaucoup trop lourd et dépendent certainements d'autres utilitaires d'Ubuntu Desktop. Ce n'est pas adapté pour une installation automatique réseau, il faudra donc installer à la main un environnement Ubuntu Desktop complet (ce que j'ai commencé dans la liste `packages` du fichier user-data). C'est pas très propre et c'est assez compliqué, cependant ça évite d'avoir des choses dont on a pas besoin et ça allège la configuration.


Fichier vide mais nécessaire au fonctionnement de cloud-init.
```bash
touch /var/www/html/autoinstall/meta-data
```

---

### 8.2 Automatisation Debian

Debian utilise le mécanisme preseed pour automatiser la réponse aux questions posés lors de l'installation de l'OS, la documentation est très bien rédigé [Ici](https://www.debian.org/releases/stable/amd64/apb.fr.html). Voici un exemple de fichier preseed [Ici](https://www.debian.org/releases/stable/example-preseed.txt).

```bash
nano /var/www/html/debian/preseed.cfg
```

```bash
d-i debian-installer/locale string fr_FR

d-i keyboard-configuration/xkb-keymap select fr
d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string fr

d-i netcfg/choose_interface select auto
d-i netcfg/hostname string rechidov-test
d-i netcfg/get_domain string

d-i auto-install/enable boolean true
d-i debconf/priority string critical

d-i partman-auto/method string regular
d-i partman-auto/choose_recipe select atomic

d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true

d-i partman-basicfilesystems/no_swap boolean true

d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true

d-i passwd/root-login boolean true
d-i passwd/root-password-crypted password HASH_À_GÉNÉRER

d-i passwd/user-fullname string soulim
d-i passwd/username string soulim
d-i passwd/user-password-crypted password HASH_À_GÉNÉRER

tasksel tasksel/first multiselect standard, gnome-desktop

d-i hw-detect/load_firmware boolean true

d-i finish-install/reboot_in_progress note
```

Pour des raisons de sécurité on ne va pas stocker les mots de passes en clair, il faudra générer le hash du mot de passe via la commande `mkpasswd -m sha-512` du paquet `whois`. Puis copier le résultat à la place de `HASH_À_GÉNÉRER`.  

## 9. Fichier install.ipxe

C'est le fichier final qui sera exécuté par les clients iPXE. Il faudra adapter le fichier pour que ça choisisse une des distributions automatiquement selon ce qu'il sera choisit. Ici je laisse les 2 pour montrer la configuration.
```bash
cat /var/www/html/install.ipxe

#!ipxe
set menu-timeout 30000
set server_ip IP_SERVEUR

menu === Menu de déploiement réseau ===
item --gap --        --- Ubuntu ---
item ubuntu          Ubuntu
item debian	         Debian
item --gap --        --- Outils ---
item shell           Shell iPXE
item exit            Booter sur le disque local
choose --timeout ${menu-timeout} --default server target && goto ${target}

:debian
kernel http://${server_ip}/debian/debian-installer/amd64/linux
initrd http://${server_ip}/debian/debian-installer/amd64/initrd.gz
imgargs linux auto=true priority=critical url=http://${server_ip}/debian/preseed.cfg ip=dhcp ---
boot

:ubuntu
kernel http://${server_ip}/tftpboot/tftp/ubuntu/vmlinuz
initrd http://${server_ip}/tftpboot/tftp/ubuntu/initrd
imgargs vmlinuz root=/dev/ram0 ramdisk_size=1500000 ip=dhcp url=http://${server_ip}/ubuntu/ubuntu-server.iso autoinstall ds=nocloud-net;s=http://${server_ip}/autoinstall/ cloud-config-url=/dev/null
boot

:shell
shell

:exit
exit
```

---

## 10. Notes & explications

### Pourquoi Subiquity et non Live Casper ?

> **Subiquity + autoinstall Ubuntu Server** — l'installation des paquets ubuntu-desktop ne fonctionne pas avec cette méthode.
> Faire télécharger Ubuntu Desktop n'est pas possible car cela représente 6 Go de téléchargement en RAM.
> **On déploie donc Ubuntu Server.**

| Version Ubuntu | Comportement |
|---|---|
| ≤ 18.04 | kernel + initrd pur fonctionnait |
| ≥ 20.04 | Canonical a remplacé `debian-installer` par **subiquity**, qui exige une ISO complète en téléchargement |

- **Subiquity** → pour Ubuntu Server, télécharge l'ISO directement sur le disque
- **Boot casper** → pour Ubuntu Desktop, monte l'OS en RAM puis installe via Ubiquity → très lourd en RAM

### Les 2 grandes méthodes de déploiement Ubuntu

**Méthode 1 : Subiquity (méthode actuelle)**

Téléchargement de l'ISO via HTTP. Un peu lourd et lent.

```
PXE → vmlinuz + initrd → télécharge l'ISO en HTTP → installeur subiquity → lit user-data → installe automatiquement
```

> Pas de NFS, pas de squashfs monté à distance — l'ISO entière est téléchargée localement sur le client puis montée en RAM.

**Méthode 2 : Casper/live via HTTP** *(à explorer)*

Disponible uniquement pour Ubuntu Desktop. Plus rapide que Subiquity.

```
PXE → vmlinuz + initrd → monte squashfs via NFS → système live en RAM → installer
```

> L'ISO Ubuntu Server n'est **pas** une image live. Elle ne contient pas de système à faire tourner en RAM, juste un installeur. C'est pourquoi `boot=casper` et `netboot=nfs` ne fonctionnent pas — ces paramètres sont conçus pour Ubuntu Desktop.

---

## 11. Hostname dynamique avec iPXE (tip communauté)

> **Problème :** difficulté à définir dynamiquement le nom d'hôte de la machine cible lors d'un déploiement automatisé Ubuntu 22.04 via iPXE.

**Solution trouvée :**

Lire une variable avec iPXE et la définir dans le paramètre `ds` :

```
ds=nocloud-net;s=${autoinstall_url};h=${hostname}
```

Puis utiliser un **script de pré-installation** pour injecter la valeur avec `sed` dans `/autoinstall.yaml`.

> La clé est que le YAML est rechargé **après** l'exécution du script de pré-installation.
> La variable de nom d'hôte est stockée dans `/run/cloud-init/instance-data.json` lors de l'installation subiquity initiale.

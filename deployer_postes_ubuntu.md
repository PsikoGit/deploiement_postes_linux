# Configuration PXE / iPXE — Déploiement réseau Ubuntu

## Sommaire

- [0. Avant-propos](#0-avant-propos)
- [1. Configuration IP statique](#1-ip-statique-sur-le-serveur)
- [2. Installation des paquets](#2-installation-des-paquets)
- [3. Configuration TFTP](#3-configuration-tftp)
- [4. Téléchargement des bootloaders iPXE](#4-téléchargement-des-bootloaders-ipxe)
- [5. Installation du serveur web](#5-serveur-web-pour-ipxe)
- [6. Montage et copie de l'ISO](#6-récupération-de-liso-ubuntu)
- [7. Configuration DHCP](#7-configuration-dhcp-dnsmasq)
- [8. Automatisation via cloud-config](#8-autoinstall--user-data)
- [9. Fichier iPXE](#9-fichier-installipxe)

---

## 0. Avant-propos

Pour déployer du Ubuntu on peut soit déployer un Ubuntu Desktop soit un Ubuntu Server, le problème d'Ubuntu Desktop est que l'iso à télécharger fait 5 Go voir plus selon les versions, c'est beaucoup trop lourd à télécharger. En faisant des recherches il est apparement possible de déployer du Ubuntu Desktop en faisant télécharger aux PC seulement le kernel, l'initrd et un fichier squashfs, le tout pour un total de 1,5 Go environ ce qui allège beaucoup. J'ai essayer de faire cette solution mais ça n'a pas fonctionner pour moi, j'ai eu des erreurs de type `Unable to find a live system on the network`, etc...

Je me suis donc tourner vers Ubuntu Server qui est beaucoup plus adapté pour le déploiement automatisé, il faudra cependant construire à la main l'environnement Desktop. Voir plus en détail toute la section 8 [Ici](#8-autoinstall--user-data)

## 1. IP statique sur le serveur

Sur les versions récentes de Debian c'est Network Manager qui est utilisé par défaut, voici comment mettre des paramètres IP statiques avec Network Manager
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

> Le fichier iPXE sera installé à la racine du serveur sous le nom `install.ipxe`.

---

## 6. Récupération de l'ISO Ubuntu

```bash
wget https://releases.ubuntu.com/releases/resolute/ubuntu-26.04-live-server-amd64.iso
```

Monter l'ISO et copier les fichiers suivants dans le répertoire `/srv/tftp/ubuntu` :

```bash
mkdir /mnt/ubuntu-iso
mount -o loop /chemin/vers/fichier/ubuntu-26.04-live-server-amd64.iso /mnt/ubuntu-iso/
```

```bash
ls /mnt/ubuntu-iso/
boot  boot.catalog  casper  dists  EFI  md5sum.txt  pool  ubuntu

mkdir /srv/tftp/ubuntu
cp /mnt/ubuntu-iso/casper/{initrd,vmlinuz} /srv/tftp/ubuntu/
umount /mnt/ubuntu-iso

cp /chemin/vers/fichier/ubuntu-26.04-live-server-amd64.iso /var/www/html/ubuntu-server.iso
```

---

## 7. Configuration DHCP (dnsmasq)

Pour Réseau Canopé la configuration devra se faire sur les firewall Stormshield, pour indiquer aux clients qui bootent sur le réseau l'option 066 (next-server) qui sera le serveur TFTP à contacter et l'option 067 (filename) le fichier de boot à récupérer. Il faudra donner le bon fichier de boot selon si l'ordinateur client est en UEFI ou BIOS.

Cependant pour le PoC local c'est le serveur PXE/iPXE lui même qui gérait cela via `dnsmasq`, c'est aussi possible avec `isc-dhcp-server`.

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
dhcp-range=192.168.1.0,proxy

# Éviter des erreurs avec certains vieux clients DHCP
dhcp-no-override

# Détecter si le client est déjà iPXE
dhcp-match=set:ipxe-http,175,19
dhcp-match=set:ipxe-menu,175,39

# On crée le tag ipxe-ok si le client supporte les options iPXE http et menu
tag-if=set:ipxe-ok,tag:ipxe-http,tag:ipxe-menu

# Si le client n'est pas iPXE, on lui envoie le bootloader iPXE
# Client en Legacy BIOS
pxe-service=tag:!ipxe-ok,X86PC,PXE,undionly.kpxe,192.168.1.187
# Client en UEFI 32 bit (rare mais sait-on jamais !)
pxe-service=tag:!ipxe-ok,IA32_EFI,PXE,ipxe.efi,192.168.1.187
# Client en BC_EFI (carte réseau rare)
pxe-service=tag:!ipxe-ok,BC_EFI,PXE,ipxe.efi,192.168.1.187
# Client en UEFI 64 bit (tous les PCs modernes)
pxe-service=tag:!ipxe-ok,X86-64_EFI,PXE,ipxe.efi,192.168.1.187

# Si le client est iPXE, on passe à la suite
dhcp-boot=tag:ipxe-ok,http://192.168.1.187/install.ipxe
```

```bash
systemctl restart dnsmasq.service
```
Il y a plusieurs manières et syntaxes d'écrire le fichier pur reproduire le même fonctionnement, notamment sur la détection iPXE.

---

## 8. Autoinstall — user-data

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

L'hypothèse est que ces paquets sont beaucoup trop lourd et dépendent certainements d'autres paquets d'Ubuntu Desktop. Ce n'est pas adapté pour une installation automatique réseau, il faudra donc installer à la main un environnement Ubuntu Desktop complet, ce qui peut-être assez compliqué. Cependant ça évite d'avoir des choses dont on a pas besoin et d'alléger la configuration.


Fichier vide mais nécessaire au fonctionnement de cloud-init.
```bash
touch /var/www/html/autoinstall/meta-data
```

---

## 9. Fichier install.ipxe

```bash
cat /var/www/html/install.ipxe

#!ipxe
set menu-timeout 30000
set server_ip 192.168.1.187

menu === Menu de déploiement réseau ===
item --gap --        --- Ubuntu ---
item server          Ubuntu
item --gap --        --- Outils ---
item shell           Shell iPXE
item exit            Booter sur le disque local
choose --timeout ${menu-timeout} --default server target && goto ${target}

:server
kernel http://${server_ip}/tftpboot/tftp/ubuntu/vmlinuz
initrd http://${server_ip}/tftpboot/tftp/ubuntu/initrd
imgargs vmlinuz root=/dev/ram0 ramdisk_size=1500000 ip=dhcp url=http://${server_ip}/ubuntu-server.iso autoinstall ds=nocloud-net;s=http://${server_ip}/autoinstall/ cloud-config-url=/dev/null
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

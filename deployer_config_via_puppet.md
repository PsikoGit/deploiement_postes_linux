Une fois qu'on a fait en sorte que les ordinateurs qu'on déploie [soient automatiquement des agents Puppet](configuration_puppet8.md), on va configurer le serveur Puppet pour qu'il déploie une certaine configuration sur les agents.

La première fonctionnalité qu'on va mettre en place est un portail d'applications filtrés que les clients seront autorisés de télécharger. J'ai essayé de faire ça avec le gnome-software intégré et un dépot debian privé mais la mise en place est très complexe. Je suis donc parti sur une approche avec un script python qui sera poussé sur les agents. Ce script Python affichera une fenêtre avec la liste des applications autorisés par l'administrateur, avec la possibilité d'installer et de désinstaller.

La gestion centralisée des applications se basera sur un fichier json `/var/www/html/apps.json`, grâce à ce fichier on automatisera la liste des logiciels pouvant être installé via l'application Python, on automatisera également l'attribution des droits sudo aux utilisateurs pour l'installation des logiciels.

On fera un portail d'applications avec Firefox et VLC pour la démonstration :

```bash
{
  "apps": [
    {
      "name": "Firefox ESR",
      "package": "firefox-esr",
      "description": "Navigateur web",
      "icon": "firefox-esr",
      "command": "apt install -y firefox"
    },
    {
      "name": "VLC",
      "package": "vlc",
      "description": "Lecteur multimédia",
      "icon": "vlc",
      "command": "apt install -y vlc"
    }
  ]
}
```

Les dictionnaires importants sont package et command : 

La valeur de la clé package sera le nom du paquet une fois installé sur l'ordinateur. Par exemple le logiciel firefox à pour nom de paquet `firefox-esr`, on peut le vérifier en faisant `dpkg -l firefox-esr`. On peut récupérer le nom d'un paquet via la commande `dpkg-deb -f fichier.deb Package`. Le dictionnaire package va nous servir pour supprimer l'application depuis le Portail d'Applications, avec la commande `sudo apt remove nom_du_paquet` qui se lancera quand on clique sur le bouton 'Désinstaller' #ICI Y'AURA UN SCREEN

⚠️ Le script python développé pour le portail d'application prends en compte que les .deb, il faudra le modifier pour ajouter la gestion des AppImage par exemple, qui est répandu. Une version supportant une gamme plus larges de format sera proposé ultérieurement ⚠️

La valeur de la clé command est la commande qui permettra d'installer le logiciel en question. Ce fichier nous sera utile pour avoir une gestion centralisée des applications autorisées et de permettre d'octroyer les droits sudo dynamiquement.

Une fois le fichier apps.json créé sur le serveur à l'emplacement `/var/www/html/apps.json`, on va commencer à configurer le serveur Puppet pour lancer la configuration souhaitée. 

Préparer l'environnement :

```bash
mkdir -p /etc/puppetlabs/code/environments/manage/modules/portail/files
mkdir -p /etc/puppetlabs/code/environments/manage/modules/portail/manifests
touch /etc/puppetlabs/code/environments/manage/modules/portail/manifests/init.pp
```

L'environnement utilisé par défaut est `/etc/puppetlabs/code/environments/production/`, pour faire utiliser l'environnement `/etc/puppetlabs/code/environments/manage/` (celui qu'on vient de créer), il faut que sur les agents Puppet, dans le fichier `/etc/puppetlabs/puppet/puppet.conf` il y ait l'instruction :

```bash
[main]
...
environment = manage
```

Rappelez vous que pour écrire ce fichier on utilisait le [script `/var/www/html/scripts/puppet-bootstrap.sh`](configuration_puppet8.md#4-script-dinstallation-des-agents) qui est exécuté sur les clients à l'installation. On va le modifier en changeant cette partie du script :

```bash
# --- 2. Configuration ---
cat > /etc/puppetlabs/puppet/puppet.conf << PUPPETCONF
[main]
server      = ${PUPPET_HOSTNAME}
certname    = ${HOSTNAME}
environment = manage
runinterval = 30m

PUPPETCONF
```

En rajoutant la ligne `environment = manage`

Ensuite récupérer le script `portail-apps.py` : 

<details>
  <summary>Afficher le script portail-apps.py</summary>

```bash
import gi
gi.require_version('Gtk', '4.0')
from gi.repository import Gtk, GLib
import requests
import subprocess
import threading

SERVER = "http://IP_SERVEUR"

class PortailApps(Gtk.ApplicationWindow):
    def __init__(self, app):
        super().__init__(application=app, title="Portail Applications")
        self.set_default_size(600, 400)

        box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=10)
        box.set_margin_top(10)
        box.set_margin_bottom(10)
        box.set_margin_start(10)
        box.set_margin_end(10)
        self.set_child(box)

        label = Gtk.Label(label="Applications disponibles")
        label.add_css_class("title-1")
        box.append(label)

        self.listbox = Gtk.ListBox()
        self.listbox.set_selection_mode(Gtk.SelectionMode.NONE)
        box.append(self.listbox)

        self.load_apps()

    def load_apps(self):
        try:
            r = requests.get(f"{SERVER}/apps.json")
            apps = r.json()["apps"]
            for app in apps:
                self.add_app_row(app)
        except Exception as e:
            print(f"Erreur: {e}")

    def add_app_row(self, app):
        # Vérifie si le paquet est installé
        result = subprocess.run(
            f"dpkg -l {app['package']} | grep ^ii",
            shell=True,
            capture_output=True
        )
        is_installed = result.returncode == 0

        row = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=5)
        row.set_margin_top(5)
        row.set_margin_bottom(5)
        row.set_margin_start(5)
        row.set_margin_end(5)

        top = Gtk.Box(orientation=Gtk.Orientation.HORIZONTAL, spacing=10)
        info = Gtk.Box(orientation=Gtk.Orientation.VERTICAL)
        name = Gtk.Label(label=app["name"])
        name.set_halign(Gtk.Align.START)
        desc = Gtk.Label(label=app["description"])
        desc.set_halign(Gtk.Align.START)
        desc.add_css_class("dim-label")
        info.append(name)
        info.append(desc)
        top.append(info)

        btn_install = Gtk.Button(label="Installer")
        btn_cancel = Gtk.Button(label="Annuler")
        btn_uninstall = Gtk.Button(label="Désinstaller")

        btn_install.set_hexpand(True)
        btn_install.set_halign(Gtk.Align.END)
        btn_cancel.set_halign(Gtk.Align.END)
        btn_uninstall.set_halign(Gtk.Align.END)

        btn_cancel.set_visible(False)
        btn_install.set_visible(not is_installed)
        btn_uninstall.set_visible(is_installed)

        top.append(btn_install)
        top.append(btn_cancel)
        top.append(btn_uninstall)

        progress = Gtk.ProgressBar()
        progress.set_visible(False)

        status = Gtk.Label(label="Installé" if is_installed else "")
        status.set_halign(Gtk.Align.START)
        status.set_visible(is_installed)

        row.append(top)
        row.append(progress)
        row.append(status)

        btn_install.connect("clicked", self.install_app, app["command"], btn_install, btn_cancel, btn_uninstall, progress, status)
        btn_cancel.connect("clicked", self.cancel_install, btn_install, btn_cancel, progress, status)
        btn_uninstall.connect("clicked", self.uninstall_app, app["package"], btn_install, btn_uninstall, progress, status)

        self.listbox.append(row)

    def install_app(self, btn, command, btn_install, btn_cancel, btn_uninstall, progress, status):
        btn_install.set_sensitive(False)
        btn_cancel.set_visible(True)
        progress.set_visible(True)
        status.set_visible(True)
        status.set_label("Installation en cours...")
        self.process = None
        self._pulse_active = True
        GLib.timeout_add(100, self._pulse_progress, progress)
        threading.Thread(target=self.run_install, args=(command, btn_install, btn_cancel, btn_uninstall, progress, status)).start()


    def _pulse_progress(self, progress):
        if self._pulse_active:
            progress.pulse()
            return True
        return False

    def cancel_install(self, btn, btn_install, btn_cancel, progress, status):
        if self.process:
            self.process.terminate()
        self._pulse_active = False
        btn_install.set_label("Installer")
        btn_install.set_sensitive(True)
        btn_cancel.set_visible(False)
        progress.set_visible(False)
        status.set_label("Installation annulée")

    def run_install(self, command, btn_install, btn_cancel, btn_uninstall, progress, status):
        self.process = subprocess.Popen(["sudo", "bash", "-c", command])
        self.process.wait()
        returncode = self.process.returncode
        self.process = None
        self._pulse_active = False
        GLib.idle_add(self.install_done, btn_install, btn_cancel, btn_uninstall, progress, status, returncode == 0)


    def install_done(self, btn_install, btn_cancel, btn_uninstall, progress, status, success):
        progress.set_visible(False)
        btn_cancel.set_visible(False)
        if success:
            btn_install.set_visible(False)
            btn_uninstall.set_visible(True)
            status.set_label("Installé ✓")
        else:
            btn_install.set_sensitive(True)
            status.set_label("Échec de l'installation")
        return False

    def uninstall_app(self, btn, package, btn_install, btn_uninstall, progress, status):
        btn_uninstall.set_sensitive(False)
        progress.set_visible(True)
        status.set_visible(True)
        status.set_label("Désinstallation en cours...")
        self._pulse_active = True
        GLib.timeout_add(100, self._pulse_progress, progress)
        threading.Thread(target=self.run_uninstall, args=(package, btn_install, btn_uninstall, progress, status)).start()

    def run_uninstall(self, package, btn_install, btn_uninstall, progress, status):
        self.process = subprocess.Popen(["sudo", "apt", "remove", "-y", package])
        self.process.wait()
        returncode = self.process.returncode
        self.process = None
        self._pulse_active = False
        GLib.idle_add(self.uninstall_done, btn_install, btn_uninstall, progress, status, returncode == 0)

    def uninstall_done(self, btn_install, btn_uninstall, progress, status, success):
        progress.set_visible(False)
        if success:
            btn_uninstall.set_visible(False)
            btn_install.set_visible(True)
            btn_install.set_sensitive(True)
            status.set_label("Désinstallé")
        else:
            btn_uninstall.set_sensitive(True)
            status.set_label("Échec de la désinstallation")
        return False

class App(Gtk.Application):
    def __init__(self):
        super().__init__(application_id="com.entreprise.portail")

    def do_activate(self):
        win = PortailApps(self)
        win.present()

App().run()
```  
</details>

Et le mettre à l'emplacement `/etc/puppetlabs/code/environments/manage/modules/portail/files`

Voici un aperçu de l'application Python #ICI DEUXIEME SCREEN

Prochaine étape on va écrire le manifest Puppet sur le serveur, c'est le fichier qui permet de spécifier les instructions pour les clients.

`nano /etc/puppetlabs/code/environments/manage/manifests/exemple.pp`

<details>
  <summary>Afficher le fichier exemple.pp</summary>
  
```bash
node /^exemple-/ {
  file { '/usr/local/bin/portail-apps.py':
    ensure => present,
    source => 'puppet:///modules/portail/portail-apps.py',
    mode   => '0755',
  }

file { '/etc/sudoers.d/portail':
  ensure => present,
  source => 'puppet:///modules/portail/portail-sudoers',
  mode   => '0440',
  owner  => 'root',
  group  => 'root',
}

  file { '/usr/share/applications/portail.desktop':
    ensure  => present,
    content => "[Desktop Entry]
Name=Portail Applications
Exec=python3 /usr/local/bin/portail-apps.py
Icon=system-software-install
Terminal=false
Type=Application
Categories=System;
",
  }

file { '/home/soulim/Bureau/portail.desktop':
  ensure  => present,
  content => "[Desktop Entry]
Name=Portail Applications
Exec=python3 /usr/local/bin/portail-apps.py
Icon=system-software-install
Terminal=false
Type=Application
Categories=System;
",
  owner => 'soulim',
  mode  => '0755',
}

package { 'gnome-shell-extension-desktop-icons-ng':
  ensure => present,
}

exec { 'enable-ding':
  command => '/bin/su -c "gnome-extensions enable ding@rastersoft.com" soulim',
  require => Package['gnome-shell-extension-desktop-icons-ng'],
}

}
```

</details>

Explications : 

`node /^exemple-/` : permet d'appliquer la configuration aux noeuds dont le hostname débute par `exemple-`

```bash
 file { '/usr/local/bin/portail-apps.py':
    ensure => present,
    source => 'puppet:///modules/portail/portail-apps.py',
    mode   => '0755',
  }
``` : créer le fichier `/usr/local/bin/portail-apps.py` sur les clients, le contenu de ce fichier sera celui de `/etc/puppetlabs/code/environments/manage/modules/portail/files/portail-apps.py`. C'est pratique car, en cas de problème, ça permet de modifier le script depuis le serveur, puis le script sera envoyé automatiquement aux clients.

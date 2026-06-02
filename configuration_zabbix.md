Pour installer Zabbix aller sur ce lien : https://www.zabbix.com/download <br>
Documentation officiel de Zabbix : https://www.zabbix.com/documentation/7.4/en/manual

Chosir les composants que vous souhaitez, moi j'ai pris ceux là : Debian, Bookworm, Server Frontend Agent, MySQL, Apache

Installer zabbix repository : 

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian12_all.deb
dpkg -i zabbix-release_latest_7.4+debian12_all.deb
apt update
```

Installer Zabbix Server, Agent et Frontend : 

```bash
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

Installer la base de donnée, moi je vais utiliser mariadb :

```bash
apt install mariadb-server
mysql -u root -p
#ou faire sudo mysql

mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@localhost identified by 'password';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;

zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix

mysql -u root -p
#ou faire sudo mysql

mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```

Modifier la ligne `DBPassword=password` du fichier `/etc/zabbix/zabbix_server.conf`

Lancer le service Zabbix Agent et Serveur : 

```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```

Ensuite aller sur l'URL `http://IP_SERVEUR/zabbix` et faire l'installation Web, une fois terminé, le login par défaut : `Admin` et mot de passe par défaut : `zabbix`

Créer un utilisateur normal pour l'appli web Zabbix : https://www.zabbix.com/documentation/7.4/en/manual/quickstart/login

Faire en sorte que le PC soit Zabbix client automatiquement : 

Installer Zabbix Agents sur le client : https://www.zabbix.com/download?zabbix=7.4&os_distribution=debian&os_version=13&components=agent&db=&ws=
et suivre cette doc :https://www.zabbix.com/documentation/7.4/en/manual/discovery/auto_registration?hl=agent%2Cautoregistration

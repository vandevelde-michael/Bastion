# Bastion
## Guide d'installation du Bastion Apache Guacamole

### 1. Introduction

Ce document détaille **étape par étape** l'installation et la configuration d'un bastion sécurisé basé sur Apache Guacamole pour accéder via navigateur à des sessions RDP, VNC et SSH.

### 2. Architecture et composants

* **VM Bastion** :

  * OS : Debian 12 (CLI)
  * Logiciels : guacd, Tomcat9, base SQL, Nginx
* **VM Linux GUI** : Debian 12 + environnement Xfce (VNC)
* **VM Linux CLI** : Debian 12 (SSH)
* **VM Windows** : Windows 10 (RDP)
* **Réseau** :

  * Interface `eth0` (frontale) : accès web (port 80/443)
  * Interface `eth1` (privée) : accès interne aux VM cibles

### 3. Prérequis

1. Accès root ou utilisateur sudo sur la VM Bastion
2. Internet pour récupérer paquets
3. Ports à ouvrir :

   * 4822 (guacd)
   * 8080 (Tomcat)
   * 80/443 (HTTP/HTTPS)
4. Nom de domaine ou IP fixe pour le bastion

### 4. Installation de guacd

```bash
sudo apt update
sudo apt install -y guacd
sudo systemctl enable --now guacd
```

Vérifier :

```bash
systemctl status guacd
```

### 5. Installation de Tomcat et de l'application

1. Installer Tomcat9 :

   ```bash
   sudo apt install -y tomcat9 tomcat9-admin
   ```
2. Télécharger la Web App Guacamole WAR :

   ```bash
   wget https://apache.org/dist/guacamole/1.5.0/binary/guacamole-1.5.0.war -O guacamole.war
   ```
3. Copier dans Tomcat :

   ```bash
   sudo mv guacamole.war /var/lib/tomcat9/webapps/
   ```
4. Créer dossier config Guacamole :

   ```bash
   sudo mkdir /etc/guacamole
   sudo chown tomcat:tomcat /etc/guacamole
   ```
5. Lier configura­tion :

   ```bash
   sudo ln -s /etc/guacamole /usr/share/tomcat9/.guacamole
   sudo systemctl restart tomcat9
   ```

### 6. Installation et configuration de la base SQL

1. Installer MariaDB :

   ```bash
   sudo apt install -y mariadb-server
   ```
2. Sécuriser l'install :

   ```bash
   sudo mysql_secure_installation
   ```
3. Créer base et utilisateur :

   ```sql
   CREATE DATABASE guacamole_db;
   CREATE USER 'guacuser'@'localhost' IDENTIFIED BY 'ChangeMe!';
   GRANT ALL PRIVILEGES ON guacamole_db.* TO 'guacuser'@'localhost';
   FLUSH PRIVILEGES;
   ```
4. Charger le schéma JDBC :

   ```bash
   wget https://apache.org/dist/guacamole/1.5.0/binary/guacamole-auth-jdbc-1.5.0.tar.gz
   tar -xzf guacamole-auth-jdbc-1.5.0.tar.gz
   cat guacamole-auth-jdbc-1.5.0/mysql/schema/*.sql | mysql -u root -p guacamole_db
   ```

### 7. Configuration de Guacamole

Créer `/etc/guacamole/guacamole.properties` :

```
guacd-hostname: localhost
guacd-port: 4822
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacuser
mysql-password: ChangeMe!
```

Redémarrer services :

```bash
sudo systemctl restart tomcat9 guacd
```

### 8. Configuration du reverse-proxy HTTPS (Nginx)

1. Installer Nginx :

   ```bash
   sudo apt install -y nginx
   ```
2. Activer site Guacamole : `/etc/nginx/sites-available/guacamole` :

```
server {
  listen 80;
  server_name bastion.example.com;
  location / {
    return 301 https://$host$request_uri;
  }
}
server {
  listen 443 ssl http2;
  server_name bastion.example.com;

  ssl_certificate /etc/ssl/certs/fullchain.pem;
  ssl_certificate_key /etc/ssl/private/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:8080/guacamole/;
    proxy_buffering off;
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
}
```

3. Activer et recharger :

```bash
sudo ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 9. Configuration des connexions et des utilisateurs

1. Accéder à l'interface : `https://bastion.example.com`

   * Login initial : `guacadmin` / `guacadmin`
2. Modifier mot de passe admin
3. Dans **Settings > Connections**, créer :

   * **SSH** vers VM CLI (IP privée)
   * **VNC** vers Linux GUI
   * **RDP** vers Windows 10
4. Dans **Users & Groups**, créer groupes et attribuer connexions

### 10. Tests et validation

* Tester chaque connexion depuis le navigateur
* Vérifier transferts de fichiers et redimensionnement
* Contrôler logs :

  * `/var/log/guacd/guacd.log`
  * `/var/log/tomcat9/catalina.out`
  * `/var/log/nginx/error.log`

### 11. Sécurisations complémentaires

* Forcer MFA (plugin TOTP)
* Durcir Firewall (UFW ou iptables)
* Mettre à jour régulièrement (`apt upgrade`)

---

*Fin du guide d'installation.*

#PROJET BASTION

## Guide d'installation du Bastion Apache Guacamole

### 1. Introduction

Ce guide détaille **étape par étape** l'installation d'un bastion sécurisé sur Debian 12, basé sur Apache Guacamole. Chaque étape inclut la raison d'être (le « pourquoi ») pour comprendre l’intérêt de chaque composant.

### 2. Architecture et composants

* **VM Bastion** :

  * **OS : Debian 12 (CLI)** – version stable, légère, et supportée pour serveurs.
  * **Logiciels** :

    * **guacd** : démon qui convertit les protocoles distants (RDP, VNC, SSH) en flux WebSocket pour le navigateur.
    * **Tomcat9** : conteneur Java pour héberger l’application Web Guacamole.
    * **Base SQL** (MariaDB) : stocke les utilisateurs, connexions et paramètres de manière persistante et sécurisée.
    * **Nginx** : sert de reverse‑proxy HTTPS, offrant chiffrement et protection contre certaines attaques.
* **VM Linux GUI** : Debian 12 + Xfce – cible VNC pour tester l’accès GUI.
* **VM Linux CLI** : Debian 12 – cible SSH pour accès ligne de commande.
* **VM Windows** : Windows 10 – cible RDP pour accès poste de travail.
* **Réseau** :

  * `eth0` (frontale) : expose les ports web (80/443) au monde extérieur.
  * `eth1` (privée) : communication interne sécurisée vers les VM cibles.

### 3. Prérequis

Avant de démarrer, assurez-vous de :

1. Avoir un **utilisateur sudo** ou root sur la VM Bastion pour installer des paquets.
2. Avoir **Internet** pour télécharger les logiciels.
3. **Ouvrir les ports** essentiels :

   * **4822** (guacd) : flux interne entre guacd et l’app Web.
   * **8080** (Tomcat) : Tomcat écoute là, mais sera protégé par Nginx.
   * **80/443** : HTTP/HTTPS pour l’accès utilisateur final.
4. Disposer d’un **nom de domaine** pointant vers l’IP fixe du bastion pour un accès SSL correct.

### 4. Installation de guacd (le moteur de protocole)

**Pourquoi ?** guacd traduit les flux RDP/VNC/SSH en WebSockets, sans lui le navigateur ne pourrait pas se connecter.

```bash
sudo apt update
sudo apt install -y guacd
sudo systemctl enable --now guacd
```

Vérifier que le démon tourne :

```bash
systemctl status guacd
```

### 5. Installation de Tomcat9 et déploiement de l’application

**Pourquoi Tomcat ?** Guacamole Web App est une application Java (WAR) nécessitant un conteneur Java.

1. Installer Tomcat9 et l’interface d’administration :

   ```bash
   sudo apt install -y tomcat9 tomcat9-admin
   ```
2. Télécharger la version stable de Guacamole Web App (WAR) :

   ```bash
   wget https://apache.org/dist/guacamole/1.5.0/binary/guacamole-1.5.0.war -O guacamole.war
   ```
3. Placer le WAR dans le répertoire de déploiement Tomcat :

   ```bash
   sudo mv guacamole.war /var/lib/tomcat9/webapps/
   ```
4. Créer le dossier de configuration :

   ```bash
   sudo mkdir /etc/guacamole
   sudo chown tomcat:tomcat /etc/guacamole
   ```
5. Lier cette config au home Tomcat, pour qu’il la charge :

   ```bash
   sudo ln -s /etc/guacamole /usr/share/tomcat9/.guacamole
   sudo systemctl restart tomcat9
   ```

### 6. Base de données MariaDB – pourquoi une base SQL ?

Guacamole stocke persistent les utilisateurs, mots de passe, et connexions. Une base relationnelle garantit intégrité et possibilité de sauvegarde.

1. Installer MariaDB :

   ```bash
   sudo apt install -y mariadb-server
   ```
2. Sécuriser l’installation (supprime comptes anonymes, désactive remote root, etc.) :

   ```bash
   sudo mysql_secure_installation
   ```
3. Créer la base et l’utilisateur dédié :

   ```sql
   CREATE DATABASE guacamole_db;
   CREATE USER 'guacuser'@'localhost' IDENTIFIED BY 'ChangeMe!';
   GRANT ALL PRIVILEGES ON guacamole_db.* TO 'guacuser'@'localhost';
   FLUSH PRIVILEGES;
   ```
4. Charger le schéma JDBC fourni par Guacamole :

   ```bash
   wget https://apache.org/dist/guacamole/1.5.0/binary/guacamole-auth-jdbc-1.5.0.tar.gz
   tar -xzf guacamole-auth-jdbc-1.5.0.tar.gz
   cat guacamole-auth-jdbc-1.5.0/mysql/schema/*.sql | mysql -u root -p guacamole_db
   ```

### 7. Configuration de l’application Guacamole

**Pourquoi ce fichier ?** `guacamole.properties` lie l’appli Java à guacd et à la base de données.
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

Appliquer :

```bash
sudo systemctl restart tomcat9 guacd
```

### 8. Reverse‑proxy HTTPS avec Nginx (sécurisation de l’accès)

**Pourquoi Nginx ?** Pour fournir TLS (HTTPS) et protéger Tomcat, démultiplexer l’accès HTTP/HTTPS.

1. Installer Nginx :

   ```bash
   sudo apt install -y nginx
   ```
2. Créer le fichier `/etc/nginx/sites-available/guacamole` :

```
server {
  listen 80;
  server_name bastion.example.com;
  # Redirection HTTP -> HTTPS
  return 301 https://$host$request_uri;
}
server {
  listen 443 ssl http2;
  server_name bastion.example.com;

  # Chemins vers vos certificats TLS
  ssl_certificate /etc/ssl/certs/fullchain.pem;
  ssl_certificate_key /etc/ssl/private/privkey.pem;

  location / {
    # Proxy vers Tomcat
    proxy_pass http://127.0.0.1:8080/guacamole/;
    proxy_buffering off;
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
}
```

3. Activer et tester :

```bash
sudo ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 9. Configuration des connexions et des utilisateurs (interface Web)

**Pourquoi ?** Définir qui peut accéder à quoi, et comment.

1. Visiter `https://bastion.example.com` et se connecter (par défaut `guacadmin`/`guacadmin`).
2. Changer le mot de passe admin.
3. Dans **Settings > Connections**, ajouter :

   * **SSH** vers VM CLI (IP privée)
   * **VNC** vers Linux GUI
   * **RDP** vers Windows 10
4. Dans **Users & Groups**, créer des groupes (Admin, Linux, Windows) et assigner les connexions.

### 10. Tests et validation

* **Pourquoi ?** Pour vérifier la fiabilité et le bon fonctionnement.
* Tester chaque protocole depuis le navigateur.
* Vérifier le transfert de fichiers et le redimensionnement d’écran.
* Inspecter les logs pour détecter erreurs :

  * `/var/log/guacd/guacd.log`
  * `/var/log/tomcat9/catalina.out`
  * `/var/log/nginx/error.log`

### 11. Mesures de sécurité supplémentaires

* **MFA (TOTP)** : ajoute une couche supplémentaire d’authentification.
* **Firewall** : restreindre les accès à `eth1` (UFW ou iptables).
* **Mises à jour** : planifier `apt upgrade` régulier pour corriger les vulnérabilités.

---

*Fin du guide d'installation. Maintenant, vous comprenez à la fois le comment **et** le pourquoi de chaque composant.*

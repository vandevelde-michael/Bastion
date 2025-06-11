# 🛡️ Bastion Guacamole – Guide Simplifié

## ⚙️ 1. Prérequis

* **VM Bastion** (Debian 12 CLI) avec sudo
* **Nom de domaine/IP fixe** pointant sur la VM
* **Ports ouverts** : 80, 443, 4822 (guacd), 8080 (Tomcat)
* Internet pour télécharger les paquets

## 📥 2. Installer guacd

```bash
sudo apt update
sudo apt install -y guacd
sudo systemctl enable --now guacd
```

*Pourquoi ?* guacd fait le pont entre RDP/VNC/SSH et le navigateur.

## 🚀 3. Tomcat + Guacamole Web App

1. **Tomcat9** :

   ```bash
   sudo apt install -y tomcat9
   ```
2. **Guacamole WAR** :

   ```bash
   wget https://apache.org/dist/guacamole/1.5.0/binary/guacamole-1.5.0.war -O guacamole.war
   sudo mv guacamole.war /var/lib/tomcat9/webapps/
   sudo mkdir /etc/guacamole && sudo chown tomcat: /etc/guacamole
   sudo ln -s /etc/guacamole /usr/share/tomcat9/.guacamole
   sudo systemctl restart tomcat9
   ```

*Pourquoi ?* Tomcat héberge l’application Java Guacamole.

## 🗄️ 4. Base de données MariaDB

```bash
sudo apt install -y mariadb-server
sudo mysql_secure_installation
```

```sql
CREATE DATABASE guacamole_db;
CREATE USER 'guacuser'@'localhost' IDENTIFIED BY 'ChangeMe!';
GRANT ALL ON guacamole_db.* TO 'guacuser'@'localhost';
```

```bash
wget https://apache.org/dist/guacamole/1.5.0/binary/guacamole-auth-jdbc-1.5.0.tar.gz
tar xzf guacamole-auth-jdbc-1.5.0.tar.gz
cat guacamole-auth-jdbc-1.5.0/mysql/schema/*.sql | mysql -u root -p guacamole_db
```

*Pourquoi ?* Stockage persistant des utilisateurs et configs.

## 🔧 5. Configurer Guacamole

Créer `/etc/guacamole/guacamole.properties` :

```
guacd-hostname: localhost
guacd-port: 4822
mysql-hostname: localhost
mysql-database: guacamole_db
mysql-username: guacuser
mysql-password: ChangeMe!
```

```bash
sudo systemctl restart tomcat9 guacd
```

## 🔒 6. HTTPS avec Nginx

1. **Installer** :

   ```bash
   sudo apt install -y nginx
   ```
2. **Config** `/etc/nginx/sites-available/guacamole` :

   * rediriger 80→443
   * proxy\_pass → `http://127.0.0.1:8080/guacamole/`
3. **Activer** :

   ```bash
   sudo ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
   sudo nginx -t && sudo systemctl reload nginx
   ```

*Pourquoi ?* TLS pour sécuriser l’accès.

## 💻 7. Connexions & utilisateurs

* Ouvrir `https://bastion.example.com`
* Login : `guacadmin` / `guacadmin`, puis changer le mot de passe
* **Settings > Connections** : ajouter SSH, VNC, RDP
* **Users & Groups** : créer groupes (Admin, Linux, Windows)

## ✅ 8. Tests & Fin

* Tester chaque protocole dans le navigateur
* Vérifier logs (`guacd`, `tomcat9`, `nginx`)
* (Optionnel) activer **MFA**, **Firewall**, **MAJ régulières**

---

*Guide simplifié avec icônes pour un aperçu rapide.*

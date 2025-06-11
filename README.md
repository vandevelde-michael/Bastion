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

## 📂 4. Configuration via guacamole.xml

Guacamole peut fonctionner sans base SQL, en utilisant un fichier XML pour définir utilisateurs et connexions.

1. Créer le dossier de config si nécessaire :

   ```bash
   sudo mkdir -p /etc/guacamole
   ```
2. Éditer `/etc/guacamole/guacamole.xml` :

   ```xml
   <user-mapping>
     <!-- Admin -->
     <authorize username="admin" password="adminpass">
       <connection name="Debian-CLI (SSH)">
         <protocol>ssh</protocol>
         <param name="hostname">192.168.1.10</param>
         <param name="port">22</param>
         <param name="username">usercli</param>
       </connection>
       <connection name="Linux-GUI (VNC)">
         <protocol>vnc</protocol>
         <param name="hostname">192.168.1.11</param>
         <param name="port">5901</param>
       </connection>
       <connection name="Windows-10 (RDP)">
         <protocol>rdp</protocol>
         <param name="hostname">192.168.1.20</param>
         <param name="port">3389</param>
       </connection>
     </authorize>
     <!-- Autres utilisateurs -->
   </user-mapping>
   ```

*Pourquoi ?* Le XML est simple, sans dépendance externe, parfait pour de petits déploiements.

## 🔧 5. Configurer Guacamole

Modifier `/etc/guacamole/guacamole.properties` pour passer en mode XML :

```
guacd-hostname: localhost
guacd-port: 4822
guacamole-home: /etc/guacamole
user-mapping: /etc/guacamole/guacamole.xml
```

Redémarrer les services :

```bash
sudo systemctl restart guacd tomcat9
```

*Pourquoi ?* Ces paramètres indiquent à Guacamole d’utiliser le fichier XML plutôt que la base de données.

## 🔒 6. HTTPS avec Nginx Configurer Guacamole

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

---

**Projet réalisé par :** Quentin, Etienne, Morane et Michael

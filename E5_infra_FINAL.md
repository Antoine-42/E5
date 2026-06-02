# 🖥️ E5 — Infrastructure Serveur Linux — Guide Final

---

## 🗺️ Mon rôle dans l'architecture

```
[Capteurs] → [M5Stack] → [MQTT] → [Serveur Linux] → [MySQL] → [PHP] → [Navigateur]
                                         ↑
                                     MA PARTIE
```

Je prépare le serveur pour que tout le reste fonctionne :
- Apache2 (héberger les sites)
- MySQL (stocker les données)
- Mosquitto (broker MQTT)
- UFW (sécuriser les accès)
- SSH (accès distant)

---

## ÉTAPE 1 — 📍 S'orienter sur la VM

```bash
ip a                        # noter ton IP
whoami                      # vérifier ton user
sudo -l                     # vérifier tes droits sudo
systemctl status apache2    # déjà installé ?
systemctl status mysql      # déjà installé ?
systemctl status ssh        # déjà installé ?
systemctl status mosquitto  # déjà installé ?
```

> Ne pas réinstaller ce qui est déjà en place.

---

## ÉTAPE 2 — 📦 Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

> ⏳ Attendre que ça finisse, ça peut prendre 1-2 minutes.

---

## ÉTAPE 3 — 📦 Installer Apache

```bash
sudo apt install apache2 -y
```

Vérifier :

```bash
sudo systemctl status apache2
```

> ✅ Tu dois voir **active (running)** en vert.

---

## ÉTAPE 4 — 📦 Installer PHP

```bash
sudo apt install php libapache2-mod-php php-mysql -y
```

---

## ÉTAPE 5 — 📦 Installer MySQL

```bash
sudo apt install mysql-server -y
```

---

## ÉTAPE 6 — 📦 Installer Mosquitto

```bash
sudo apt install mosquitto mosquitto-clients -y
```

---

## ÉTAPE 7 — ✅ Démarrer et activer tous les services

```bash
sudo systemctl start apache2
sudo systemctl start mysql
sudo systemctl start ssh
sudo systemctl start mosquitto
sudo systemctl enable apache2
sudo systemctl enable mysql
sudo systemctl enable ssh
sudo systemctl enable mosquitto
```

Vérifier les quatre :

```bash
systemctl status apache2
systemctl status mysql
systemctl status ssh
systemctl status mosquitto
```

> Les quatre doivent afficher **active (running)**.

---

## ÉTAPE 8 — 🔒 Sécuriser MySQL

```bash
sudo mysql_secure_installation
```

Répondre exactement comme ça :

```
Would you like to setup VALIDATE PASSWORD component? → y
Please enter 0 = LOW, 1 = MEDIUM, 2 = STRONG: → 0
New password: → Root1234   (note ce mot de passe !)
Re-enter new password: → Root1234
Do you wish to continue with the password provided? → y
Remove anonymous users? → y
Disallow root login remotely? → y
Remove test database and access to it? → y
Reload privilege tables now? → y
```

> ⚠️ **NOTE TON MOT DE PASSE ROOT MYSQL** tu en auras besoin tout le temps !

---

## ÉTAPE 9 — 🗄️ Configurer MySQL

### Se connecter en root :

```bash
sudo mysql -u root -p
# Taper ton mot de passe root puis ENTRÉE
```

### Créer la base, les tables et les utilisateurs :

```sql
-- Créer la base de données
CREATE DATABASE salle_serveur;

-- Utiliser la base
USE salle_serveur;

-- Créer la table des mesures capteurs
CREATE TABLE mesures (
    id INT AUTO_INCREMENT PRIMARY KEY,
    capteur VARCHAR(50),
    valeur FLOAT,
    date_heure DATETIME DEFAULT NOW()
);

-- Créer la table des utilisateurs dashboard
CREATE TABLE utilisateurs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    login VARCHAR(50),
    password VARCHAR(255),
    role VARCHAR(20)
);

-- Vérifier que les tables existent
SHOW TABLES;

-- Créer l'user local (pour PHP et backend C++ sur la VM)
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'motdepasse';
GRANT ALL PRIVILEGES ON salle_serveur.* TO 'admin'@'localhost';

-- Créer l'user réseau (pour dBeaver depuis un autre poste)
CREATE USER 'admin'@'%' IDENTIFIED BY 'motdepasse';
GRANT ALL PRIVILEGES ON salle_serveur.* TO 'admin'@'%';

FLUSH PRIVILEGES;
EXIT;
```

> ✅ `SHOW TABLES` doit afficher **mesures** et **utilisateurs**
> Le `'localhost'` = connexions depuis la VM uniquement
> Le `'%'` = connexions depuis n'importe quelle IP (dBeaver)

### Si un script SQL est fourni à la place :

```bash
sudo mysql -u root -p < script.sql
```

### Vérifier la connexion avec l'user créé :

```bash
mysql -u admin -p salle_serveur
```

```sql
SHOW TABLES;
DESCRIBE mesures;
DESCRIBE utilisateurs;
EXIT;
```

---

## ÉTAPE 10 — 🔑 Réinitialiser le mot de passe MySQL (si oublié)

```bash
# Arrêter MySQL
sudo systemctl stop mysql

# Démarrer en mode sans authentification
sudo mysqld_safe --skip-grant-tables &

# Attendre 3-4 secondes puis se connecter
sudo mysql -u root
```

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NouveauMotDePasse';
FLUSH PRIVILEGES;
EXIT;
```

```bash
# Tuer le processus sans auth
sudo kill $(sudo cat /var/run/mysqld/mysqld.pid)

# Redémarrer MySQL normalement
sudo systemctl start mysql

# Tester
sudo mysql -u root -p
```

---

## ÉTAPE 11 — 📡 Configurer Mosquitto (Broker MQTT)

### Créer le fichier de configuration :

```bash
sudo nano /etc/mosquitto/conf.d/default.conf
```

Coller :

```
listener 1883
allow_anonymous false
password_file /etc/mosquitto/passwd
```

> - `listener 1883` → écoute sur le port MQTT standard
> - `allow_anonymous false` → connexion avec user/mdp obligatoire
> - `password_file` → fichier des users MQTT

Sauvegarder : **Ctrl+X** → **Y** → **Entrée**

### Créer les utilisateurs MQTT :

```bash
# Créer le fichier et ajouter le premier user (-c crée le fichier)
sudo mosquitto_passwd -c /etc/mosquitto/passwd mqttuser
# Il demande un mot de passe → taper : password
```

> Pour ajouter un 2ème user sans écraser le fichier (sans `-c`) :

```bash
sudo mosquitto_passwd /etc/mosquitto/passwd mqttuser2
```

### Redémarrer Mosquitto pour appliquer la config :

```bash
sudo systemctl restart mosquitto
systemctl status mosquitto
```

### Tester Mosquitto :

Ouvrir **deux terminaux** sur la VM.

**Terminal 1 — S'abonner :**
```bash
mosquitto_sub -h localhost -t "test/topic" -u mqttuser -P password
```

**Terminal 2 — Publier :**
```bash
mosquitto_pub -h localhost -t "test/topic" -m "hello" -u mqttuser -P password
```

> ✅ Si "hello" apparaît dans le terminal 1 → Mosquitto fonctionne !

---

## ÉTAPE 12 — 🌍 Configurer les VirtualHosts Apache

### Créer les dossiers des sites :

```bash
sudo mkdir -p /var/www/nuc.local
sudo mkdir -p /var/www/dashboard.local
sudo chown -R www-data:www-data /var/www/nuc.local
sudo chown -R www-data:www-data /var/www/dashboard.local
```

### Créer le VirtualHost nuc.local :

```bash
sudo nano /etc/apache2/sites-available/nuc.local.conf
```

```apache
<VirtualHost *:80>
    ServerName www.nuc.local
    DocumentRoot /var/www/nuc.local
    <Directory /var/www/nuc.local>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Sauvegarder : **Ctrl+X** → **Y** → **Entrée**

### Créer le VirtualHost dashboard.local :

```bash
sudo nano /etc/apache2/sites-available/dashboard.local.conf
```

```apache
<VirtualHost *:80>
    ServerName www.dashboard.local
    DocumentRoot /var/www/dashboard.local
    <Directory /var/www/dashboard.local>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Sauvegarder : **Ctrl+X** → **Y** → **Entrée**

### Activer les sites :

```bash
sudo a2ensite nuc.local.conf
sudo a2ensite dashboard.local.conf
sudo a2dissite 000-default.conf
sudo a2enmod rewrite
```

### Vérifier et redémarrer :

```bash
sudo apache2ctl configtest       # doit dire "Syntax OK"
sudo systemctl restart apache2
```

---

## ÉTAPE 13 — 🔥 UFW Firewall

> ⚠️ Si UFW est déjà actif, ouvrir les ports AVANT tout. Sinon tu perds l'accès SSH.

```bash
sudo ufw allow 22        # SSH — EN PREMIER OBLIGATOIRE
sudo ufw allow 80        # HTTP (Apache)
sudo ufw allow 443       # HTTPS
sudo ufw allow 1883      # MQTT (Mosquitto)
sudo ufw allow 3306      # MySQL (pour dBeaver depuis un autre poste)
sudo ufw enable          # activer le firewall
```

```
Command may disrupt existing ssh connections. Proceed with operation (y|n)? → y
```

```bash
sudo ufw reload
sudo ufw status verbose  # vérifier
```

---

## ÉTAPE 14 — 🌐 Configuration IP statique (si demandé)

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

> Remplacer l'IP par celle donnée dans le sujet.

```bash
sudo netplan apply
ip a    # vérifier la nouvelle IP
```

---

## ÉTAPE 15 — 🌐 /etc/hosts sur la VM

```bash
sudo nano /etc/hosts
```

Ajouter à la fin :

```
127.0.0.1   www.nuc.local
127.0.0.1   www.dashboard.local
```

Tester :

```bash
curl http://www.nuc.local
curl http://www.dashboard.local
```

> ✅ Si tu vois du HTML c'est bon !

---

## ÉTAPE 16 — 🌐 /etc/hosts sur TA machine (pour le navigateur)

**Mac/Linux :**

```bash
sudo nano /etc/hosts
```

**Windows** (Notepad en admin) :

```
C:\Windows\System32\drivers\etc\hosts
```

Ajouter :

```
IP_DE_LA_VM   www.nuc.local
IP_DE_LA_VM   www.dashboard.local
```

Vider le cache DNS :

```bash
# Mac
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

# Linux
sudo systemd-resolve --flush-caches

# Windows
ipconfig /flushdns
```

---

## ÉTAPE 17 — 👤 Gestion des utilisateurs Linux (si demandé)

```bash
# Créer un utilisateur
sudo adduser nomutilisateur
# Répondre aux questions, taper un mot de passe, ENTRÉE pour le reste

# Ajouter au groupe sudo si besoin
sudo usermod -aG sudo nomutilisateur
```

---

## ÉTAPE 18 — 🖥️ dBeaver — Vérifier MySQL depuis un poste

dBeaver est un logiciel graphique sur le **poste de travail** pour visualiser MySQL sans ligne de commande.

### Prérequis côté VM :
- UFW autorise le port 3306 ✅ (fait étape 13)
- User MySQL avec `'%'` existe ✅ (fait étape 9)

### Se connecter :

1. Ouvrir dBeaver → **New Database Connection**
2. Choisir **MySQL** → Next
3. Remplir :

```
Host     : IP_DE_LA_VM
Port     : 3306
Database : salle_serveur
Username : admin
Password : motdepasse
```

4. Cliquer **Test Connection** → doit dire **Connected** ✅
5. **Finish**

### Ce que tu vois :

```
salle_serveur
  └── Tables
        ├── mesures      → double-clic → onglet "Data"
        └── utilisateurs → double-clic → onglet "Data"
```

> Utile pour montrer aux jurés que la base est bien configurée.

---

## 🧪 TESTS FINAUX

### Services :

```bash
systemctl status apache2
systemctl status mysql
systemctl status ssh
systemctl status mosquitto
```

### VirtualHosts :

```bash
sudo apache2ctl -S
sudo apache2ctl configtest
```

### Sites web :

```bash
curl http://www.nuc.local
curl http://www.dashboard.local
```

### MySQL :

```bash
mysql -u admin -p salle_serveur
```

```sql
SHOW TABLES;
DESCRIBE mesures;
DESCRIBE utilisateurs;
EXIT;
```

### Mosquitto :

```bash
# Terminal 1
mosquitto_sub -h localhost -t "test/topic" -u mqttuser -P password

# Terminal 2
mosquitto_pub -h localhost -t "test/topic" -m "hello" -u mqttuser -P password
```

### Firewall :

```bash
sudo ufw status verbose
```

### Ports ouverts :

```bash
ss -tlnp
# doit voir :22 :80 :1883 :3306
```

---

## 🗺️ Ordre résumé

```
1.  S'orienter (ip a, whoami)
2.  apt update && upgrade
3.  Installer Apache
4.  Installer PHP
5.  Installer MySQL
6.  Installer Mosquitto
7.  Démarrer les 4 services
8.  Sécuriser MySQL (mysql_secure_installation)
9.  MySQL (base + tables + users localhost + %)
10. Mosquitto (config + users MQTT + test)
11. VirtualHosts Apache (nuc + dashboard)
12. UFW (22 EN PREMIER, puis 80, 443, 1883, 3306)
13. IP statique (si demandé)
14. /etc/hosts VM
15. /etc/hosts ta machine
16. dBeaver (vérifier connexion)
17. Tout tester
```

---

## 📋 Tableau récap des tests

| Quoi | Commande | Résultat attendu |
|---|---|---|
| Apache tourne | `systemctl status apache2` | active (running) |
| MySQL tourne | `systemctl status mysql` | active (running) |
| SSH tourne | `systemctl status ssh` | active (running) |
| Mosquitto tourne | `systemctl status mosquitto` | active (running) |
| Config Apache | `sudo apache2ctl configtest` | Syntax OK |
| VirtualHosts actifs | `sudo apache2ctl -S` | nuc.local + dashboard.local |
| Site nuc | `curl http://www.nuc.local` | réponse HTTP |
| Site dashboard | `curl http://www.dashboard.local` | réponse HTTP |
| MySQL tables | `SHOW TABLES;` | mesures + utilisateurs |
| MySQL user | `mysql -u admin -p salle_serveur` | connecté sans erreur |
| MySQL dBeaver | Test Connection | Connected |
| MQTT test | pub/sub sur test/topic | message reçu |
| Firewall | `sudo ufw status verbose` | 22/80/443/1883/3306 ALLOW |
| Ports | `ss -tlnp` | :22 :80 :1883 :3306 visibles |

---

## 🚨 Problèmes fréquents et solutions

| Problème | Cause | Solution |
|---|---|---|
| Site inaccessible | Apache arrêté | `sudo systemctl start apache2` |
| Port 80 bloqué | UFW | `sudo ufw allow 80` |
| `Access denied` MySQL | Mauvais user/droits | Vérifier `GRANT` + `FLUSH PRIVILEGES` |
| Mot de passe MySQL oublié | — | Voir étape 10 |
| dBeaver ne connecte pas | Port 3306 bloqué | `sudo ufw allow 3306` |
| dBeaver `Access denied` | User `%` manquant | `CREATE USER 'admin'@'%'` |
| Mosquitto refuse connexion | Config mal faite | Vérifier `/etc/mosquitto/conf.d/default.conf` |
| MQTT pub/sub échoue | Mauvais user/mdp | Vérifier `mosquitto_passwd` |
| Nom domaine marche pas | /etc/hosts pas à jour | Ajouter IP dans /etc/hosts |
| `Syntax OK` mais site répond pas | Site pas activé | `a2ensite` + `systemctl restart apache2` |

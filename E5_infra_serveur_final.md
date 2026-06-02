# 🖥️ E5 — Ma partie : Infrastructure Serveur

---

## 🗺️ Mon rôle dans l'architecture

```
[Capteurs] → [M5Stack] → [MQTT] → [Serveur Linux] → [MySQL] → [PHP] → [Navigateur]
                                         ↑
                                     MA PARTIE
```

Je prépare le serveur Linux pour que tout le reste fonctionne :
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

## ÉTAPE 2 — 📦 Installation LAMP + SSH + Mosquitto

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php mysql-server php-mysql openssh-server mosquitto mosquitto-clients -y
```

---

## ÉTAPE 3 — ✅ Démarrer et activer les services

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

Vérifier :

```bash
systemctl status apache2
systemctl status mysql
systemctl status ssh
systemctl status mosquitto
```

> Les quatre doivent afficher **active (running)**.

---

## ÉTAPE 4 — 🔥 UFW Firewall

> ⚠️ Toujours ouvrir le port 22 EN PREMIER avant d'activer UFW.
> Sinon tu perds l'accès SSH à la VM.

```bash
sudo ufw allow 22        # SSH — EN PREMIER OBLIGATOIRE
sudo ufw allow 80        # HTTP (Apache)
sudo ufw allow 1883      # MQTT (Mosquitto)
sudo ufw allow 3306      # MySQL (pour dBeaver depuis un autre poste)
sudo ufw enable          # activer le firewall
sudo ufw status verbose  # vérifier
```

---

## ÉTAPE 5 — 🗄️ Configurer MySQL

### Se connecter en root :

```bash
sudo mysql
```

### Créer la base de données :

```sql
CREATE DATABASE iot_db;
```

### Créer les tables :

```sql
USE iot_db;

-- Table des mesures capteurs
CREATE TABLE mesures (
    id INT AUTO_INCREMENT PRIMARY KEY,
    capteur VARCHAR(50),
    type_mesure VARCHAR(50),
    valeur FLOAT,
    unite VARCHAR(10),
    date_mesure DATETIME DEFAULT NOW()
);

-- Table des utilisateurs (pour authentification dashboard)
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(255) NOT NULL
);
```

> Si le sujet fournit un script SQL à la place :
> ```bash
> sudo mysql < script.sql
> ```

### Créer les utilisateurs MySQL :

```sql
-- User local (pour PHP et backend C++ sur la VM)
CREATE USER 'iotuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON iot_db.* TO 'iotuser'@'localhost';

-- User réseau (pour dBeaver depuis un autre poste)
CREATE USER 'iotuser'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON iot_db.* TO 'iotuser'@'%';

FLUSH PRIVILEGES;
EXIT;
```

> Le `'localhost'` = connexions depuis la VM uniquement (PHP, backend C++)
> Le `'%'` = connexions depuis n'importe quelle IP (dBeaver, autre machine)

### Vérifier que tout est bien créé :

```bash
mysql -u iotuser -p iot_db
```

```sql
SHOW TABLES;       -- doit afficher mesures et users
DESCRIBE mesures;  -- doit afficher la structure de la table
DESCRIBE users;    -- doit afficher la structure de la table
EXIT;
```

---

## ÉTAPE 6 — 📡 Configurer Mosquitto (Broker MQTT)

### Créer un fichier de configuration :

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
> - `allow_anonymous false` → oblige à s'authentifier
> - `password_file` → fichier des users MQTT

### Créer les utilisateurs MQTT :

```bash
# Créer le fichier de mots de passe et ajouter un user
sudo mosquitto_passwd -c /etc/mosquitto/passwd mqttuser
# Il va demander un mot de passe → taper : password
```

> Le `-c` crée le fichier. Pour ajouter un 2ème user sans écraser le fichier :

```bash
sudo mosquitto_passwd /etc/mosquitto/passwd mqttuser2
```

### Redémarrer Mosquitto pour appliquer :

```bash
sudo systemctl restart mosquitto
```

### Vérifier que Mosquitto tourne :

```bash
systemctl status mosquitto
```

### Tester Mosquitto :

Ouvrir deux terminaux sur la VM.

**Terminal 1 — S'abonner à un topic :**
```bash
mosquitto_sub -h localhost -t "test/topic" -u mqttuser -P password
```

**Terminal 2 — Publier un message :**
```bash
mosquitto_pub -h localhost -t "test/topic" -m "hello" -u mqttuser -P password
```

> Si "hello" apparaît dans le terminal 1 → ✅ Mosquitto fonctionne

---

## ÉTAPE 7 — 🌍 Configurer les VirtualHosts Apache

### Créer les dossiers des sites :

```bash
sudo mkdir -p /var/www/nuc
sudo mkdir -p /var/www/dashboard
sudo chown -R www-data:www-data /var/www/nuc
sudo chown -R www-data:www-data /var/www/dashboard
```

### Créer le VirtualHost nuc.local :

```bash
sudo nano /etc/apache2/sites-available/nuc.local.conf
```

```apache
<VirtualHost *:80>
    ServerName www.nuc.local
    DocumentRoot /var/www/nuc
    <Directory /var/www/nuc>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### Créer le VirtualHost dashboard.local :

```bash
sudo nano /etc/apache2/sites-available/dashboard.local.conf
```

```apache
<VirtualHost *:80>
    ServerName www.dashboard.local
    DocumentRoot /var/www/dashboard
    <Directory /var/www/dashboard>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### Activer les sites + désactiver le site par défaut :

```bash
sudo a2ensite nuc.local.conf
sudo a2ensite dashboard.local.conf
sudo a2dissite 000-default.conf
```

### Vérifier et recharger :

```bash
sudo apache2ctl configtest       # doit dire "Syntax OK"
sudo systemctl reload apache2
```

---

## ÉTAPE 8 — 🌐 /etc/hosts sur la VM

```bash
sudo nano /etc/hosts
```

Ajouter à la fin (remplacer avec ton IP vue à l'étape 1) :

```
192.168.x.x   www.nuc.local
192.168.x.x   www.dashboard.local
```

---

## ÉTAPE 9 — 🌐 /etc/hosts sur TA machine

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
192.168.x.x   www.nuc.local
192.168.x.x   www.dashboard.local
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

## ÉTAPE 10 — 🖥️ dBeaver — Vérifier MySQL depuis un poste

dBeaver est un logiciel graphique sur le **poste de travail** (pas la VM) pour visualiser MySQL sans ligne de commande.

### Prérequis côté VM :

- UFW autorise le port 3306 (fait à l'étape 4)
- User MySQL avec `'%'` existe (fait à l'étape 5)

### Se connecter dans dBeaver :

1. Ouvrir dBeaver → **New Database Connection**
2. Choisir **MySQL** → Next
3. Remplir :

```
Host     : IP de la VM (ex: 192.168.1.114)
Port     : 3306
Database : iot_db
Username : iotuser
Password : password
```

4. Cliquer **Test Connection** → doit dire **Connected** ✅
5. **Finish**

### Ce que tu vois dans dBeaver :

```
iot_db
  └── Tables
        ├── mesures    → double-clic → onglet "Data" pour voir les données
        └── users      → double-clic → onglet "Data" pour voir les users
```

> Utile pour montrer aux jurés que la base est bien configurée.

---

## ÉTAPE 11 — 🧪 Tout tester

### Services :

```bash
systemctl status apache2
systemctl status mysql
systemctl status ssh
systemctl status mosquitto
```

### VirtualHosts :

```bash
sudo apache2ctl -S          # liste les VirtualHosts actifs
sudo apache2ctl configtest  # doit dire "Syntax OK"
```

### Sites web :

```bash
curl http://www.nuc.local
curl http://www.dashboard.local
```

### MySQL :

```bash
mysql -u iotuser -p iot_db
```

```sql
SHOW TABLES;
DESCRIBE mesures;
DESCRIBE users;
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
# doit voir :22 (ssh), :80 (apache), :1883 (mosquitto), :3306 (mysql)
```

### dBeaver :

```
Test Connection → Connected ✅
Tables mesures et users visibles ✅
```

---

## 🗺️ Ordre résumé

```
1.  S'orienter (ip a, whoami)
2.  apt install LAMP + SSH + Mosquitto
3.  Démarrer les 4 services
4.  UFW (22 EN PREMIER, puis 80, 1883, 3306)
5.  MySQL (base + tables + users local + %)
6.  Mosquitto (config + users MQTT)
7.  VirtualHosts Apache (nuc + dashboard)
8.  /etc/hosts VM
9.  /etc/hosts ta machine
10. dBeaver (vérifier connexion)
11. Tout tester
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
| MySQL tables | `SHOW TABLES;` | mesures + users |
| MySQL user local | `mysql -u iotuser -p iot_db` | connecté |
| MySQL dBeaver | Test Connection | Connected |
| MQTT test | pub/sub sur test/topic | message reçu |
| Firewall | `sudo ufw status verbose` | 22/80/1883/3306 ALLOW |
| Ports | `ss -tlnp` | :22 :80 :1883 :3306 |

---

## 🚨 Problèmes fréquents et solutions

| Problème | Cause | Solution |
|---|---|---|
| Site inaccessible | Apache arrêté | `sudo systemctl start apache2` |
| Port 80 bloqué | UFW | `sudo ufw allow 80` |
| `Access denied` MySQL | Mauvais user/droits | Vérifier `GRANT` + `FLUSH PRIVILEGES` |
| dBeaver ne connecte pas | Port 3306 bloqué | `sudo ufw allow 3306` |
| dBeaver `Access denied` | User `%` manquant | `CREATE USER 'iotuser'@'%'` |
| Mosquitto refuse connexion | auth mal configurée | Vérifier `/etc/mosquitto/conf.d/default.conf` |
| MQTT pub/sub échoue | Mauvais user/mdp | Vérifier `mosquitto_passwd` |
| Nom domaine marche pas | /etc/hosts pas à jour | Ajouter IP dans /etc/hosts |
| `Syntax OK` mais site répond pas | Site pas activé | `a2ensite` + `systemctl reload apache2` |

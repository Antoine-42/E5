# 🖥️ E5 — Infrastructure Serveur — Guide complet A à Z

---

## 🗺️ Rappel architecture globale

```
[Capteurs] → [M5Stack] → [MQTT] → [Serveur Linux] → [MySQL] → [PHP/Apache] → [Navigateur]
```

Ta partie = tout ce qui est dans le **Serveur Linux** :
- Apache2 + PHP (les deux sites)
- MySQL (base de données)
- UFW (firewall)
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
```

> Ne pas réinstaller ce qui est déjà en place.

---

## ÉTAPE 2 — 📦 Installation LAMP + SSH

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php mysql-server php-mysql openssh-server -y
```

---

## ÉTAPE 3 — ✅ Démarrer et activer les services

```bash
sudo systemctl start apache2
sudo systemctl start mysql
sudo systemctl start ssh
sudo systemctl enable apache2
sudo systemctl enable mysql
sudo systemctl enable ssh
```

Vérifier :

```bash
systemctl status apache2
systemctl status mysql
systemctl status ssh
```

> Les trois doivent afficher **active (running)**.

---

## ÉTAPE 4 — 🔥 UFW Firewall

> ⚠️ Toujours faire AVANT d'activer UFW sinon tu perds l'accès SSH.

```bash
sudo ufw allow 22        # SSH — EN PREMIER OBLIGATOIRE
sudo ufw allow 80        # HTTP
sudo ufw allow 1883      # MQTT
sudo ufw allow 3306      # MySQL (pour dBeaver)
sudo ufw enable          # activer le firewall
sudo ufw status verbose  # vérifier
```

---

## ÉTAPE 5 — 🗄️ Configurer MySQL

### Se connecter :

```bash
sudo mysql
```

### Créer la base, les users et la table :

```sql
CREATE DATABASE iot_db;

-- User local (pour PHP et backend C++)
CREATE USER 'iotuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON iot_db.* TO 'iotuser'@'localhost';

-- User réseau (pour dBeaver depuis un autre poste)
CREATE USER 'iotuser'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON iot_db.* TO 'iotuser'@'%';

FLUSH PRIVILEGES;

USE iot_db;

-- Créer la table mesures
CREATE TABLE mesures (
    id INT AUTO_INCREMENT PRIMARY KEY,
    capteur VARCHAR(50),
    type_mesure VARCHAR(50),
    valeur FLOAT,
    unite VARCHAR(10),
    date_mesure DATETIME DEFAULT NOW()
);

EXIT;
```

### Si un script SQL est fourni à la place :

```bash
sudo mysql < script.sql
```

---

## ÉTAPE 6 — 📥 Insérer des données fictives

> Selon le synoptique attribué, utilise le bloc correspondant.

### ▶️ Synoptique 1 — RFID + Relai

```bash
sudo mysql
```

```sql
USE iot_db;

INSERT INTO mesures (capteur, type_mesure, valeur, unite) VALUES
('RFID', 'acces', 1, ''),
('RFID', 'acces', 0, ''),
('RFID', 'acces', 1, ''),
('RELAI', 'etat', 1, ''),
('RELAI', 'etat', 0, ''),
('RELAI', 'etat', 1, '');

EXIT;
```

### ▶️ Synoptique 2 — ENV4 + SCD040

```bash
sudo mysql
```

```sql
USE iot_db;

INSERT INTO mesures (capteur, type_mesure, valeur, unite) VALUES
('ENV4', 'temperature', 22.5, '°C'),
('ENV4', 'humidite', 45.2, '%'),
('ENV4', 'pression', 1013.25, 'hPa'),
('ENV4', 'temperature', 23.1, '°C'),
('ENV4', 'humidite', 46.8, '%'),
('ENV4', 'pression', 1012.80, 'hPa'),
('SCD040', 'temperature', 21.8, '°C'),
('SCD040', 'humidite', 48.3, '%'),
('SCD040', 'co2', 412.0, 'ppm'),
('SCD040', 'temperature', 22.0, '°C'),
('SCD040', 'humidite', 47.1, '%'),
('SCD040', 'co2', 430.5, 'ppm');

EXIT;
```

### ▶️ Synoptique 3 — ENV4 + Radar LD2410

```bash
sudo mysql
```

```sql
USE iot_db;

INSERT INTO mesures (capteur, type_mesure, valeur, unite) VALUES
('ENV4', 'temperature', 22.5, '°C'),
('ENV4', 'humidite', 45.2, '%'),
('ENV4', 'pression', 1013.25, 'hPa'),
('ENV4', 'temperature', 23.1, '°C'),
('ENV4', 'humidite', 46.8, '%'),
('ENV4', 'pression', 1012.80, 'hPa'),
('LD2410', 'presence', 1, ''),
('LD2410', 'presence', 0, ''),
('LD2410', 'presence', 1, '');

EXIT;
```

### Vérifier les données :

```bash
mysql -u iotuser -p iot_db
```

```sql
SELECT * FROM mesures ORDER BY date_mesure DESC;
EXIT;
```

---

## ÉTAPE 7 — 🌍 VirtualHosts Apache

### Créer les dossiers :

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

## ÉTAPE 8 — 📝 Créer le site statique nuc.local

```bash
sudo nano /var/www/nuc/index.php
```

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Supervision IoT</title>
    <style>
        body { font-family: Arial; background: #f0f0f0; padding: 40px; text-align: center; }
        h1 { color: #3a3a8c; }
        a { display: inline-block; margin-top: 20px; padding: 10px 20px;
            background: #3a3a8c; color: white; text-decoration: none; border-radius: 5px; }
    </style>
</head>
<body>
    <h1>🖥️ Système de supervision IoT — Salle Serveur</h1>
    <p>Architecture : Capteurs → M5Stack → MQTT → Serveur Linux → MySQL → Dashboard PHP</p>
    <a href="http://www.dashboard.local">Accéder au Dashboard</a>
</body>
</html>
```

Sauvegarder : **Ctrl+X** → **Y** → **Entrée**

---

## ÉTAPE 9 — 📊 Créer le Dashboard PHP

```bash
sudo nano /var/www/dashboard/index.php
```

```php
<?php
// Connexion à MySQL via PDO
$pdo = new PDO('mysql:host=localhost;dbname=iot_db', 'iotuser', 'password');
?>

<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <!-- Rafraîchissement automatique toutes les 5 secondes = temps réel -->
    <meta http-equiv="refresh" content="5">
    <title>Dashboard IoT</title>
    <style>
        body { font-family: Arial; background: #f0f0f0; padding: 20px; }
        h1 { color: #333; }
        table { width: 100%; border-collapse: collapse; background: white; }
        th { background: #3a3a8c; color: white; padding: 10px; }
        td { padding: 10px; border-bottom: 1px solid #ddd; text-align: center; }
        tr:hover { background: #f5f5f5; }
    </style>
</head>
<body>

<h1>📊 Dashboard IoT — Salle Serveur</h1>

<table>
    <tr>
        <th>ID</th>
        <th>Capteur</th>
        <th>Type</th>
        <th>Valeur</th>
        <th>Unité</th>
        <th>Date</th>
    </tr>

    <?php
    $stmt = $pdo->query("SELECT * FROM mesures ORDER BY date_mesure DESC");
    while($row = $stmt->fetch()):
    ?>
    <tr>
        <td><?= $row['id'] ?></td>
        <td><?= $row['capteur'] ?></td>
        <td><?= $row['type_mesure'] ?></td>
        <td><?= $row['valeur'] ?></td>
        <td><?= $row['unite'] ?></td>
        <td><?= $row['date_mesure'] ?></td>
    </tr>
    <?php endwhile; ?>

</table>

</body>
</html>
```

Sauvegarder : **Ctrl+X** → **Y** → **Entrée**

---

## ÉTAPE 10 — 🌐 /etc/hosts sur la VM

```bash
sudo nano /etc/hosts
```

Ajouter à la fin :

```
IP_DE_LA_VM   www.nuc.local
IP_DE_LA_VM   www.dashboard.local
```

> Remplacer `IP_DE_LA_VM` par l'IP vue avec `ip a` à l'étape 1.

---

## ÉTAPE 11 — 🌐 /etc/hosts sur TA machine

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

## ÉTAPE 12 — 🖥️ dBeaver — Connexion graphique à MySQL

> dBeaver est installé sur le poste de travail, pas sur la VM.

1. Ouvrir dBeaver → **New Database Connection**
2. Choisir **MySQL** → Next
3. Remplir :

```
Host     : IP_DE_LA_VM
Port     : 3306
Database : iot_db
Username : iotuser
Password : password
```

4. Cliquer **Test Connection** → doit dire **Connected** ✅
5. **Finish**

Double-cliquer sur `mesures` → onglet **Data** pour voir les données.

---

## ÉTAPE 13 — 🧪 Tout tester

### Services :

```bash
systemctl status apache2
systemctl status mysql
systemctl status ssh
```

### VirtualHosts :

```bash
sudo apache2ctl -S
sudo apache2ctl configtest
```

### Sites web depuis la VM :

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
SELECT * FROM mesures ORDER BY date_mesure DESC LIMIT 5;
EXIT;
```

### Firewall :

```bash
sudo ufw status verbose
```

### Ports :

```bash
ss -tlnp
```

### Navigateur :

```
http://www.nuc.local
http://www.dashboard.local
```

### Tester le temps réel :

```bash
sudo mysql -u root iot_db -e "INSERT INTO mesures (capteur, type_mesure, valeur, unite) VALUES ('ENV4', 'temperature', 99.9, '°C');"
```

> Attendre 5 secondes → 99.9°C doit apparaître tout seul dans le dashboard ✅

Supprimer après :

```bash
sudo mysql -u root iot_db -e "DELETE FROM mesures WHERE valeur = 99.9;"
```

---

## 🗺️ Ordre résumé

```
1. S'orienter (ip a, whoami)
2. apt install LAMP + SSH
3. Démarrer les services
4. UFW (22 EN PREMIER, puis 80, 1883, 3306)
5. MySQL (base + users local + % + table mesures)
6. Données fictives selon le synoptique
7. VirtualHosts Apache (nuc + dashboard)
8. Site statique nuc.local
9. Dashboard PHP dashboard.local
10. /etc/hosts VM
11. /etc/hosts ta machine
12. dBeaver
13. Tout tester
```

---

## 📋 Tableau récap des tests

| Quoi | Commande / Action | Résultat attendu |
|---|---|---|
| Apache tourne | `systemctl status apache2` | active (running) |
| MySQL tourne | `systemctl status mysql` | active (running) |
| SSH tourne | `systemctl status ssh` | active (running) |
| Config Apache | `sudo apache2ctl configtest` | Syntax OK |
| VirtualHosts actifs | `sudo apache2ctl -S` | nuc.local + dashboard.local |
| Site nuc VM | `curl http://www.nuc.local` | HTML affiché |
| Site dashboard VM | `curl http://www.dashboard.local` | HTML avec données |
| Site navigateur | `http://www.nuc.local` | Page statique |
| Dashboard navigateur | `http://www.dashboard.local` | Tableau des mesures |
| MySQL connexion | `mysql -u iotuser -p iot_db` | Connecté sans erreur |
| Données présentes | `SELECT * FROM mesures;` | Lignes visibles |
| dBeaver | Test Connection | Connected |
| Firewall | `sudo ufw status verbose` | 22/80/1883/3306 ALLOW |
| Ports | `ss -tlnp` | :22 :80 :3306 visibles |
| Temps réel | Insérer ligne → attendre 5s | Apparaît dans le dashboard |

---

## 🚨 Problèmes fréquents et solutions

| Problème | Cause | Solution |
|---|---|---|
| Site inaccessible | Apache arrêté | `sudo systemctl start apache2` |
| Site inaccessible | UFW bloque port 80 | `sudo ufw allow 80` |
| Dashboard vide | PHP pas connecté MySQL | Vérifier PDO dans index.php |
| `Access denied` MySQL | Mauvais user/droits | Vérifier `GRANT` + `FLUSH PRIVILEGES` |
| Nom de domaine marche pas | /etc/hosts pas à jour | Ajouter l'IP dans /etc/hosts |
| dBeaver ne connecte pas | Port 3306 bloqué | `sudo ufw allow 3306` |
| dBeaver `Access denied` | User `%` manquant | `CREATE USER 'iotuser'@'%'` |
| `Syntax OK` mais site répond pas | Site pas activé | `a2ensite` + `systemctl reload apache2` |

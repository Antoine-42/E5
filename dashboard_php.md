# 📊 Affichage et Supervision — Dashboard PHP temps réel

---

## 🗺️ Rappel du processus complet

```
[Capteurs] → [M5Stack] → [MQTT] → [Serveur Linux] → [MySQL] → [PHP/Apache] → [Navigateur]
```

Le dashboard PHP est la **dernière étape** : il lit les données stockées dans MySQL et les affiche dans un navigateur web, avec un rafraîchissement automatique pour simuler le temps réel.

---

## 📋 Prérequis avant de commencer

Avant de créer le dashboard, vérifier que tout est en place :

```bash
systemctl status apache2       # Apache tourne
systemctl status mysql         # MySQL tourne
sudo apache2ctl -S             # VirtualHost dashboard.local actif
mysql -u iotuser -p iot_db -e "SELECT * FROM mesures LIMIT 5;"   # données présentes
```

---

## 1. 🗄️ Créer la base de données et la table

### Se connecter à MySQL :

```bash
sudo mysql
```

### Créer la base, la table et les utilisateurs :

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

---

## 2. 📥 Insérer des données fictives selon les capteurs

### ▶️ Synoptique 1 — Capteur RFID + Boitier Relai

```sql
sudo mysql -u root iot_db
```

```sql
INSERT INTO mesures (capteur, type_mesure, valeur, unite) VALUES
('RFID', 'acces', 1, ''),
('RFID', 'acces', 0, ''),
('RFID', 'acces', 1, ''),
('RELAI', 'etat', 1, ''),
('RELAI', 'etat', 0, ''),
('RELAI', 'etat', 1, '');
EXIT;
```

### ▶️ Synoptique 2 — Capteur ENV4 + SCD040

```sql
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

### ▶️ Synoptique 3 — Capteur ENV4 + Radar LD2410

```sql
INSERT INTO mesures (capteur, type_mesure, valeur, unite) VALUES
('ENV4', 'temperature', 22.5, '°C'),
('ENV4', 'humidite', 45.2, '%'),
('ENV4', 'pression', 1013.25, 'hPa'),
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

## 3. 🌍 Créer le dossier du dashboard

```bash
sudo mkdir -p /var/www/dashboard
sudo chown -R www-data:www-data /var/www/dashboard
```

---

## 4. 📝 Créer le fichier PHP/HTML du dashboard

```bash
sudo nano /var/www/dashboard/index.php
```

Coller ce code complet :

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
    // Récupérer toutes les mesures, les plus récentes en premier
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

## 5. ✅ Vérifier le VirtualHost dashboard.local

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

Activer et recharger :

```bash
sudo a2ensite dashboard.local.conf
sudo apache2ctl configtest        # doit dire "Syntax OK"
sudo systemctl reload apache2
```

---

## 6. 🌐 /etc/hosts — Résolution du nom de domaine local

### Sur la VM :

```bash
sudo nano /etc/hosts
```

Ajouter :

```
192.168.64.16   www.dashboard.local
```

### Sur TA machine (pour accéder depuis le navigateur) :

**Linux/Mac :**

```bash
sudo nano /etc/hosts
```

**Windows** (Notepad en admin) :

```
C:\Windows\System32\drivers\etc\hosts
```

Ajouter :

```
192.168.64.16   www.dashboard.local
```

---

## 7. 🧪 Tester

### Depuis la VM :

```bash
curl http://www.dashboard.local
# doit afficher du HTML avec les données
```

### Depuis le navigateur sur ta machine :

```
http://www.dashboard.local
```

Tu dois voir le tableau avec toutes les mesures, **les plus récentes en haut**.

---

## 8. ⏱️ Tester le temps réel

Pour vérifier que le rafraîchissement automatique fonctionne, insérer une donnée manuellement depuis la VM et regarder si elle apparaît dans le navigateur au bout de 5 secondes :

```bash
sudo mysql -u root iot_db -e "INSERT INTO mesures (capteur, type_mesure, valeur, unite) VALUES ('ENV4', 'temperature', 99.9, '°C');"
```

Si **99.9°C** apparaît tout seul dans le dashboard → ✅ temps réel confirmé

Supprimer la donnée de test après :

```bash
sudo mysql -u root iot_db -e "DELETE FROM mesures WHERE valeur = 99.9;"
```

---

## 9. 🖥️ Vérifier avec dBeaver

dBeaver permet de visualiser les données MySQL depuis un poste de travail sans ligne de commande.

### Prérequis côté VM :

```bash
sudo ufw allow 3306    # autoriser MySQL depuis l'extérieur
```

L'user MySQL doit avoir `'%'` (voir section 1).

### Connexion dans dBeaver :

```
Host     : 192.168.64.16
Port     : 3306
Database : iot_db
Username : iotuser
Password : password
```

Cliquer **Test Connection** → **Connected** ✅

Double-cliquer sur la table `mesures` → onglet **Data** → voir les données en tableau.

---

## 📋 Tableau récap — Ce que fait chaque ligne du code PHP

| Code | Rôle |
|---|---|
| `new PDO(...)` | Connexion à MySQL |
| `meta http-equiv="refresh" content="5"` | Rafraîchissement toutes les 5 secondes |
| `$pdo->query("SELECT * FROM mesures ...")` | Récupérer les données |
| `while($row = $stmt->fetch())` | Boucler sur chaque ligne |
| `<?= $row['valeur'] ?>` | Afficher la valeur dans le HTML |

---

## 📋 Tableau récap — Tests finaux

| Quoi | Commande / Action | Résultat attendu |
|---|---|---|
| Apache tourne | `systemctl status apache2` | active (running) |
| MySQL tourne | `systemctl status mysql` | active (running) |
| Données présentes | `SELECT * FROM mesures;` | lignes visibles |
| Site accessible VM | `curl http://www.dashboard.local` | HTML avec données |
| Site navigateur | `http://www.dashboard.local` | tableau affiché |
| Temps réel | Insérer une ligne → attendre 5s | apparaît tout seul |
| dBeaver | Test Connection | Connected |

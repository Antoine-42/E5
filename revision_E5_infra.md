# 🖥️ Révision E5 — Infrastructure Serveur (A à Z)

---

## 1. 📍 S'orienter

```bash
ip a                # noter ton IP (ex: 192.168.64.16)
whoami              # vérifier ton user
sudo -l             # vérifier tes droits sudo
```

---

## 2. 📦 Mise à jour + installation LAMP + SSH

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php mysql-server php-mysql openssh-server -y
```

---

## 3. ✅ Démarrer et activer les services

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

## 4. 🔥 UFW — Firewall (AVANT TOUT LE RESTE)

```bash
sudo ufw allow 22        # SSH en PREMIER obligatoire
sudo ufw allow 80        # HTTP
sudo ufw allow 1883      # MQTT
sudo ufw enable          # activer
sudo ufw status verbose  # vérifier
```

---

## 5. 🗄️ Configurer MySQL

```bash
sudo mysql
```

```sql
CREATE DATABASE iot_db;
CREATE USER 'iotuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON iot_db.* TO 'iotuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Si t'as un script SQL fourni :

```bash
sudo mysql < script.sql
```

Vérifier la connexion :

```bash
mysql -u iotuser -p iot_db
```

```sql
SHOW TABLES;
EXIT;
```

---

## 6. 🌍 VirtualHosts Apache

### Créer les dossiers :

```bash
sudo mkdir -p /var/www/nuc
sudo mkdir -p /var/www/dashboard
```

### Créer les fichiers de test :

```bash
echo "<?php echo 'NUC OK'; ?>" | sudo tee /var/www/nuc/index.php
echo "<?php echo 'DASHBOARD OK'; ?>" | sudo tee /var/www/dashboard/index.php
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

### Vérifier la config et recharger :

```bash
sudo apache2ctl configtest       # doit dire "Syntax OK"
sudo systemctl reload apache2
```

---

## 7. 🌐 /etc/hosts sur la VM

```bash
sudo nano /etc/hosts
```

Ajouter à la fin :

```
192.168.64.16   www.nuc.local
192.168.64.16   www.dashboard.local
```

---

## 8. 🌐 /etc/hosts sur TA machine (pour le navigateur)

**Linux/Mac :**

```bash
sudo nano /etc/hosts
```

**Windows** (Notepad en admin) :

```
C:\Windows\System32\drivers\etc\hosts
```

Ajouter à la fin :

```
192.168.64.16   www.nuc.local
192.168.64.16   www.dashboard.local
```

---

## 9. 🧪 Tout tester

### Services :

```bash
systemctl status apache2
systemctl status mysql
systemctl status ssh
```

### VirtualHosts actifs :

```bash
sudo apache2ctl -S
# doit lister www.nuc.local et www.dashboard.local
```

### Sites web depuis la VM :

```bash
curl http://www.nuc.local           # doit afficher "NUC OK"
curl http://www.dashboard.local     # doit afficher "DASHBOARD OK"
curl http://192.168.64.16           # test par IP
```

### MySQL :

```bash
mysql -u iotuser -p iot_db
```

```sql
SHOW TABLES;
EXIT;
```

### Firewall :

```bash
sudo ufw status verbose
# doit voir 22, 80, 1883 en ALLOW
```

### Ports ouverts :

```bash
ss -tlnp
# doit voir :80 (apache), :3306 (mysql), :22 (ssh)
```

### Depuis un navigateur sur ta machine :

```
http://www.nuc.local
http://www.dashboard.local
```

Si ça marche pas, vider le cache DNS :

```bash
# Linux
sudo systemd-resolve --flush-caches

# Windows
ipconfig /flushdns
```

---

## 🗺️ Ordre résumé

```
S'orienter → apt install → Démarrer services →
UFW → MySQL → VirtualHosts → /etc/hosts VM →
/etc/hosts ta machine → Tester tout
```

---

## 📋 Tableau récap des tests

| Quoi | Commande | Résultat attendu |
|---|---|---|
| Apache tourne | `systemctl status apache2` | active (running) |
| MySQL tourne | `systemctl status mysql` | active (running) |
| SSH tourne | `systemctl status ssh` | active (running) |
| Site nuc | `curl http://www.nuc.local` | NUC OK |
| Site dashboard | `curl http://www.dashboard.local` | DASHBOARD OK |
| MySQL connexion | `mysql -u iotuser -p iot_db` | connecté sans erreur |
| Firewall | `sudo ufw status verbose` | 22/80/1883 ALLOW |
| Ports | `ss -tlnp` | :22 :80 :3306 visibles |

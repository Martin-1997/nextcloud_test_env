# Nextcloud mit Apache, Redis & MariaDB in WSL2 oder Lima deployen


### Option B — macOS: Lima

```bash
brew install lima
limactl start --name=nextcloud template:ubuntu-24.04
```

Shell öffnen:

```bash
limactl shell nc-workshop
```

systemd ist im aktuellen Ubuntu-24.04-Lima-Template bereits aktiv — kein manueller Schritt nötig.

### Beide Umgebungen: Portforwarding funktioniert automatisch

Sowohl WSL2 als auch Lima leiten `localhost`-Ports vom Gast automatisch an den Host weiter. Später reicht also `http://localhost` im Windows-/macOS-Browser — kein `netsh`, kein manuelles Portmapping.

**Wichtig für Modul-3-Lektion "Dateirechte":** Das Nextcloud-Datenverzeichnis muss im nativen Linux-Dateisystem liegen (z.B. `~/...` oder `/var/www/...`), **nicht** unter `/mnt/c/...` (WSL) — sonst funktioniert das Berechtigungsmodell (`www-data`-Ownership) nicht wie erwartet.

---

## Schritt 2: Pakete installieren (Apache, PHP, MariaDB, Redis)

```bash
apt install -y sudo vim bzip2

sudo apt update && sudo apt upgrade -y

sudo apt install -y apache2 mariadb-server redis-server \
  libapache2-mod-php php-gd php-mysql php-curl php-mbstring \
  php-intl php-gmp php-xml php-imagick php-zip php-apcu php-redis
```

**Erklärung für die Teilnehmer:**
- `libapache2-mod-php` installiert automatisch die auf Ubuntu 24.04 vorinstallierte PHP-Version (PHP 8.3) — Nextcloud 34 unterstützt PHP 8.2–8.5, keine PPA nötig.
- `php-apcu` = lokaler Cache, `php-redis` = PHP-Modul für Redis (Distributed Cache + File Locking) — beide zusammen sind die von Nextcloud empfohlene Kombination.
- `redis-server` aktiviert und startet Redis automatisch als systemd-Dienst.

Kurzcheck:

```bash
php -v
sudo systemctl status apache2
sudo systemctl status mariadb
sudo systemctl status redis-server
```

---

## Schritt 3: MariaDB absichern & Datenbank anlegen

```bash
sudo mariadb-secure-installation
```

(Auf älteren Systemen heißt der Befehl `mysql_secure_installation` — beides landet beim gleichen Assistenten.) Root-Passwort setzen, anonyme User/Test-Datenbank entfernen — alles mit "Y" bestätigen.

Dedizierten, least-privilege Datenbank-User anlegen (**nicht** `root` für Nextcloud verwenden):

```bash
sudo mariadb
```

In der MariaDB-Konsole:

```sql
CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'CHANGE_ME';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
FLUSH PRIVILEGES;
quit;

# Use database
use nextcloud;
```

**Erklärung:** Der User `nextcloud` hat nur Rechte auf die Datenbank `nextcloud` — nicht auf den ganzen Server. Gleiches Prinzip wie bei Redis/S3 (least-privilege Service-Accounts).

---

## Schritt 4: Redis prüfen

Redis läuft nach der Paketinstallation bereits lokal auf `127.0.0.1:6379`. Kurzcheck:

```bash
redis-cli ping
```

Erwartete Ausgabe: `PONG`

Für dieses Lab-Setup reicht die TCP-Standardverbindung. (In einer echten Kunden-VM mit Nextcloud und Redis auf demselben Host empfiehlt die offizielle Doku stattdessen einen Unix-Socket — das lässt sich als Vertiefung erwähnen, aber für den Workshop unnötige Komplexität.)

---

## Schritt 5: Nextcloud herunterladen & entpacken

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
wget https://download.nextcloud.com/server/releases/latest.tar.bz2.sha256
sha256sum -c --ignore-missing latest.tar.bz2.sha256

tar -xjf latest.tar.bz2
sudo mv nextcloud /var/www/
sudo chown -R www-data:www-data /var/www/nextcloud
rm latest.tar.bz2 latest.tar.bz2.sha256
```

**Erklärung "Dateirechte" (Kernlektion):** `www-data` ist der Benutzer, unter dem Apache läuft. Ohne `chown -R www-data:www-data` kann Nextcloud später nicht in sein eigenes Datenverzeichnis schreiben — die häufigste Stolperfalle bei manuellen Installationen.

---

## Schritt 6: Apache-VirtualHost einrichten

```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

Inhalt:

```apache
<VirtualHost *:80>
    DocumentRoot /var/www/nextcloud/
    ServerName localhost

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>
</VirtualHost>
```

Aktivieren und Standard-Seite deaktivieren:

```bash
sudo a2ensite nextcloud.conf
sudo a2dissite 000-default.conf
```

Benötigte und empfohlene Apache-Module aktivieren:

```bash
sudo a2enmod rewrite headers env dir mime
```

- `rewrite` ist **zwingend erforderlich** (Pretty URLs, interne Weiterleitungen).
- `headers`, `env`, `dir`, `mime` sind offiziell empfohlen.

Apache neu starten:

```bash
sudo systemctl restart apache2
```

Kontrolle, ob die Module aktiv sind:

```bash
apache2ctl -M | grep -E "rewrite|headers|env_module|dir_module|mime_module"
```

---

## Schritt 7: Nextcloud per `occ` installieren

```bash
cd /var/www/nextcloud
sudo -u www-data php occ maintenance:install \
  --database "mysql" \
  --database-name "nextcloud" \
  --database-user "nextcloud" \
  --database-pass "CHANGE_ME" \
  --admin-user "admin" \
  --admin-pass "admin" \
  --data-dir "/var/www/nextcloud/data"
```

Bei Erfolg erscheint: `Nextcloud was successfully installed`

**Wichtig:** `occ` muss immer als `www-data` laufen (`sudo -u www-data php occ ...`) — nie als eigener User oder root, sonst verschieben sich Datei-Owner wieder.

---

## Schritt 8: Redis in `config.php` aktivieren

```bash
sudo vim /var/www/nextcloud/config/config.php
```

Innerhalb des `$CONFIG = array( ... );`-Blocks ergänzen:

```php
'memcache.local' => '\OC\Memcache\APCu',
'memcache.locking' => '\OC\Memcache\Redis',
'redis' => [
    'host' => '127.0.0.1',
    'port' => 6379,
],
```

Apache neu starten, damit die Änderung greift:

```bash
sudo systemctl restart apache2
```

**Erklärung:** `memcache.local` (APCu) beschleunigt lokale Lookups, `memcache.locking` (Redis) übernimmt das Transactional File Locking — ohne das kann es bei gleichzeitigen Zugriffen zu Datei-Sperrkonflikten kommen.

---

## Schritt 9: Verifikation

Im Windows-/macOS-Browser:

```
http://localhost
```

`occ status` zur Kontrolle:

```bash
sudo -u www-data php occ status
```

PHP-Module gegenchecken:

```bash
php -m | grep -iE "redis|apcu"
```

In der Admin-Oberfläche (**Einstellungen → Übersicht**) sollte danach keine Warnung mehr zu fehlendem Memory-Caching erscheinen.

---

## Schritt 10: Troubleshooting-Kurzreferenz

| Symptom | Wahrscheinliche Ursache | Befehl zur Diagnose |
|---|---|---|
| Weiße Seite / Server Error 500 | Dateirechte falsch | `sudo chown -R www-data:www-data /var/www/nextcloud` |
| "Trusted domain" Fehler im Browser | `localhost` fehlt in `trusted_domains` | `sudo -u www-data php occ config:system:get trusted_domains` |
| `occ` bricht mit PHP-Fehler ab | Falscher User oder falsches Verzeichnis | Immer aus `/var/www/nextcloud` und als `www-data` ausführen |
| Redis-Warnung bleibt in der Admin-UI | `config.php` nicht gespeichert oder Apache nicht neugestartet | `sudo systemctl restart apache2`, Datei erneut prüfen |
| `apache2ctl -M` zeigt Modul nicht | Modul nicht aktiviert oder Apache nicht neugestartet | `sudo a2enmod <modul>` gefolgt von `sudo systemctl restart apache2` |

---

## Optional — nächster Schritt: Cron statt AJAX

Für den Übergang zu Modul 4 (occ & Admin-Oberfläche) bietet sich an, direkt den Hintergrundjob-Modus umzustellen:

```bash
sudo crontab -u www-data -e
```

Zeile ergänzen:

```
*/5 * * * * php -f /var/www/nextcloud/cron.php
```

Und Nextcloud selbst auf Cron umstellen:

```bash
sudo -u www-data php occ background:cron
```

---

## Quellen

- [Installation on Linux – Nextcloud 34 Administration Manual](https://docs.nextcloud.com/server/stable/admin_manual/installation/source_installation.html)
- [Example installation on Ubuntu 24.04 LTS](https://docs.nextcloud.com/server/stable/admin_manual/installation/example_ubuntu.html)
- [Installing from command line (occ maintenance:install)](https://docs.nextcloud.com/server/stable/admin_manual/installation/command_line_installation.html)
- [Memory caching (APCu/Redis)](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/caching_configuration.html)
- [System requirements / unterstützte PHP-Versionen](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html)
- [Lima – Linux virtual machines](https://lima-vm.io/docs/)
- [Ubuntu Blog – systemd support in WSL](https://ubuntu.com/blog/ubuntu-wsl-enable-systemd)
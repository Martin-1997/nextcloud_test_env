# occ-Befehlsreferenz für den Workshop

Alle Befehle **immer als `www-data`** ausführen, aus dem Nextcloud-Installationsverzeichnis heraus (sonst verschieben sich Datei-Owner oder der Befehl findet die Installation nicht):

```bash
cd /var/www/nextcloud
sudo -u www-data php occ <befehl> [optionen]
```

**Geprüft gegen:** aktuelle offizielle Nextcloud-34-Dokumentation (docs.nextcloud.com). Quellen am Ende des Dokuments.

---

## 1. Installation

### `occ maintenance:install`

Installiert Nextcloud einmalig auf einer frisch entpackten Instanz — Ersatz für den Web-Installer, wenn man die Kommandozeile bevorzugt (z.B. bei Skript-/Automatisierungs-Deployments).

```bash
sudo -u www-data php occ maintenance:install \
  --database "mysql" \
  --database-name "nextcloud" \
  --database-user "nextcloud" \
  --database-pass "CHANGE_ME" \
  --admin-user "admin" \
  --admin-pass "CHANGE_ME_TOO" \
  --data-dir "/var/www/nextcloud/data"
```

Legt Admin-Konto, Datenbankschema und `config.php` in einem Rutsch an. Danach ist die Instanz sofort einsatzbereit.

---

## 2. Status & Diagnose

### `occ status`

Zeigt auf einen Blick, ob die Instanz installiert ist, im Wartungsmodus steckt und welche Version läuft. Guter erster Befehl bei jedem Support-Fall.

```bash
sudo -u www-data php occ status
```

### `occ check`

Prüft, ob grundlegende Systemvoraussetzungen (Schreibrechte, PHP-Module etc.) noch erfüllt sind. Läuft normalerweise automatisch bei jedem `occ`-Aufruf im Hintergrund mit; manuell nützlich, um genau diesen Teil isoliert zu testen.

```bash
sudo -u www-data php occ check
```

---

## 3. Wartungsmodus & Reparatur

### `occ maintenance:mode`

Sperrt alle aktiven Sitzungen und verhindert neue Logins — Pflichtschritt vor jedem Update oder größeren Eingriff an der Datenbank.

```bash
sudo -u www-data php occ maintenance:mode --on
sudo -u www-data php occ maintenance:mode --off
```

### `occ maintenance:repair`

Führt die gleichen Reparaturschritte aus, die auch automatisch bei jedem Update laufen (Datenbank-Aufräumarbeiten). Manuell selten nötig, aber hilfreich, wenn die Admin-Übersicht Inkonsistenzen meldet.

```bash
sudo -u www-data php occ maintenance:repair
```

---

## 4. Konfiguration (config.php ohne Editor)

### `occ config:list`

Gibt die komplette aktuelle Konfiguration als JSON aus — schneller Überblick, ohne die Datei manuell zu öffnen.

```bash
sudo -u www-data php occ config:list system
```

### `occ config:system:get`

Liest **einen einzelnen** Konfigurationswert aus. Praktisch für Troubleshooting, z.B. um `trusted_domains` zu prüfen.
`trusted_domains` sind eine Whitelist aller IPs/Domains, die auf den Nextcloud Webserver zugreifen dürfen

```bash
sudo -u www-data php occ config:system:get trusted_domains
```

### `occ config:system:set`

Setzt **einen einzelnen** Konfigurationswert — legt ihn an, falls er noch nicht existiert. `--type` bei Nicht-Text-Werten nicht vergessen (z.B. `boolean`, `integer`), sonst landet ggf. ein String in `config.php`, wo eigentlich ein anderer Typ erwartet wird.

```bash
sudo -u www-data php occ config:system:set maintenance_window_start --type=integer --value=1
```
`maintenance_window_start` legt den Beginn eines 4-Stunden-Fensters fest, in dem der Server selbst Maintenance-Aufgaben durchführt

---

## 5. Benutzer & Gruppen

### `occ user:add`

Legt einen neuen Benutzer an, optional mit Anzeigename und Gruppenzuordnung. Nicht existierende Gruppen werden automatisch mit angelegt — bei Tippfehlern im Gruppennamen entsteht so ungewollt eine neue Gruppe statt einer Fehlermeldung.

```bash
## Allow weak passwords
sudo -u www-data php occ app:disable password_policy

sudo -u www-data php occ user:add --display-name="Peter Schmidt" --group="users" peter
```

### `occ user:resetpassword`

Setzt das Passwort eines Benutzers zurück — funktioniert auch für Admin-Konten und ist damit der Standardweg, ein verlorenes Admin-Passwort wiederherzustellen.

```bash
sudo -u www-data php occ user:resetpassword peter
```

### `occ user:list`

Listet alle vorhandenen Benutzer auf — guter Schnellcheck nach einer Migration oder LDAP-Anbindung.

```bash
sudo -u www-data php occ user:list
```

### `occ group:add`

Legt eine neue Gruppe an (unabhängig von Benutzern).

```bash
sudo -u www-data php occ group:add support
```

### `occ group:adduser`

Fügt einen oder mehrere bestehende Benutzer zu einer bestehenden Gruppe hinzu.

```bash
sudo -u www-data php occ group:adduser support peter
```

---

## 6. Apps

### `occ app:list`

Zeigt alle installierten Apps inkl. Status (aktiviert/deaktiviert) — Ausgangspunkt für jede App-Verwaltung ohne die Admin-Oberfläche.

```bash
sudo -u www-data php occ app:list
```

### `occ app:enable` / `occ app:disable`

Aktiviert bzw. deaktiviert eine App direkt über die Kommandozeile — nützlich für Skripte oder wenn die Admin-UI wegen einer defekten App gerade nicht lädt.

```bash
sudo -u www-data php occ app:enable files_external
sudo -u www-data php occ app:disable files_external
```

---

## 7. Updates

### `occ upgrade`

Führt nach dem Austauschen der Programmdateien das eigentliche Datenbank-Update durch — der zentrale Befehl im Update-Workflow (Backup → Wartungsmodus → Dateien tauschen → `occ upgrade` → Wartungsmodus aus).

```bash
sudo -u www-data php occ upgrade
```

### `occ db:add-missing-indices`

Legt optionale Datenbank-Indizes nach, die z.B. nach einem Update noch fehlen (wird in der Admin-Übersicht als Warnung angezeigt). Kann je nach Datenmenge einige Zeit dauern.

```bash
sudo -u www-data php occ db:add-missing-indices
```

---

## 8. Hintergrundjobs (Cron)

### `occ background:cron`

Stellt den Hintergrundjob-Modus auf Cron um (empfohlen, statt AJAX/Webcron) — muss durch einen echten Cronjob-Eintrag ergänzt werden (`*/5 * * * * php -f /var/www/nextcloud/cron.php`), sonst laufen die Jobs trotzdem nicht.

```bash
sudo -u www-data php occ background:cron
```

### `occ background-job:list`

Listet alle registrierten Hintergrundjobs mit Zeitpunkt ihres letzten Laufs auf — guter Weg zu prüfen, ob Cron tatsächlich regelmäßig durchläuft, statt nur auf die Warnung in der Admin-Übersicht zu vertrauen.

```bash
sudo -u www-data php occ background-job:list
```

---

## Quellen

- [Command line installation](https://docs.nextcloud.com/server/stable/admin_manual/installation/command_line_installation.html)
- [Using the occ command](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/occ_command.html)
- [User & group commands](https://docs.nextcloud.com/server/stable/admin_manual/occ_users.html)
- [Apps, background jobs & config commands](https://docs.nextcloud.com/server/stable/admin_manual/occ_apps.html)
- [How to upgrade](https://docs.nextcloud.com/server/stable/admin_manual/maintenance/upgrade.html)
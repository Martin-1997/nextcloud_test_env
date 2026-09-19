# Nextcloud Office (EuroOffice) deployen — Ergänzung zum Files-Deployment

**Zweck:** Aufbauend auf eurer bestehenden Nextcloud-Installation (siehe `nextcloud-deployment-wsl-lima-schritt-fuer-schritt.md`) den EuroOffice Document Server ergänzen — die von Nextcloud seit dem Hub-26-Spring-Release bevorzugte Office-Option (AGPL-Fork, Ersatz für die frühere OnlyOffice-Partnerschaft).

**Voraussetzung:** Eine laufende Nextcloud-Instanz nach dem Files-Deployment-Guide (Apache, MariaDB/PostgreSQL, Redis bereits eingerichtet).

**Geprüft gegen:** aktuelle offizielle Euro-Office- und Nextcloud-Dokumentation (Stand: Installationsanleitung Ubuntu 24.04 LTS). Quellen am Ende des Dokuments.

⚠️ **Wichtiger Hinweis zum Ressourcenbedarf:** Der Document Server bringt seinen **eigenen** PostgreSQL- und RabbitMQ-Stack mit — unabhängig von Nextclouds eigener Datenbank aus dem Files-Deployment. Redis lässt sich dagegen wiederverwenden (siehe Schritt 1). Mindestens 3–4 GB RAM zusätzlich einplanen (8 GB empfohlen für Mehrbenutzerbetrieb), 10 GB freier Speicher.

---

## Schritt 1: Document-Server-Abhängigkeiten installieren

```bash
sudo apt-get update
sudo apt-get install -y postgresql rabbitmq-server nginx supervisor
```

**Erklärung:** PostgreSQL ist eine **separate** Instanz — nicht die MariaDB, die ihr für Nextcloud selbst laufen habt (PostgreSQL ist bei EuroOffice fest verdrahtet, dazu keine Alternative). RabbitMQ übernimmt die Konvertierungs-Queue.

**Redis bewusst nicht neu installiert:** Anders als PostgreSQL ist Redis beim Document Server frei konfigurierbar (Host, Port, sogar eine eigene logische Datenbank-Nummer) und muss nicht lokal neu aufgesetzt werden. Wir nutzen in Schritt 6 denselben Redis-Server aus eurem Files-Deployment weiter — nur mit einer anderen Datenbank-Nummer, damit sich die Keys nicht mit Nextclouds eigenem `memcache.locking` überschneiden.

**Nur auf Debian nötig, auf Ubuntu überspringen:** Debian braucht für die mitgelieferten Microsoft-Kernschriftarten (`ttf-mscorefonts-installer`) die `contrib`-Komponente, die dort nicht standardmäßig aktiv ist. Ubuntu hat diese Schriftarten bereits über `universe`/`multiverse` verfügbar — dieser Schritt entfällt auf Ubuntu 24.04.

---

## Schritt 2: PostgreSQL-Datenbank für den Document Server anlegen

```bash
sudo -u postgres psql -c "CREATE USER ds WITH PASSWORD 'CHANGE_ME';"
sudo -u postgres psql -c "CREATE DATABASE ds OWNER ds;"
```

**Erklärung:** Das Post-Install-Skript des Pakets verbindet sich während der Installation direkt mit dieser Datenbank — sie muss **vor** der Paketinstallation existieren, sonst bricht sie ab.

---

## Schritt 3: Installation nicht-interaktiv vorbereiten (debconf)

```bash
echo "ds ds/db-type select postgres
ds ds/db-host string localhost
ds ds/db-port string 5432
ds ds/db-user string ds
ds ds/db-pwd password CHANGE_ME
ds ds/db-name string ds" | sudo debconf-set-selections
```

**Erklärung:** Der Paket-Installer fragt sonst interaktiv nach den Datenbank-Zugangsdaten. Diese Vorbelegung (debconf) beantwortet die Fragen automatisch — praktisch für Skript-/Automatisierungs-Deployments.

---

## Schritt 4: Euro-Office Document Server herunterladen & installieren

```bash
cd /tmp
wget "https://github.com/Euro-Office/DocumentServer/releases/download/v9.3.2/euro-office-documentserver_9.3.1-dev.1_arm64.deb" \
  -O euro-office-documentserver.deb
sudo apt-get install -y ./euro-office-documentserver.deb
```

**Hinweis:** Der Installer generiert dabei Fonts, WOPI-Schlüssel und JS-Caches — das kann ein bis zwei Minuten dauern, wirkt aber nicht hängengeblieben.

---

## Schritt 5: Nextcloud-Connector-App installieren

```bash
cd /var/www/nextcloud
sudo -u www-data php occ app:install eurooffice
```

Falls das Kommando meldet, die App sei bereits vorhanden, mit `occ app:enable eurooffice` prüfen.

---

## Schritt 6: Bestehenden Redis-Server einbinden

Statt eines eigenen Redis-Servers auf eine **eigene logische Datenbank-Nummer** des bereits laufenden Redis aus dem Files-Deployment ausweichen (Redis unterstützt standardmäßig die Nummern 0–15 über `SELECT`; Nextcloud selbst nutzt für `memcache.locking` üblicherweise Datenbank 0):

In der Document-Server-Konfiguration (`local.json` bzw. der entsprechenden Umgebungsvariable) eintragen:

```json
"redis": {
  "host": "127.0.0.1",
  "port": 6379,
  "db": 1
}
```

**Erklärung:** `db: 1` sorgt dafür, dass EuroOffice in einem eigenen, isolierten Namensraum innerhalb desselben Redis-Prozesses arbeitet — die Keys kollidieren dadurch nicht mit denen, die Nextcloud selbst für Cache/Locking verwendet. Ein zweiter `redis-server`-Prozess ist damit nicht nötig.

---

## Schritt 7: Gemeinsamen JWT-Secret einrichten

```bash
openssl rand -hex 32
```
7a72c3de24ada03f917ebde6b7da924f5b5528b856ab002e2cf71abda8ad574f

Diesen Wert an **zwei** Stellen eintragen — er muss auf beiden Seiten exakt übereinstimmen:

1. **Document Server:** in dessen Konfiguration (siehe Euro-Office-Doku zur JWT-Konfiguration)
2. **Nextcloud:** Admin-Oberfläche → Einstellungen → Nextcloud Office → Server-Adresse + Secret eintragen

**Stolperfalle aus der Community:** Ein zu kurzer/simpler Secret führt zu kryptischen Fehlermeldungen ("JWT Secret is too short"). Immer über `openssl rand -hex 32` generieren, nicht von Hand ausdenken.

---

## Schritt 8: Port-Konflikt auflösen (nur bei Single-Host-Betrieb)

Document Server und Nextcloud laufen beide standardmäßig auf **Port 80**. Auf demselben Host müsst ihr einen der beiden auf einen anderen Port legen — z.B. den Document Server über die Nginx-Konfiguration auf Port 8080 umstellen und in der Nextcloud-Admin-Oberfläche entsprechend referenzieren.

---

## Schritt 9: Verifikation

```bash
sudo -u www-data php occ status
systemctl status supervisor
redis-cli -n 1 ping
```

Danach in der Nextcloud-Weboberfläche ein neues Dokument anlegen — der Editor sollte sich öffnen, ohne eine "Document Server nicht erreichbar"-Meldung zu zeigen. Für einen isolierten Test ohne Nextcloud-Integration bringt der Document Server außerdem eine **eingebaute Beispiel-App** mit, die den Editor direkt im Browser testet.

---

## Troubleshooting-Kurzreferenz

| Symptom | Wahrscheinliche Ursache | Was prüfen |
|---|---|---|
| Paketinstallation bricht ab | PostgreSQL-User/DB fehlt oder debconf nicht vorbelegt | Schritt 2 + 3 in dieser Reihenfolge wiederholen |
| "JWT Secret is too short/invalid" | Secret zu kurz oder auf beiden Seiten unterschiedlich | Mit `openssl rand -hex 32` neu generieren, an beiden Stellen identisch eintragen |
| SSL-Zertifikatsfehler bei Discovery-Aufruf | Self-signed-Zertifikat zwischen Nextcloud und Document Server | In Produktion: gültiges HTTPS-Zertifikat zwingend, kein Self-signed |
| App nicht im App Store sichtbar | Bekannte aktuelle Einschränkung | `occ app:install eurooffice` statt Store-UI verwenden |
| Beide Dienste kollidieren auf Port 80 | Single-Host-Betrieb ohne Port-Trennung | Document Server auf eigenen Port legen (Schritt 8) |
| Dokument-Sitzungen/Locks wirken fehlerhaft | Gleiche Redis-Datenbank-Nummer wie Nextcloud verwendet | `db` in der Redis-Konfiguration auf eine ungenutzte Nummer setzen (Schritt 6) |

---

## Deinstallation (falls nötig)

```bash
sudo apt-get remove --purge euro-office-documentserver
sudo -u postgres psql -c "DROP DATABASE ds;"
sudo -u postgres psql -c "DROP USER ds;"
```

---

## Quellen

- [How to install Euro-Office in Nextcloud – offizieller Nextcloud-Blog](https://nextcloud.com/blog/how-to-install-euro-office/)
- [Euro-Office Installation Guide (Ubuntu/Debian/Fedora/Docker)](https://euro-office.github.io/documentation/installation/)
- [Euro-Office app for Nextcloud – GitHub](https://github.com/Euro-Office/eurooffice-nextcloud)
- [Euro-Office DocumentServer – GitHub Releases](https://github.com/Euro-Office/DocumentServer/releases)
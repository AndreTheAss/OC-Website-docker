# WordPress-Stack für Portainer

Dieses Repository stellt eine Docker-Compose-Definition bereit, um über Portainer (oder direkt per `docker compose`) eine WordPress-Instanz auf dem bestehenden Netzwerk `npm_default` bereitzustellen. Zusätzlich wird ein FTP-Dienst (externer Command-Port `26`) eingebunden, der direkt auf die WordPress-Dateien zugreift.

## Inhalt der Compose-Datei
- **MariaDB**: persistent mit `db_data`-Volume.
- **WordPress (Apache)**: nutzt das gemeinsame `wordpress_data`-Volume für Dateien.
- **FTP (fauria/vsftpd)**: greift auf das `wordpress_data`-Volume zu und stellt einen FTP-Login bereit.
- **Netzwerke**: nutzt das externe Netzwerk `npm_default` (z. B. für Nginx Proxy Manager) und ein internes Netzwerk `wp-internal` für die DB-Kommunikation.

## Vorbereitungen
1. Stelle sicher, dass das externe Docker-Netzwerk `npm_default` bereits existiert (z. B. durch Nginx Proxy Manager). Falls nicht, lege es an:
   ```bash
   docker network create --driver bridge npm_default
   ```
2. Passe alle Platzhalter-Credentials in der `docker-compose.yml` an (z. B. `change-me`). Verwende starke und unterschiedliche Passwörter für Datenbank und FTP.
3. Setze vor dem Deploy idealerweise die Umgebungsvariable `PASV_ADDRESS` auf die öffentliche IP/Domain des Hosts, damit der passive FTP-Modus funktioniert (z. B. in Portainer unter *Environment variables* oder per `.env`).
4. Standard-Passivports sind `21100-21110`. Wenn diese bereits belegt sind, passe sie per Umgebungsvariablen `PASV_MIN_PORT`/`PASV_MAX_PORT` an und öffne die Ports in Firewall/Portainer.

## Deployment mit Portainer
1. Öffne Portainer, wähle **Stacks** und klicke auf **Add stack**.
2. Gib einen Namen ein (z. B. `wordpress-ftp`) und füge den Inhalt der `docker-compose.yml` in das Web-Editor-Feld ein.
3. Passe die gewünschten Passwörter/Benutzernamen an und überprüfe die Ports (`26` sowie Standard `21100-21110` für FTP-Passivmodus). Stelle sicher, dass sie in der Firewall freigeschaltet sind.
4. Deploye den Stack. Nginx Proxy Manager kann den `wordpress`-Service direkt über das gemeinsame `npm_default`-Netzwerk erreichen.

## Deployment per CLI
```bash
# Optional: PASV_ADDRESS vorher setzen
export PASV_ADDRESS=deine.offentliche.ip

# Stack starten
docker compose up -d
```

## Hinweise
- Das Volume `wordpress_data` wird von WordPress und dem FTP-Container gemeinsam genutzt, sodass hochgeladene Dateien direkt im CMS erscheinen.
- Der Datenbankdienst ist nur im internen Netzwerk sichtbar. WordPress ist sowohl im internen Netzwerk (für die DB) als auch im `npm_default`-Netzwerk erreichbar.
- Wenn du andere FTP-Ports verwenden möchtest, passe die Port-Mappings im `ftp`-Service an (Command-Port `26` oder Passive Ports `PASV_MIN_PORT`–`PASV_MAX_PORT`) und öffne die Ports in der Firewall.

## Berechtigungen und Dateizugriffe (FTP & WordPress)
- Der FTP-Benutzer wird in der `docker-compose.yml` auf UID/GID `33` gesetzt (`www-data`), damit WordPress (läuft als `www-data`) und FTP dieselben Dateibesitzer verwenden. Falls du andere Nutzer verwenden möchtest, passe `FTP_USER_UID` und `FTP_USER_GID` an.
- Nach dem ersten Start (oder nach größeren Datei-Imports) kannst du alle Dateien einmalig auf `www-data` setzen, damit WordPress Schreibrechte hat:
  ```bash
  docker compose exec wordpress chown -R www-data:www-data /var/www/html
  ```
- Standardmäßig erzeugt FTP neue Dateien mit umask `022` (über `LOCAL_UMASK`) und Modus `0666` (durch `FILE_OPEN_MODE`), sodass WordPress (als `www-data`) darauf zugreifen kann. Für restriktivere Rechte passe die beiden Variablen im `ftp`-Dienst an.

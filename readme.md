# WordPress-Stack für Portainer

Dieses Repository stellt eine Docker-Compose-Definition bereit, um über Portainer (oder direkt per `docker compose`) eine WordPress-Instanz auf dem bestehenden Netzwerk `npm_default` bereitzustellen. Zusätzlich wird ein FTP-Dienst (externer Command-Port `26`) eingebunden, der direkt auf die WordPress-Dateien zugreift.

## Inhalt der Compose-Datei
- **MariaDB**: persistent mit `db_data_oc`-Volume (eigener Namespace, kollidiert nicht mit bestehenden DB-Stacks). Service-Name: `olympiccamp-db` (für den DB-Host in WordPress).
- **WordPress (Apache)**: nutzt das gemeinsame `wordpress_data_oc`-Volume für Dateien (separat von anderen WP-Stacks, damit kein altes `wp-config.php` gezogen wird). Verbindet sich mit `olympiccamp-db:3307`.
- **FTP (fauria/vsftpd)**: greift auf das `wordpress_data_oc`-Volume zu und stellt einen FTP-Login bereit. Das FTP-Home ist `/home/vsftpd/wordpress`, damit du die WordPress-Dateien (inkl. `wp-config.php`) direkt siehst.
- **Netzwerke**: nutzt das externe Netzwerk `npm_default` (z. B. für Nginx Proxy Manager) und ein internes Netzwerk `wp-internal` für die DB-Kommunikation.

## Vorbereitungen
1. Stelle sicher, dass das externe Docker-Netzwerk `npm_default` bereits existiert (z. B. durch Nginx Proxy Manager). Falls nicht, lege es an:
   ```bash
   docker network create --driver bridge npm_default
   ```
2. Passe alle Platzhalter-Credentials in der `docker-compose.yml` an (z. B. `change-me`). Verwende starke und unterschiedliche Passwörter für Datenbank und FTP. Die Datenbank nutzt einen eindeutigen Namen/Benutzer (`wordpress_oc` / `wp_user_oc`) und lauscht intern auf Port `3307` (mit `mariadbd --port=3307`), damit keine bestehenden DBs mit Standardport `3306` oder Standardnamen berührt werden. Standard-Image ist `mariadb:latest`; falls du eine feste Version brauchst, pinne den Tag (z. B. `mariadb:11.3`). WordPress nutzt `wordpress:latest`; auch hier kannst du bei Bedarf auf eine feste Version pinnen (z. B. `wordpress:6.6.1-php8.3-apache`). Eigene Volumes (`db_data_oc`, `wordpress_data_oc`) verhindern, dass eine alte `wp-config.php` aus einem anderen Stack falsche DB-Hosts (z. B. `wp_db`) zieht. Der DB-Host in WP ist `olympiccamp-db:3307`.
3. Setze vor dem Deploy idealerweise die Umgebungsvariable `PASV_ADDRESS` auf die öffentliche IP/Domain des Hosts (Standard hier: `87.106.81.201`), damit der passive FTP-Modus funktioniert (z. B. in Portainer unter *Environment variables* oder per `.env`). Wenn du über Nginx Proxy Manager auf `olympic-camp.de` routest, kannst du auch die Domain in `PASV_ADDRESS` setzen.
4. Standard-Passivports sind jetzt `21210-21220` (gegen Konflikte mit anderen FTP-Stacks). Wenn diese bereits belegt sind, passe sie per Umgebungsvariablen `PASV_MIN_PORT`/`PASV_MAX_PORT` an und öffne die Ports in Firewall/Portainer.

## Deployment mit Portainer
1. Öffne Portainer, wähle **Stacks** und klicke auf **Add stack**.
2. Gib einen Namen ein (z. B. `wordpress-ftp`) und füge den Inhalt der `docker-compose.yml` in das Web-Editor-Feld ein.
3. Passe die gewünschten Passwörter/Benutzernamen an und überprüfe die Ports (`26` sowie Standard `21210-21220` für FTP-Passivmodus). Stelle sicher, dass sie in der Firewall freigeschaltet sind. Die WordPress-Datenbank ist intern auf Port `3307` erreichbar (Service `olympiccamp-db:3307`) und verwendet den eindeutigen DB-Namen `wordpress_oc`.
4. Falls du Passivports gar nicht nach außen freigeben willst (z. B. wenn FTP nur intern genutzt wird), entferne das Port-Mapping des Passivbereichs und nutze FTP ausschließlich intern.
5. Deploye den Stack. Nginx Proxy Manager kann den `wordpress`-Service direkt über das gemeinsame `npm_default`-Netzwerk erreichen.

## Deployment per CLI
```bash
# Optional: PASV_ADDRESS vorher setzen
export PASV_ADDRESS=deine.offentliche.ip

# Stack starten
docker compose up -d
```

## Hinweise
- Das Volume `wordpress_data` wird von WordPress und dem FTP-Container gemeinsam genutzt, sodass hochgeladene Dateien direkt im CMS erscheinen. Das FTP-Home `/home/vsftpd/wordpress` zeigt direkt auf das WordPress-Verzeichnis, sodass du `wp-config.php` per FTP erreichst.
- Der Datenbankdienst ist nur im internen Netzwerk sichtbar. WordPress ist sowohl im internen Netzwerk (für die DB) als auch im `npm_default`-Netzwerk erreichbar.
- Wenn du andere FTP-Ports verwenden möchtest, passe die Port-Mappings im `ftp`-Service an (Command-Port `26` oder Passive Ports `PASV_MIN_PORT`–`PASV_MAX_PORT`) und öffne die Ports in der Firewall. Wähle einen Bereich, der nicht von anderen FTP-Stacks genutzt wird (Standard: `21210-21220`).

## Berechtigungen und Dateizugriffe (FTP & WordPress)
- Der FTP-Benutzer wird in der `docker-compose.yml` auf UID/GID `33` gesetzt (`www-data`), damit WordPress (läuft als `www-data`) und FTP dieselben Dateibesitzer verwenden. Falls du andere Nutzer verwenden möchtest, passe `FTP_USER_UID` und `FTP_USER_GID` an.
- Nach dem ersten Start (oder nach größeren Datei-Imports) kannst du alle Dateien einmalig auf `www-data` setzen, damit WordPress Schreibrechte hat:
  ```bash
  docker compose exec wordpress chown -R www-data:www-data /var/www/html
  ```
- Standardmäßig erzeugt FTP neue Dateien mit umask `022` (über `LOCAL_UMASK`) und Modus `0666` (durch `FILE_OPEN_MODE`), sodass WordPress (als `www-data`) darauf zugreifen kann. Für restriktivere Rechte passe die beiden Variablen im `ftp`-Dienst an.

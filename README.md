# Mo 158

## Phase 1

### Projektplan

```mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#1E88E5',
      'primaryTextColor': '#ffffff',
      'primaryBorderColor': '#1565C0',
      'lineColor': '#000000',
      'secondaryColor': '#90CAF9',
      'tertiaryColor': '#FFB74D',
      'textColor': '#212121',
      'gridColor': '#e0e0e0',
      'background': '#fafafa'
    }
  }
}%%

gantt
    dateFormat  DD-MM-YYYY
    axisFormat  %d-%m-%Y
    title       Projektplan Software-Migration M158
    excludes    weekends

    section Entwicklung & Testing
    Projektplanung           :a1, 23-02-2025, 7d
    Dokumentation/Arbeitsjournal :after a1, 50d

    section Phase 2
    AWS                      :b1, after a1, 7d
    S1                       :milestone, after b1, 0d

    section Phase 3
    OS-konf                  :c1, after b1, 7d
    S2                       :milestone, after c1, 0d
    Webserver/DB             :c2, after c1, 7d
    S3                       :milestone, after c2, 0d
    V-Host                   :c3, after c2, 7d
    S4                       :milestone, after c3, 0d
    DNS                      :c4, after c3, 7d
    S5                       :milestone, after c4, 0d

    section Phase 4
    WP-Files                 :d1, after c4, 7d
    WP-DB                    :d2, after c4, 7d
    WP-Config                :d3, after c4, 7d
    S6                       :milestone, after d3, 0d

    section Phase 5
    Tests                    :e1, after d3,7d
    Health-Check             :e2, after d3,7d

```


### Architekturdiagramm

```mermaid
graph TD;
    A[Internet] --> B["AWS EC2 Webserver<br>nm158.local<br>IP: DHCP (Public)<br>10.0.0.77 (Private)"];
    D["Externer Client<br>Hostname: client-pc<br>IP: DHCP"] -->|Access via HTTPS| A;
    B -->|Database Connection| C["AWS RDS Datenbank<br>wp_m158_db<br>IP: 10.0.0.20"];
    B -->|Nginx Virtual Host| E["Virtual Host<br>/etc/nginx/sites-available/m158.local"];

    subgraph AWS Umgebung
        B
        C
        E
    end

    %% Farben und Stil
    style A fill:#90CAF9,stroke:#333,stroke-width:2px,color:#000000;
    style D fill:#90CAF9,stroke:#333,stroke-width:2px,color:#000000;
    style B fill:#81C784,stroke:#2e7d32,stroke-width:2px,color:#000000;
    style C fill:#FFB74D,stroke:#e65100,stroke-width:2px,color:#000000;
    style E fill:#BA68C8,stroke:#6A1B9A,stroke-width:2px,color:#ffffff;
```

## Phase 2

#### instanz 1

ipconfig 
![1](/test_1.png)

ping:
![1](/test_2.png)

ping 8.8.8.8
![1](/test_3.png)


#### instanz 2

ipconfig 
![1](/2test_1.png)

ping:
![1](/2test_2.png)

ping 8.8.8.8
![1](/2test_3.png)


## Phase 3

### DNS

![](/DNS/DNS.png)
![](/DNS/dns_internal_query.png)
![](/ionos.png)
![](/webserver.info.png)
# Webserver bis FTP-Einrichtung (Aufgaben 5–9)

---

## Aufgabe 5: Webserver (Apache + HTTPS + Rewrite)

### Ziel:
Sicherer Webserver mit HTTPS, Redirect von HTTP, RewriteEngine und Virtual Hosts.

### Schritte:
```bash
sudo apt update
sudo apt install apache2 -y
sudo a2enmod rewrite ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/apache-selfsigned.key \
  -out /etc/ssl/certs/apache-selfsigned.crt
```

### Konfiguration:
- `/etc/apache2/sites-available/000-default.conf`: HTTP → HTTPS Redirect
- `/etc/apache2/sites-available/edris-ssl.conf`: HTTPS + SSL-Zertifikat

### Screenshots:
![](/webserver.info.png)
![](/apache.png)
![](/webserver.info%20seite.png)

---

## Aufgabe 6: PHP + PHP-FPM

### Ziel:
Aktuelle PHP-Version mit `php.ini`-Anpassung und PHP-FPM Integration.

### Schritte:
```bash
sudo apt install php8.3 php8.3-fpm -y
sudo nano /etc/php/8.3/apache2/php.ini
# Werte anpassen:
# upload_max_filesize = 64M, post_max_size = 64M, memory_limit = 256M
```

### Screenshots:
![](/execution.png)
![](/php.png)

---

## Aufgabe 7: Dedizierter MariaDB-Server

### Ziel:
Eigener DB-Server mit Benutzer `wpuser`, eingeschränkten Rechten und Root nur lokal.

### Schritte:
```bash
sudo apt install mariadb-server -y
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
# bind-address = 0.0.0.0

# In MySQL:
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'%' IDENTIFIED BY 'passwort';
GRANT SELECT, INSERT, UPDATE, DELETE ON wordpress.* TO 'wpuser'@'%';
DELETE FROM mysql.user WHERE User='root' AND Host!='localhost';
```

### Screenshots:
![](/wpuser.png)

---

## Aufgabe 8: PhpMyAdmin via Docker + Subdomain

### Ziel:
PhpMyAdmin über `phpmyadmin.edris.info` per Docker + Apache Reverse Proxy.

### Schritte:
**docker-compose.yml**
```yaml
services:
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    environment:
      PMA_HOST: <DB-IP>
      PMA_PORT: 3306
    ports:
      - "8081:80"
```

**Apache-VHost:**
```apache
<VirtualHost *:80>
    ServerName phpmyadmin.edris.info
    ProxyPass / http://localhost:8081/
    ProxyPassReverse / http://localhost:8081/
</VirtualHost>
```

**SSL aktivieren:**
```bash
sudo certbot --apache -d phpmyadmin.edris.info
```

### Screenshots:
![](/myadminlogin.png)
![](/set.png)
![](/konfig.png)
---

## Aufgabe 9: FTP-Server (vsftpd + FTPS)

### Ziel:
Sicherer FTP-Zugang mit TLS-Verschlüsselung und eingeschränktem Zugriff.

### Schritte:
```bash
sudo apt install vsftpd -y
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/vsftpd.key \
  -out /etc/ssl/certs/vsftpd.crt

sudo adduser ftpuser
sudo usermod -d /var/www/html ftpuser
sudo chmod a-w /var/www/html
sudo mkdir /var/www/html/ftp
sudo chown ftpuser:ftpuser /var/www/html/ftp
```

**Wichtige vsftpd.conf Optionen:**
```ini
chroot_local_user=YES
allow_writeable_chroot=YES
ssl_enable=YES
force_local_data_ssl=YES
force_local_logins_ssl=YES
```

### Screenshots:
- ![vsftpd.conf Ausschnitt](screenshots/vsftpd_conf.png)
- ![FTP Login FileZilla](screenshots/filezilla_tls.png)
- ![Verzeichnisrechte](screenshots/ftp_permissions.png)


## Fazit

- Alle Systeme sind modular, sicher und dokumentiert
- SSL und Benutzerberechtigungen korrekt gesetzt
- Screenshots belegen erfolgreiche Umsetzung aller Aufgaben


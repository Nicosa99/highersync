# Nginx Konfiguration für HigherSync

## 1. Nginx-Konfigurationsdatei erstellen

### Für eine Domain (z.B. highersync.com)

Erstelle die Datei: `/etc/nginx/sites-available/highersync.com`

```nginx
# HTTP -> HTTPS Redirect
server {
    listen 80;
    listen [::]:80;
    server_name highersync.com www.highersync.com;

    # Let's Encrypt ACME Challenge
    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
        try_files $uri =404;
    }

    # Redirect zu HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name highersync.com www.highersync.com;

    # SSL Zertifikate (nach Let's Encrypt Setup)
    ssl_certificate /etc/letsencrypt/live/highersync.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/highersync.com/privkey.pem;

    # Moderne SSL Konfiguration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;

    # Upload-Größe anpassen (falls benötigt)
    client_max_body_size 50M;

    # Backend API (falls separate Backend/Frontend Architektur)
    location /api/ {
        proxy_pass http://localhost:3012;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_read_timeout 300s;
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
    }

    # Frontend / Hauptanwendung
    location / {
        proxy_pass http://localhost:3013;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Für eine Subdomain (z.B. app.highersync.com)

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name app.highersync.com;

    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
        try_files $uri =404;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name app.highersync.com;

    ssl_certificate /etc/letsencrypt/live/app.highersync.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.highersync.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    client_max_body_size 50M;

    location / {
        proxy_pass http://localhost:3012;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 2. Nginx-Konfiguration aktivieren

```bash
# Symlink erstellen
ln -s /etc/nginx/sites-available/highersync.com /etc/nginx/sites-enabled/

# Konfiguration testen
nginx -t

# Nginx neu laden
systemctl reload nginx
```

## 3. SSL-Zertifikat mit Let's Encrypt einrichten

### Vor dem SSL-Setup:
1. Stelle sicher, dass die Domain auf die Server-IP zeigt
2. Erstelle die Nginx-Konfiguration (OHNE SSL-Zeilen vorerst)
3. Aktiviere die Konfiguration und lade Nginx neu

### SSL-Zertifikat generieren:

```bash
# Für Hauptdomain
certbot certonly --webroot -w /var/www/letsencrypt -d highersync.com -d www.highersync.com

# Für Subdomain
certbot certonly --webroot -w /var/www/letsencrypt -d app.highersync.com
```

### Nach SSL-Generierung:
1. Füge die SSL-Zeilen zur Nginx-Konfiguration hinzu
2. Teste die Konfiguration: `nginx -t`
3. Lade Nginx neu: `systemctl reload nginx`

## 4. Automatische Zertifikatserneuerung

Certbot richtet automatisch einen Cronjob ein. Teste die Erneuerung:

```bash
certbot renew --dry-run
```

## Wichtige Hinweise

### Lieferfly wird NICHT beeinflusst
- Lieferfly läuft auf Ports 3010/3011
- Lieferfly hat eigene Nginx-Konfiguration: `/etc/nginx/sites-enabled/lieferfly.com`
- HigherSync hat komplett separate Konfiguration
- Keine Überschneidungen oder Konflikte möglich

### Port-Übersicht
- **HigherSync Backend**: 3012 (intern)
- **HigherSync Frontend**: 3013 (intern)
- **Nginx**: 80 (HTTP) + 443 (HTTPS) - shared mit allen anderen Services
- **Lieferfly**: 3010/3011 (NICHT berühren!)

### Troubleshooting

```bash
# Nginx-Logs prüfen
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log

# Nginx-Konfiguration testen
nginx -t

# Nginx neu starten (falls nötig)
systemctl restart nginx

# Offene Ports prüfen
netstat -tlnp | grep nginx
```

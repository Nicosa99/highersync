# HigherSync Quick Start Guide

## 🚀 Schnellstart in 5 Minuten

### 1. Code vorbereiten
```bash
cd /root/highersync

# Falls Code noch nicht vorhanden:
git clone <your-repo-url> .

# Dependencies installieren
npm install

# Environment-Variablen
cp .env.example .env
nano .env  # Anpassen!
```

### 2. Datenbank starten
```bash
# docker-compose.yml für Datenbank erstellen
cat > docker-compose.yml << 'DOCKER'
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: highersync-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: highersync
      POSTGRES_USER: highersync_user
      POSTGRES_PASSWORD: CHANGE_THIS_PASSWORD
    ports:
      - "5434:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
DOCKER

# Starten
docker-compose up -d
docker ps  # Prüfen ob läuft
```

### 3. Build & Migration
```bash
# Anwendung bauen
npm run build

# Datenbank migrieren (falls Prisma/TypeORM)
npm run migrate
# oder
npx prisma migrate deploy
```

### 4. PM2 konfigurieren
```bash
# ecosystem.config.js erstellen
cat > ecosystem.config.js << 'PM2'
module.exports = {
  apps: [
    {
      name: 'highersync-app',
      script: './dist/main.js',  // Anpassen je nach Framework
      instances: 1,
      env: {
        NODE_ENV: 'production',
        PORT: 3012
      }
    }
  ]
};
PM2

# App starten
pm2 start ecosystem.config.js
pm2 save
pm2 startup  # Befehl ausführen der angezeigt wird
```

### 5. Nginx konfigurieren
```bash
# Nginx-Config erstellen
cat > /etc/nginx/sites-available/highersync.com << 'NGINX'
server {
    listen 80;
    server_name highersync.com www.highersync.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
    }
    
    location / {
        proxy_pass http://localhost:3012;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
NGINX

# Aktivieren
ln -s /etc/nginx/sites-available/highersync.com /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### 6. SSL einrichten
```bash
# Let's Encrypt Zertifikat
certbot certonly --webroot -w /var/www/letsencrypt -d highersync.com -d www.highersync.com

# Nginx-Config für HTTPS erweitern
nano /etc/nginx/sites-available/highersync.com
# SSL-Zeilen hinzufügen (siehe NGINX-SETUP.md)

nginx -t
systemctl reload nginx
```

### 7. Fertig! 🎉
```bash
# Status prüfen
pm2 status
docker ps
systemctl status nginx

# Logs
pm2 logs
docker-compose logs -f

# Testen
curl https://highersync.com
```

## Troubleshooting

### App startet nicht
```bash
pm2 logs  # Fehler prüfen
pm2 restart all
```

### Datenbank-Verbindung fehlgeschlagen
```bash
docker ps  # Läuft Container?
docker logs highersync-postgres
# DATABASE_URL in .env prüfen
```

### Nginx 502 Bad Gateway
```bash
# App läuft?
pm2 status

# Port korrekt?
netstat -tlnp | grep 3012

# Nginx-Logs
tail -f /var/log/nginx/error.log
```

### Port bereits belegt
```bash
# Welcher Prozess nutzt Port?
lsof -i :3012

# Anderen Port wählen
nano .env  # PORT ändern
nano ecosystem.config.js  # PORT ändern
pm2 restart all
```

## Checkliste ✓

- [ ] Code hochgeladen/geklont
- [ ] `.env` konfiguriert
- [ ] Dependencies installiert (`npm install`)
- [ ] Build erstellt (`npm run build`)
- [ ] Datenbank läuft (`docker ps`)
- [ ] Migrations ausgeführt
- [ ] PM2 läuft (`pm2 status`)
- [ ] Nginx konfiguriert
- [ ] SSL eingerichtet
- [ ] App erreichbar via HTTPS
- [ ] **Lieferfly läuft weiter** (Port 3010/3011)

## Nächste Schritte

1. Backup-Cronjob einrichten (siehe CHECKLIST.md)
2. Monitoring aktivieren
3. Performance optimieren

Mehr Details in den anderen Docs! 📚

# Deployment-Optionen für HigherSync

## Option 1: PM2 Process Manager (wie Lieferfly)

### Vorteile
- Einfaches Setup
- Automatischer Neustart bei Crashes
- Log-Management
- Monitoring
- Gleiche Methode wie Lieferfly

### Nachteile
- Läuft direkt auf Host-System
- Weniger Isolation
- Dependency-Management manuell

### Setup

```bash
# PM2 installieren (falls nicht vorhanden)
npm install -g pm2

# App starten
cd /root/highersync
pm2 start npm --name "highersync-backend" -- run start:backend
pm2 start npm --name "highersync-frontend" -- run start:frontend

# Oder mit ecosystem.config.js
pm2 start ecosystem.config.js

# PM2 Konfiguration speichern
pm2 save

# PM2 beim Systemstart aktivieren
pm2 startup
```

### ecosystem.config.js Beispiel

Erstelle: `/root/highersync/ecosystem.config.js`

```javascript
module.exports = {
  apps: [
    {
      name: 'highersync-backend',
      script: './backend/dist/main.js',
      instances: 1,
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3012,
        DATABASE_URL: 'postgresql://user:pass@localhost:5434/highersync'
      },
      error_file: './logs/backend-error.log',
      out_file: './logs/backend-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      merge_logs: true
    },
    {
      name: 'highersync-frontend',
      script: 'node_modules/next/dist/bin/next',
      args: 'start -p 3013',
      instances: 1,
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3013,
        NEXT_PUBLIC_API_URL: 'https://highersync.com/api'
      },
      error_file: './logs/frontend-error.log',
      out_file: './logs/frontend-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      merge_logs: true
    }
  ]
};
```

### PM2 Befehle

```bash
# Status prüfen
pm2 status
pm2 list

# Logs anzeigen
pm2 logs highersync-backend
pm2 logs highersync-frontend
pm2 logs --lines 100

# App neu starten
pm2 restart highersync-backend
pm2 restart highersync-frontend
pm2 restart all

# App stoppen
pm2 stop highersync-backend
pm2 stop highersync-frontend

# App löschen
pm2 delete highersync-backend
pm2 delete highersync-frontend

# Monitoring
pm2 monit
```

## Option 2: Docker Container

### Vorteile
- Vollständige Isolation
- Reproduzierbare Umgebung
- Einfaches Deployment
- Kein Konflikt mit Host-Dependencies

### Nachteile
- Etwas komplexeres Setup
- Mehr Ressourcenverbrauch

### Docker Compose Setup

Erstelle: `/root/highersync/docker-compose.yml`

```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: highersync-backend
    restart: unless-stopped
    ports:
      - "3012:3012"
    environment:
      NODE_ENV: production
      PORT: 3012
      DATABASE_URL: postgresql://highersync_user:password@postgres:5432/highersync
    depends_on:
      - postgres
    networks:
      - highersync-network
    volumes:
      - ./backend/uploads:/app/uploads
      - ./backend/logs:/app/logs

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: highersync-frontend
    restart: unless-stopped
    ports:
      - "3013:3013"
    environment:
      NODE_ENV: production
      PORT: 3013
      NEXT_PUBLIC_API_URL: https://highersync.com/api
    depends_on:
      - backend
    networks:
      - highersync-network

  postgres:
    image: postgres:15-alpine
    container_name: highersync-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: highersync
      POSTGRES_USER: highersync_user
      POSTGRES_PASSWORD: SICHERES_PASSWORT
    ports:
      - "5434:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - highersync-network

volumes:
  postgres-data:

networks:
  highersync-network:
    driver: bridge
```

### Dockerfile für Backend (Beispiel)

Erstelle: `/root/highersync/backend/Dockerfile`

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

EXPOSE 3012

CMD ["node", "dist/main.js"]
```

### Docker Befehle

```bash
# Container starten
cd /root/highersync
docker-compose up -d

# Logs anzeigen
docker-compose logs -f
docker-compose logs -f backend
docker-compose logs -f frontend

# Container neu starten
docker-compose restart

# Container stoppen
docker-compose down

# Container mit rebuild
docker-compose up -d --build

# Status prüfen
docker-compose ps
```

## Option 3: Hybrid (Docker für DB, PM2 für App)

### Empfohlen für viele Use-Cases

```yaml
# docker-compose.yml (nur Datenbank)
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: highersync-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: highersync
      POSTGRES_USER: highersync_user
      POSTGRES_PASSWORD: SICHERES_PASSWORT
    ports:
      - "5434:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

```bash
# Datenbank in Docker
docker-compose up -d

# App mit PM2
pm2 start ecosystem.config.js
```

## Empfehlung

### Für Entwicklung/Testing
- **PM2** (Option 1) - schnell und einfach

### Für Produktion
- **Hybrid** (Option 3) - beste Balance zwischen Isolation und Performance
  - Datenbank in Docker (isoliert, einfaches Backup)
  - App mit PM2 (wie Lieferfly, bewährte Methode)

### Für Multi-Environment
- **Full Docker** (Option 2) - maximale Isolation und Portabilität

## Deployment-Checklist

```bash
# 1. Verzeichnis vorbereiten
cd /root/highersync
mkdir -p logs backups uploads

# 2. Dependencies installieren
npm install

# 3. Build erstellen
npm run build

# 4. Umgebungsvariablen setzen
cp .env.example .env
# .env bearbeiten

# 5. Datenbank starten
docker-compose up -d postgres

# 6. App starten
pm2 start ecosystem.config.js

# 7. PM2 speichern
pm2 save

# 8. Nginx konfigurieren (siehe NGINX-SETUP.md)
# 9. SSL einrichten (siehe NGINX-SETUP.md)
# 10. Testen!
```

## Monitoring & Logs

```bash
# PM2 Logs
pm2 logs
pm2 monit

# Docker Logs
docker-compose logs -f

# Nginx Logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# System Resources
htop
docker stats
```

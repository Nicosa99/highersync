# Datenbank-Setup für HigherSync

## Option 1: Eigener PostgreSQL Docker Container (Empfohlen)

### Vorteile
- Komplett isoliert von Lieferfly
- Eigene Version und Konfiguration
- Einfaches Backup und Migration
- Keine Konflikte möglich

### Docker Compose Setup

Erstelle: `/root/highersync/docker-compose.yml`

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: highersync-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: highersync
      POSTGRES_USER: highersync_user
      POSTGRES_PASSWORD: DEIN_SICHERES_PASSWORT_HIER
    ports:
      - "5434:5432"  # Port 5434 auf Host, 5432 im Container
    volumes:
      - highersync-postgres-data:/var/lib/postgresql/data
    networks:
      - highersync-network

volumes:
  highersync-postgres-data:
    driver: local

networks:
  highersync-network:
    driver: bridge
```

### Container starten

```bash
cd /root/highersync
docker-compose up -d

# Logs prüfen
docker-compose logs -f postgres

# Container-Status prüfen
docker ps | grep highersync
```

### Verbindung zur Datenbank

```bash
# Connection String für die App
DATABASE_URL=postgresql://highersync_user:DEIN_PASSWORT@localhost:5434/highersync

# Direkt verbinden (zum Testen)
docker exec -it highersync-postgres psql -U highersync_user -d highersync
```

## Option 2: Shared PostgreSQL (Host-Installation)

### Vorteile
- Nutzt vorhandene PostgreSQL-Installation (Port 5432)
- Keine zusätzlichen Container

### Nachteile
- Shared mit anderen Projekten
- Weniger Isolation

### Neue Datenbank erstellen

```bash
# Als postgres User einloggen
sudo -u postgres psql

# Innerhalb von psql:
CREATE DATABASE highersync;
CREATE USER highersync_user WITH PASSWORD 'DEIN_SICHERES_PASSWORT';
GRANT ALL PRIVILEGES ON DATABASE highersync TO highersync_user;
\q
```

### Verbindung zur Datenbank

```bash
# Connection String
DATABASE_URL=postgresql://highersync_user:DEIN_PASSWORT@localhost:5432/highersync
```

## Option 3: Verwenden von Lieferfly's PostgreSQL Container

### ⚠️ NICHT EMPFOHLEN für Produktion

Falls nur zum Testen:

```bash
# Lieferfly's PostgreSQL läuft auf Port 5433
docker exec -it lieferfly-postgres psql -U user -d postgres

# Neue Datenbank erstellen
CREATE DATABASE highersync;
CREATE USER highersync_user WITH PASSWORD 'passwort';
GRANT ALL PRIVILEGES ON DATABASE highersync TO highersync_user;
\q
```

**Connection String:**
```bash
DATABASE_URL=postgresql://highersync_user:passwort@localhost:5433/highersync
```

**Warum nicht empfohlen:**
- Lieferfly und HigherSync teilen sich Container
- Bei Container-Neustart können beide betroffen sein
- Schwierigeres Backup-Management
- Potenzielle Ressourcen-Konflikte

## Empfohlenes Setup

**Für Produktion:** Option 1 (Eigener Docker Container)
- Port 5434 verwenden
- Komplett isoliert
- Eigenes Backup-System

## Backup-Strategie

### Für Docker Container (Option 1)

```bash
# Backup erstellen
docker exec highersync-postgres pg_dump -U highersync_user highersync > /root/highersync/backups/backup_$(date +%Y%m%d_%H%M%S).sql

# Backup wiederherstellen
docker exec -i highersync-postgres psql -U highersync_user highersync < /root/highersync/backups/backup_20260415_120000.sql
```

### Automatisches Backup (Cronjob)

```bash
# Backup-Ordner erstellen
mkdir -p /root/highersync/backups

# Cronjob hinzufügen
crontab -e

# Täglich um 3 Uhr morgens
0 3 * * * docker exec highersync-postgres pg_dump -U highersync_user highersync > /root/highersync/backups/backup_$(date +\%Y\%m\%d).sql

# Alte Backups löschen (älter als 30 Tage)
0 4 * * * find /root/highersync/backups -name "backup_*.sql" -mtime +30 -delete
```

## Wichtige Hinweise

### Lieferfly-Datenbank
- **Container**: lieferfly-postgres
- **Port**: 5433
- **Datenbank**: lieferfly
- **NICHT berühren oder ändern!**

### HigherSync-Datenbank (empfohlen)
- **Container**: highersync-postgres
- **Port**: 5434
- **Datenbank**: highersync
- **Komplett isoliert von Lieferfly**

### Sicherheit
- Verwende starke Passwörter
- Ändere Standard-Credentials
- Beschränke Datenbankzugriff auf localhost (außer bei externer Verbindung nötig)

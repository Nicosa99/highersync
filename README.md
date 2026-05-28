# HigherSync

## Quick Start

### Dokumentation
Alle wichtigen Informationen befinden sich im `/docs` Verzeichnis:

- **[DEPLOYMENT.md](docs/DEPLOYMENT.md)** - Port-Übersicht und Deployment-Grundlagen
- **[NGINX-SETUP.md](docs/NGINX-SETUP.md)** - Nginx Reverse Proxy Konfiguration & SSL
- **[DATABASE-SETUP.md](docs/DATABASE-SETUP.md)** - Datenbank-Optionen und Setup
- **[DEPLOYMENT-OPTIONS.md](docs/DEPLOYMENT-OPTIONS.md)** - PM2, Docker, Hybrid-Optionen
- **[CHECKLIST.md](docs/CHECKLIST.md)** - Vollständige Deployment-Checkliste

## Wichtige Informationen

### Ports (NICHT mit Lieferfly überschneiden!)
- **Backend**: Port 3012 (empfohlen)
- **Frontend**: Port 3013 (empfohlen)
- **Datenbank**: Port 5434 (empfohlen)

### Lieferfly läuft auf
- **Backend**: Port 3010 ⚠️ NICHT verwenden!
- **Frontend**: Port 3011 ⚠️ NICHT verwenden!
- **Datenbank**: Port 5433 ⚠️ Separate DB empfohlen!

## Verzeichnisstruktur

```
/root/highersync/
├── backend/          # Backend-Code
├── frontend/         # Frontend-Code
├── docs/             # Dokumentation
├── logs/             # Anwendungs-Logs
├── backups/          # Datenbank-Backups
├── uploads/          # User-Uploads (falls vorhanden)
├── docker-compose.yml
├── ecosystem.config.js
├── .env
└── README.md
```

## Schnell-Commands

### PM2 (Process Management)
```bash
pm2 status                    # Status aller Apps
pm2 logs                      # Logs anzeigen
pm2 restart highersync-backend
pm2 restart highersync-frontend
pm2 monit                     # Monitoring
```

### Docker (Datenbank)
```bash
docker-compose up -d          # Container starten
docker-compose logs -f        # Logs anzeigen
docker-compose restart        # Neustart
docker ps | grep highersync   # Status
```

### Nginx
```bash
nginx -t                      # Konfiguration testen
systemctl reload nginx        # Neu laden
tail -f /var/log/nginx/error.log  # Error-Logs
```

### Datenbank
```bash
# Verbinden
docker exec -it highersync-postgres psql -U highersync_user -d highersync

# Backup erstellen
docker exec highersync-postgres pg_dump -U highersync_user highersync > backups/backup_$(date +%Y%m%d).sql

# Backup wiederherstellen
docker exec -i highersync-postgres psql -U highersync_user highersync < backups/backup_20260415.sql
```

## Setup-Reihenfolge

1. ✅ Ports festlegen (siehe DEPLOYMENT.md)
2. ✅ Code hochladen/klonen
3. ✅ Dependencies installieren (`npm install`)
4. ✅ `.env` Datei konfigurieren
5. ✅ Build erstellen (`npm run build`)
6. ✅ Datenbank starten (siehe DATABASE-SETUP.md)
7. ✅ PM2 konfigurieren und starten (siehe DEPLOYMENT-OPTIONS.md)
8. ✅ Nginx einrichten (siehe NGINX-SETUP.md)
9. ✅ SSL-Zertifikat generieren (siehe NGINX-SETUP.md)
10. ✅ Testen!

## Isolation von Lieferfly

HigherSync und Lieferfly sind **komplett getrennt**:
- Verschiedene Ports
- Separate Nginx-Konfigurationen
- Eigene Datenbank-Container (empfohlen)
- Separate PM2-Prozesse
- **Keine Überschneidungen oder Konflikte**

## Support & Troubleshooting

Bei Problemen:
1. Logs prüfen: `pm2 logs`, `docker-compose logs -f`
2. Nginx-Logs prüfen: `tail -f /var/log/nginx/error.log`
3. Port-Konflikte prüfen: `netstat -tlnp`
4. Dokumentation im `/docs` Verzeichnis konsultieren

## Wichtige Dateien

- `.env` - Umgebungsvariablen (NICHT committen!)
- `ecosystem.config.js` - PM2-Konfiguration
- `docker-compose.yml` - Docker-Services
- `/etc/nginx/sites-available/highersync.com` - Nginx-Config

---

**Status**: In Entwicklung
**Letzte Aktualisierung**: 2026-04-15

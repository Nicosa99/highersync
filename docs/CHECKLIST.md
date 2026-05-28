# HigherSync Deployment Checklist

## Pre-Deployment

### 1. Server-Vorbereitung
- [ ] Server-Zugriff via SSH verifizieren
- [ ] Root-Rechte bestätigt
- [ ] Verfügbare Ressourcen prüfen (RAM, CPU, Disk)
  ```bash
  free -h
  df -h
  htop
  ```

### 2. Port-Auswahl
- [ ] Freie Ports identifiziert:
  - Backend: Port _____ (empfohlen: 3012)
  - Frontend: Port _____ (empfohlen: 3013)
  - Datenbank: Port _____ (empfohlen: 5434)
- [ ] Ports mit `netstat -tlnp` geprüft
- [ ] **Bestätigt: Ports 3010/3011 NICHT verwendet** (Lieferfly)

### 3. Domain-Setup
- [ ] Domain gekauft/verfügbar
- [ ] DNS A-Record auf Server-IP gesetzt
- [ ] DNS-Propagation abgeschlossen (24-48h)
  ```bash
  nslookup deine-domain.com
  dig deine-domain.com
  ```

## Application Setup

### 4. Verzeichnisstruktur
- [ ] Ordner `/root/highersync` erstellt
- [ ] Unterordner angelegt:
  ```bash
  mkdir -p /root/highersync/{backend,frontend,docs,logs,backups,uploads}
  ```
- [ ] Code hochgeladen/geklont
- [ ] Dependencies installiert
  ```bash
  cd /root/highersync
  npm install
  ```

### 5. Umgebungsvariablen
- [ ] `.env` Datei erstellt
- [ ] Alle erforderlichen Variablen gesetzt:
  - [ ] `NODE_ENV=production`
  - [ ] `PORT=3012` (Backend)
  - [ ] `FRONTEND_PORT=3013`
  - [ ] `DATABASE_URL=postgresql://...`
  - [ ] API Keys, Secrets, etc.
- [ ] `.env` in `.gitignore` eingetragen

### 6. Build-Prozess
- [ ] TypeScript kompiliert (falls verwendet)
  ```bash
  npm run build
  ```
- [ ] Build-Fehler behoben
- [ ] Dist/Build-Ordner vorhanden

## Datenbank-Setup

### 7. PostgreSQL Datenbank
- [ ] Deployment-Option gewählt:
  - [ ] Option 1: Eigener Docker Container (empfohlen)
  - [ ] Option 2: Shared PostgreSQL (Host)
  - [ ] Option 3: Lieferfly Container (nur Testing!)

**Falls Option 1 (Docker Container):**
- [ ] `docker-compose.yml` erstellt
- [ ] Container gestartet
  ```bash
  cd /root/highersync
  docker-compose up -d postgres
  ```
- [ ] Container läuft
  ```bash
  docker ps | grep highersync-postgres
  ```
- [ ] Verbindung getestet
  ```bash
  docker exec -it highersync-postgres psql -U highersync_user -d highersync
  ```

### 8. Datenbank-Migration
- [ ] Migrations ausgeführt
  ```bash
  npm run migrate
  # oder
  npx prisma migrate deploy
  ```
- [ ] Seed-Daten importiert (falls nötig)
- [ ] Tabellen vorhanden geprüft

## Process Management

### 9. PM2 Setup
- [ ] PM2 installiert
  ```bash
  npm install -g pm2
  ```
- [ ] `ecosystem.config.js` erstellt und konfiguriert
- [ ] Apps gestartet
  ```bash
  pm2 start ecosystem.config.js
  ```
- [ ] Status geprüft
  ```bash
  pm2 status
  pm2 logs
  ```
- [ ] PM2 gespeichert
  ```bash
  pm2 save
  ```
- [ ] PM2 Startup konfiguriert
  ```bash
  pm2 startup
  # Befehl ausführen, der angezeigt wird
  ```

## Nginx & SSL

### 10. Nginx-Konfiguration
- [ ] Nginx-Konfigurationsdatei erstellt
  ```bash
  nano /etc/nginx/sites-available/highersync.com
  ```
- [ ] Konfiguration von `NGINX-SETUP.md` kopiert
- [ ] Ports angepasst (3012, 3013)
- [ ] Symlink erstellt
  ```bash
  ln -s /etc/nginx/sites-available/highersync.com /etc/nginx/sites-enabled/
  ```
- [ ] Nginx-Konfiguration getestet
  ```bash
  nginx -t
  ```
- [ ] Nginx neu geladen
  ```bash
  systemctl reload nginx
  ```

### 11. SSL-Zertifikat (Let's Encrypt)
- [ ] Certbot installiert
  ```bash
  apt install certbot python3-certbot-nginx
  ```
- [ ] Domain erreichbar via HTTP
- [ ] SSL-Zertifikat generiert
  ```bash
  certbot certonly --webroot -w /var/www/letsencrypt -d highersync.com
  ```
- [ ] Zertifikat-Pfade in Nginx eingetragen
- [ ] Nginx-Konfiguration getestet
  ```bash
  nginx -t
  ```
- [ ] Nginx neu geladen
  ```bash
  systemctl reload nginx
  ```
- [ ] HTTPS funktioniert (Browser-Test)
- [ ] Auto-Renewal getestet
  ```bash
  certbot renew --dry-run
  ```

## Testing & Verification

### 12. Funktionalität testen
- [ ] Frontend erreichbar via HTTPS
- [ ] Backend/API antwortet
- [ ] Datenbank-Verbindung funktioniert
- [ ] Uploads funktionieren (falls vorhanden)
- [ ] User-Registrierung/-Login testen
- [ ] Haupt-Features testen

### 13. Logs & Monitoring
- [ ] PM2 Logs prüfen
  ```bash
  pm2 logs --lines 50
  ```
- [ ] Nginx Access Logs prüfen
  ```bash
  tail -f /var/log/nginx/access.log
  ```
- [ ] Nginx Error Logs prüfen
  ```bash
  tail -f /var/log/nginx/error.log
  ```
- [ ] Keine kritischen Fehler

### 14. Performance & Security
- [ ] SSL-Rating testen (ssllabs.com/ssltest)
- [ ] HTTPS erzwungen (HTTP -> HTTPS Redirect)
- [ ] Upload-Limits konfiguriert
- [ ] Rate-Limiting erwogen (falls nötig)
- [ ] Firewall-Regeln geprüft
  ```bash
  ufw status
  ```

## Backup & Maintenance

### 15. Backup-System
- [ ] Backup-Verzeichnis erstellt
  ```bash
  mkdir -p /root/highersync/backups
  ```
- [ ] Backup-Script erstellt/getestet
- [ ] Cronjob für automatische Backups eingerichtet
  ```bash
  crontab -e
  # 0 3 * * * docker exec highersync-postgres pg_dump ...
  ```
- [ ] Restore-Prozess getestet

### 16. Monitoring & Alerts
- [ ] PM2 Monitoring aktiviert
  ```bash
  pm2 monit
  ```
- [ ] Uptime-Monitoring konfiguriert (optional)
- [ ] Disk-Space Monitoring (optional)
- [ ] Error-Alerts konfiguriert (optional)

## Final Checks

### 17. Lieferfly-Isolation verifizieren
- [ ] **KRITISCH**: Lieferfly läuft weiterhin auf Ports 3010/3011
  ```bash
  netstat -tlnp | grep -E ":(3010|3011)"
  ```
- [ ] Lieferfly Frontend erreichbar
- [ ] Lieferfly Backend antwortet
- [ ] Keine Konflikte zwischen HigherSync und Lieferfly
- [ ] Separate Nginx-Konfigurationen bestätigt
  ```bash
  ls -la /etc/nginx/sites-enabled/
  ```

### 18. Dokumentation
- [ ] `.env.example` erstellt
- [ ] README.md aktualisiert
- [ ] Deployment-Prozess dokumentiert
- [ ] Wichtige Befehle dokumentiert
- [ ] Kontaktinformationen/Support dokumentiert

### 19. Clean-Up
- [ ] Unnötige Dateien entfernt
- [ ] `.git` Ordner geprüft (keine Secrets committet)
- [ ] Temporäre Dateien gelöscht
- [ ] Berechtigungen geprüft

## Go-Live

### 20. Production-Ready
- [ ] Alle oben genannten Checks erfolgreich
- [ ] Keine kritischen TODOs offen
- [ ] Team informiert
- [ ] Rollback-Plan erstellt
- [ ] 🚀 **LIVE!**

## Post-Deployment

### 21. Monitoring (erste 24h)
- [ ] Logs überwachen
- [ ] Performance überwachen
- [ ] Fehlerrate überwachen
- [ ] User-Feedback sammeln

### 22. Optimierung
- [ ] Performance-Bottlenecks identifizieren
- [ ] Cache-Strategie optimieren
- [ ] Datenbank-Queries optimieren
- [ ] CDN erwägen (falls nötig)

---

## Quick Reference Commands

```bash
# Status-Check
pm2 status
docker ps
systemctl status nginx
netstat -tlnp

# Logs
pm2 logs
docker-compose logs -f
tail -f /var/log/nginx/error.log

# Restart
pm2 restart all
docker-compose restart
systemctl reload nginx

# Backup
docker exec highersync-postgres pg_dump -U highersync_user highersync > backup.sql
```

## Emergency Contacts

- Server-Admin: _________________
- Domain-Registrar: _________________
- Support: _________________

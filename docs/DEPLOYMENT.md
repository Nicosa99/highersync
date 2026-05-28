# HigherSync Deployment Guide

## Aktuelles Server-Setup

### Belegte Ports (NICHT verwenden!)
- **3000**: Docker (open-webui)
- **3002**: Node.js Service
- **3003**: Next.js Server
- **3010**: Lieferfly Backend API
- **3011**: Lieferfly Frontend
- **5432**: PostgreSQL (Host)
- **5433**: PostgreSQL (Docker - Lieferfly)
- **5678**: n8n (Docker)
- **8000, 9443**: Portainer (Docker)
- **8080**: PHP Apache (webp-api)
- **8200**: GPT OSS API (Docker)

### Freie Ports für HigherSync
Empfohlene Ports:
- **3012**: HigherSync Backend/API
- **3013**: HigherSync Frontend
- **5434**: PostgreSQL (falls eigene DB benötigt)

## Port-Konfiguration

### Option 1: Monolithische App (1 Port)
```bash
PORT=3012
```

### Option 2: Separate Backend/Frontend (2 Ports)
```bash
BACKEND_PORT=3012
FRONTEND_PORT=3013
```

## Wichtige Hinweise

- **Lieferfly läuft auf Ports 3010/3011** - Diese NICHT verwenden!
- Alle Änderungen an HigherSync haben KEINE Auswirkungen auf Lieferfly
- Bei Problemen: Nginx-Konfigurationen sind getrennt

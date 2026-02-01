<!-- TOC -->

- [n8n Migration: SQLite → PostgreSQL](#n8n-migration-sqlite--postgresql)
  - [Ziel](#ziel)
  - [Wichtige Hinweise (vorher)](#wichtige-hinweise-vorher)
    - [Backup (SQLite)](#backup-sqlite)
    - [Export aus SQLite](#export-aus-sqlite)
    - [PostgreSQL in Docker Compose hinzufügen](#postgresql-in-docker-compose-hinzufügen)
    - [Starten](#starten)
    - [Import nach PostgreSQL](#import-nach-postgresql)
    - [Warum n8n danach einen User will](#warum-n8n-danach-einen-user-will)

<!-- /TOC -->
# n8n Migration: SQLite → PostgreSQL

## Ziel
n8n lief bisher mit SQLite (Datei `database.sqlite` im `.n8n` Volume) und wird auf PostgreSQL umgestellt, damit die Datenbank stabiler skalieren kann.

## Wichtige Hinweise (vorher)
- n8n **stoppen**, bevor du ein Backup/Export machst (damit SQLite nicht währenddessen schreibt).
- Den **gleichen** `N8N_ENCRYPTION_KEY` weiterverwenden, sonst können Credentials nach dem Import nicht korrekt entschlüsselt werden.

### Backup (SQLite)
- Backup des gemounteten n8n-Datenordners erstellen (bei dir: `/home/n8n-data/n8ndata` → `/home/node/.n8n`).

Beispiel:
```bash
sudo tar -czf n8n-n8ndata-backup.tgz -C /home/n8n-data n8ndata
```

### Export aus SQLite
Empfohlen: **Entities-Export** (enthält mehr als nur Workflows/Credentials).

```bash
docker exec -u node -it n8n-main mkdir -p /home/node/Work/n8n-backup
docker exec -u node -it n8n-main n8n export:entities --all --output=/home/node/Work/n8n-backup/entities.json
```

Alternativ (nur workflows und credentials):
```bash
docker exec -u node -it n8n-main n8n export:workflow --all --output=/home/node/Work/n8n-backup/workflows.json
docker exec -u node -it n8n-main n8n export:credentials --all --output=/home/node/Work/n8n-backup/credentials.json
```
```bash
Successfully exported 21 workflows.
Successfully exported 10 credentials.

```
### PostgreSQL in Docker Compose hinzufügen
- Neuen `postgres` Service ergänzen und `n8n-main` auf Postgres umstellen über `DB_TYPE=postgresdb` + `DB_POSTGRESDB_*` Variablen.

Minimal (Konzept):
```yaml
services:
  postgres:
    image: postgres:17
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=*** 
      - POSTGRES_DB=n8n
    volumes:
      - /home/n8n-data/postgres-data:/var/lib/postgresql/data

  n8n-main:
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=***
```

### Starten
- Stack deployen

### Import nach PostgreSQL
Wenn du **Entities** exportiert hast:
```bash
docker exec -u node -it n8n-main n8n import:entities --input=/home/node/Work/n8n-backup/entities.json --truncateTables
```

Wenn du **Workflows + Credentials** separat exportiert hast: zuerst Credentials, dann Workflows.
```bash
docker exec -u node -it n8n-main n8n import:credentials --input=/home/node/Work/n8n-backup/credentials.json
docker exec -u node -it n8n-main n8n import:workflow --input=/home/node/Work/n8n-backup/workflows.json
```
```bash

Successfully imported 10 credentials.
Importing 21 workflows...
Successfully imported 21 workflows.

```
### Warum n8n danach einen User will
Wenn nur Workflows/Credentials importiert wurden, fehlen User-/Setup-Daten, daher fordert n8n beim ersten UI-Login die Owner-Erstellung.


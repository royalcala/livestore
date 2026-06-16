Basado en la interfaz `SyncBackend`, el servidor solo necesita dos operaciones:

```
push → validar orden → persistir atómicamente → actualizar cursor (1 a la vez)
pull  → validar cursor → query eventos siguientes → devolver (paralelizable)
```

Esencialmente es un **log de eventos append-only** con un cursor. Muy simple. Estas son las opciones self-hosted que consideraría, de más simple a más compleja:

---

### 1. SQLite + WebSocket (la más simple)

Un solo proceso, un solo archivo. Ideal para apps con <1000 usuarios concurrentes.

```
Browser ←WS→ Tu server (Node/Bun/Deno) → SQLite
```

- Bun con `Bun.serve` + WebSocket nativo + `bun:sqlite`: un solo binario, deploy trivial
- Las escrituras son seriales (SQLite lo garantiza con WAL mode), encaja perfecto con el requisito de "one push at a time"
- El pull con `live: true` se implementa con un `NOTIFY` o broadcast al WebSocket
- No necesitas Docker, Postgres, Redis, nada
- **Catch**: no escala horizontal. Para eso necesitas lo siguiente.

---

### 2. PostgreSQL + `LISTEN/NOTIFY`

```
Browser ←WS→ Tu server (Node/Go/Rust) → PostgreSQL
```

- Push: `INSERT` con `SERIAL` en la columna `seqNumGlobal`
- Pull: `SELECT * FROM events WHERE seqNumGlobal > $cursor ORDER BY seqNumGlobal`
- Live pull: `LISTEN events_channel` + trigger `NOTIFY` en cada insert → reenvías por WebSocket al cliente
- Escala mejor que SQLite (réplicas de lectura para pulls, writer único para pushes)
- Muchos ORMs/query builders ya lo soportan (Drizzle, Kysely, Prisma)

---

### 3. Turso / libsql (SQLite distribuido)

```
Browser ←HTTP/WS→ Tu server → Turso (libsql HTTP)
```

- Misma semántica SQLite pero con replicación automática a edge locations
- Turso ya expone HTTP API, simplifica el server
- Buena opción si quieres SQLite pero con alcance global
- Open source (libsql), puedes self-hostearlo

---

### 4. LiteFS / Litestream (SQLite replicado a S3)

```
Browser ←WS→ Node/Bun → SQLite con Litestream → S3/MinIO
```

- Misma simplicidad de SQLite pero con backups continuos a S3
- Litestream replica el WAL a S3 en tiempo real
- No da replicación de lectura, solo disaster recovery
- **Ideal si tu app es single-server y solo necesitas backups**

---

### 5. Go + BadgerDB / BoltDB

```
Browser ←WS→ Go server → BadgerDB/BoltDB (embedded)
```

- Go es excelente para servidores stateful con WebSocket
- BadgerDB es una LSM-tree embebida optimizada para escrituras secuenciales (eventos → perfecto)
- Un solo binario, sin dependencias externas
- Más performante que SQLite para este patrón append-only + cursor scan

---

### 6. NATS / JetStream (cola de eventos)

```
Browser ←WS→ Tu server → NATS JetStream
```

- NATS es un sistema de mensajería open source, JetStream agrega persistencia
- Los eventos se publican a un stream, el cursor es el sequence number de NATS
- NATS ya maneja la distribución en tiempo real (el pull `live: true` es gratis)
- Escala horizontalmente, escrito en Go
- **Overkill** para apps pequeñas, ideal para alta concurrencia

---

### Recomendación práctica

| Si tu caso es... | Recomendación |
|------------------|---------------|
| MVP, app pequeña, un solo server | **Bun + SQLite + WebSocket** |
| Necesitas réplicas de lectura | **PostgreSQL + LISTEN/NOTIFY** |
| Quieres SQLite pero multi-region | **Turso/libsql self-hosted** |
| Quieres Go y máxima performance | **Go + BadgerDB** |
| Alta concurrencia, muchos servers | **NATS JetStream** |

En todos los casos, la tabla de eventos es la misma:

```sql
CREATE TABLE eventlog (
  seq_num_global INTEGER NOT NULL,
  seq_num_client INTEGER NOT NULL,
  client_id TEXT NOT NULL,
  session_id TEXT NOT NULL,
  name TEXT NOT NULL,
  args_json TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (seq_num_global, seq_num_client)
);
```
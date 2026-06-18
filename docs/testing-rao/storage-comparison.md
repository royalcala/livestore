# LiveStore vs redb-direct vs SurrealDB vs TanStack DB vs Durable Streams — Análisis para Syntrix

## El problema

Syntrix necesita persistencia local + queries + sync P2P. Tenemos iroh-docs (redb) como source of truth para sync. ¿Qué capa de queries ponemos arriba?

## Las cinco opciones

### Opción A: LiveStore (event sourcing + WASM SQLite)

```
Frontend React → LiveStore hooks → SQLite WASM → iroh-docs (vía SyncBackend)
```

### Opción B: redb directo con capa de queries en Rust

```
Frontend React → invoke() → Rust → redb (via iroh-docs store)
```

### Opción C: SurrealDB embebido

```
Frontend React → invoke() → Rust → SurrealDB → iroh-docs (sync manual)
```

### Opción D: TanStack DB (con adapter custom a iroh-docs)

```
React UI → TanStack DB (collections, live queries, optimistic mutations)
                ↕ custom irohCollection adapter
           Rust/redb (iroh-docs) — persistencia + sync P2P
```

- **Madurez:** Beta (v0.1.87 en npm, ~1000 releases). 3.8k estrellas.
- **Dependencias:** `@tanstack/db`, `@tanstack/react-db`. ~50 KB total.
- **Curva de aprendizaje:** Baja. API tipo TanStack (selectores), schemas con Zod/Valibot/ArkType.
- **Sync:** VÍA ADAPTER CUSTOM. TanStack DB expone una interfaz `CollectionConfig` para implementar tu propio storage + sync. Existen adapters oficiales para ElectricSQL, RxDB, PowerSync, TrailBase. **Nosotros podemos implementar `irohCollectionOptions`.**
- **Offline-first:** Nativo con adapter custom (escribimos directo a redb, TanStack DB maneja el optimistic state).
- **Queries reactivas:** `useLiveQuery()` con sub-millisecond reactivity vía **differential dataflow** (d2ts, de ElectricSQL). Joins, where, orderBy, select.
- **Peso:** ~50 KB min + gzip.
- **Creador:** Tanner Linsley (TanStack Query, Router, Table, Start).

**Features que nos interesan:**

1. **Colecciones normalizadas** — cada collection es un set tipado de objetos con schema Zod/Valibot
2. **Relaciones** — joins entre collections en queries. `q.from({todo}).join({list}, eq(list.id, todo.listId))`
3. **Índices implícitos** — differential dataflow mantiene los resultados de queries incrementales. Update de 1 row en 100,000 toma ~0.7ms.
4. **Sync Modes** — eager (carga todo), on-demand (carga solo lo que la query pide), progressive (carga inmediato + sync background). Perfecto para namespaces grandes.
5. **Optimistic mutations** — `insert/update/delete` locales instantáneos, rollback si falla el handler.
6. **Schemas** — Standard Schema (Zod, Valibot, ArkType). Validación + transformación de tipos.
7. **Derived collections** — el resultado de una query ES otra collection que se puede querear.

**El adapter custom (`irohCollectionOptions`):**

```typescript
// Pattern A: User provides mutation handlers, nosotros implementamos sync
const invoicesCollection = createCollection(
  irohCollectionOptions({
    orgId: "org_abc",
    collection: "invoices",
    schema: invoiceSchema,
    getKey: (inv) => inv.id,
    // El sync function carga datos de iroh-docs y se subscribe a cambios
    // onInsert/onUpdate/onDelete escriben a iroh-docs vía Tauri invoke
  })
)
```

La función `sync` usa el patrón `begin() / write() / commit() / markReady()`:
1. `begin()` — inicia transacción
2. `write({ type: 'insert', value: item })` — inserta cada entry
3. `commit()` — aplica cambios atómicamente
4. Subscripción a eventos de iroh-docs para tiempo real
5. `markReady()` — señala que la carga inicial terminó

**Namespaces (por usuario/org):**

Cada namespace se implementa como una instancia separada de TanStack DB o con prefijos:

```
Org X namespace:
  invoicesCollection  ← datos de invoices en org X
  productsCollection  ← datos de productos en org X
  customersCollection ← datos de clientes en org X

Org Y namespace:
  invoicesCollection  ← aislado de org X
  productsCollection
  customersCollection
```

La **fusión** ocurre naturalmente: cuando el usuario cambia de org, las queries se ejecutan contra ese namespace. iroh-docs ya aísla los docs por `org_id`.

**Pros:**
- Muy liviano (~50 KB, sin WASM)
- API probada (estilo TanStack, miles de devs)
- Colecciones con relaciones e índices (vía differential dataflow)
- Live queries sub-millisecond
- Optimistic mutations con rollback
- Schema validation (Zod/Valibot)
- Custom adapter bien documentado (WebSocket example completo en docs)
- Sync modes (eager/on-demand/progressive) ideales para namespaces
- Sin Effect-TS, sin WASM, sin servidor
- Comunidad activa (Tanner Linsley + ElectricSQL partnership)

**Contras:**
- Beta (pero con ~1000 releases, activamente mantenido)
- El adapter custom hay que implementarlo (~200 líneas siguiendo el WebSocket example)
- No tiene persistencia on-disk propia — delegamos a redb
- La fusión de namespaces a nivel TanStack DB requiere queries que crucen orgs (poco común para nosotros)

### Opción E: Durable Streams

```
Frontend → @durable-streams/client → HTTP → durable-streams server → stream storage
```

- **Madurez:** Beta. 1.6k estrellas. Creado por el equipo de ElectricSQL.
- **Dependencias:** `@durable-streams/client` (TypeScript), `@durable-streams/state` (insert/update/delete sobre streams), `StreamDB` (DB reactiva sobre streams).
- **Curva de aprendizaje:** Media. Protocolo HTTP simple pero hay que entender offset-based streaming.
- **Sync:** NATIVO. Durable Streams está diseñado exactamente para sync: streams append-only, offset-based replay, live tailing, multi-device resume.
- **Offline-first:** Sí (con StreamDB). Escribís local, el stream sync cuando hay conexión.
- **Queries reactivas:** Sí (vía StreamDB — type-safe, reactive database in a stream).
- **Clientes:** TypeScript, Python, Go, Elixir, C#, Swift, PHP, Java, **Rust**, Ruby.
- **Creador:** ElectricSQL team (los de Postgres → P2P sync).

**Arquitectura de Durable Streams:**

```
Durable Streams = protocolo HTTP para streams append-only con replay.
  ├── @durable-streams/client   ← cliente para leer/escribir streams
  ├── @durable-streams/state    ← protocolo de state (insert/update/delete)
  └── StreamDB                  ← DB reactiva construida sobre streams

StreamDB ──escribe──► Durable Stream ──sync──► otros clientes
    │                                              │
  queries reactivas                          reciben updates
```

**Relación con LiveStore:**
- LiveStore ES el event store + materializer + SQL. Durable Streams es el **protocolo de transporte** para los eventos.
- Durable Streams + StreamDB = una alternativa a LiveStore pero más modular.
- ElectricSQL (los creadores de Durable Streams) usan Durable Streams como el protocolo de sync para Postgres → P2P.
- Durable Streams es a LiveStore lo que iroh-docs es a redb: el protocolo de sync.

**Pros:**
- Protocolo simple (HTTP, offset-based), bien documentado
- Cliente Rust disponible (podemos integrarlo en el backend Tauri)
- StreamDB da queries reactivas sobre streams
- Diseñado para exactamente nuestro caso (sync, offline, multi-device)
- Menos complejo que LiveStore (sin Effect-TS, sin WASM SQLite)
- Comunidad activa (ElectricSQL)

**Contras:**
- **Necesita un servidor de streams.** Durable Streams usa HTTP. No es P2P nativo. Necesitamos un servidor que hostee los streams.
- El servidor puede ser el Caddy plugin (production) o el Node.js server (dev).
- **StreamDB está en early stage.** Documentación limitada.
- **Duplicación:** si usamos StreamDB + iroh-docs, tendríamos dos stores (streams HTTP + redb P2P).
- **No es P2P nativo** — los streams viven en un servidor, no en los peers.

**¿Podríamos reemplazar iroh-docs con Durable Streams?**

No directamente. Durable Streams usa HTTP → servidor. iroh-docs es P2P sin servidor. Pero podríamos usar Durable Streams como el **SyncBackend de LiveStore** (en vez del adapter custom a iroh-docs). O al revés: usar iroh-docs como el storage de Durable Streams.

## Comparativa completa

| Dimensión | LiveStore | redb-direct | SurrealDB | TanStack DB + adapter | Durable Streams |
|-----------|-----------|-------------|-----------|-----------------------|-----------------|
| Peso | ~3MB WASM+JS | 0 KB | ~30MB | ~50 KB + adapter | ~100 KB + server |
| Dependencias | ~50 npm | 0 | 1 crate | 2 npm + adapter | 3 npm + server |
| Curva aprendizaje | Alta (Effect-TS) | Baja (Rust) | Media (SurrealQL) | Baja (TanStack) | Media (streams) |
| Sync P2P nativo | No (adapter) | ✅ (iroh-docs) | No (manual) | ✅ **vía adapter a iroh-docs** | No (HTTP server) |
| Offline-first | ✅ | ✅ | ✅ | ✅ (redb via adapter) | ✅ (StreamDB) |
| Queries SQL-like | ✅ (SQLite) | ❌ (KV only) | ✅ (SurrealQL) | ✅ (live queries + joins) | ✅ (StreamDB) |
| Relaciones (JOINs) | ✅ (SQL) | Manual | ✅ (grafos) | ✅ (join en queries) | Parcial |
| Índices | ✅ (SQL) | Manual (prefix scan) | ✅ | ✅ (differential dataflow) | Parcial |
| Reactividad | ✅ (useQuery) | Manual | ✅ (LIVE SELECT) | ✅ (useLiveQuery) | ✅ (StreamDB) |
| Optimistic mutations | ✅ | Manual | ✅ | ✅ (built-in + rollback) | Manual |
| Duplicación datos | SQLite + redb | Ninguna | SurrealDB + redb | Ninguna (redb es source) | StreamDB + redb |
| Schema validation | Effect-Schema | Manual (serde) | Schemaless | ✅ (Zod/Valibot/ArkType) | Manual |
| Mantenibilidad | Baja | Alta | Media | Alta | Media |
| Madurez | Beta | Stable | Stable | Beta (~1000 releases) | Beta |
| Necesita servidor | No | No | No | No | **Sí** |
| Namespace isolation | ✅ (storeId) | ✅ (doc prefix) | ✅ (NS/DB) | ✅ (instance/prefix) | ✅ (stream prefix) |

## Veredicto por caso de uso

### ¿Qué necesitamos realmente?

| Requisito | ¿Crítico? | Mejor opción |
|-----------|-----------|-------------|
| Sync P2P sin servidor | ✅ | redb-direct / TanStack DB |
| Relaciones entre datos | ✅ | TanStack DB / LiveStore |
| Queries reactivas en UI | ✅ | TanStack DB / LiveStore |
| Optimistic mutations | ✅ | TanStack DB |
| Schema validation | ✅ | TanStack DB (Zod) |
| Namespace isolation (por org) | ✅ | TanStack DB / redb-direct |
| Pesos livianos (Tauri mobile) | ✅ | redb-direct / TanStack DB |
| Baja complejidad | ✅ | redb-direct / TanStack DB |

### Recomendación final

**Fase 1 (MVP, AHORA): TanStack DB + adapter custom a iroh-docs.**

```
┌──────────────────────────────────────────────────────┐
│  React UI                                             │
│  useLiveQuery((q) => q.from({invoice})                │
│                    .join({customer}, eq(id, custId))   │
│                    .where(eq(invoice.status, 'open')))│
│  invoiceCollection.insert({...})  ← optimistic        │
├──────────────────────────────────────────────────────┤
│  TanStack DB                                          │
│  ┌────────────────────────────────────────────────┐  │
│  │  irohCollection adapter (~200 loc)              │  │
│  │  - sync(): carga entries de iroh-docs           │  │
│  │  - subscribe(): eventos de peers                │  │
│  │  - onInsert/onUpdate/onDelete → invoke()        │  │
│  │  - serialize/deserialize entries ↔ objects       │  │
│  └────────────────────────────────────────────────┘  │
│         ↓ write              ↑ read/sync              │
│  ┌────────────────────────────────────────────────┐  │
│  │  iroh-docs (redb)                               │  │
│  │  - org_<id>/data → entries firmadas por HLC     │  │
│  │  - sync P2P automático                          │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

**¿Por qué TanStack DB sobre redb-direct?**

redb-direct nos da persistencia, pero tenemos que implementar a mano:
- Relaciones entre colecciones
- Índices para queries rápidas
- Reactividad (eventos/polling)
- Optimistic state + rollback
- Schema validation
- Cache de queries

TanStack DB nos da todo eso **ya hecho**, probado por miles de devs. El adapter a iroh-docs son ~200 líneas siguiendo el WebSocket example de la doc oficial. Ganamos meses de desarrollo.

**¿Por qué TanStack DB sobre LiveStore?**

| | TanStack DB | LiveStore |
|---|---|---|
| Dependencias | 2 (~50 KB) | ~50 (~3 MB) |
| API | Selector-based (familiar) | Effect-TS (complejo) |
| Schemas | Zod/Valibot (ubicuo) | Effect-Schema (nicho) |
| Persistencia | Delega al adapter | SQLite WASM (duplicado) |
| Madurez | Beta con 1000 releases | Beta con releases esporádicos |
| Comunidad | TanStack (masiva) | Livestore (pequeña) |

TanStack DB es más simple, más liviano, más mantenible. Y el adapter a iroh-docs es el mismo trabajo que el adapter de LiveStore.

**Fase 2 (escalar):**

- **Durable Streams** si necesitamos sync con clientes mobile/web sin Tauri (HTTP-based)
- **LiveStore** solo si su API se estabiliza y necesitamos SQL real para analytics

**No usar:**
- **SurrealDB**: 30MB + sync manual, no se justifica
- **redb-direct solo**: nos deja sin queries, relaciones, ni reactividad (mucho trabajo manual)

## Relación entre Durable Streams y TanStack DB

Son **ortogonales**. No dependen uno del otro, pero se complementan:

| Capa | Durable Streams | TanStack DB |
|------|----------------|-------------|
| ¿Qué es? | Protocolo de transporte (HTTP streams) | Store local de datos |
| ¿Dónde vive? | Servidor HTTP | Cliente (navegador) |
| ¿Qué resuelve? | Sync, replay, resume | Queries, relaciones, optimistic state |
| ¿Pueden combinarse? | ✅ Un adapter de TanStack DB podría usar Durable Streams como sync engine | ✅ |

**Ejemplo de combinación:** ElectricSQL (creadores de Durable Streams) son sponsors de TanStack DB. Su adapter `@tanstack/electric-db-collection` integra ambos. Durable Streams mueve los datos, TanStack DB los almacena y consulta.

---

## Arquitectura de Namespaces: cómo encaja con TanStack DB

Arquitectura definida en [`decision.md`](https://github.com/royalcala/livestore/blob/syntrix/docs/testing-rao/decision.md): **namespace por escritor, merge en lectura.**

### Principio

Cada empleado escribe a su PROPIO namespace transaccional. Nadie comparte Write con nadie. Los catálogos compartidos los escribe solo admin. El merge se hace en lectura.

```
Namespaces por tipo:

Catálogos (1 escritor: admin, N lectores)
├── org_products         Write: admin      Read: todos
├── org_customers        Write: admin      Read: todos
└── org_chart_accounts   Write: admin      Read: todos

Transaccional (1 escritor por namespace)
├── invoices_alice       Write: alice      Read: admin, contabilidad
├── invoices_bob         Write: bob        Read: admin, contabilidad

Privado (1 escritor, 1-2 lectores)
├── user_alice           Write: alice      Read: alice, admin
└── user_bob             Write: bob        Read: bob, admin

Sensible (1 escritor, pocos lectores)
└── org_payroll          Write: admin      Read: admin, HR, contabilidad
```

### Cómo se crean y comparten los namespaces

**1. Admin crea la org** (`syntrix-admin/src-tauri/src/admin.rs:6-32`):

```
create_org("acme") →
  Crea control_doc + data_doc
  Escribe en org_acme/control:
    members/<admin_node_id>: { active: true, role: "admin", person: "alice" }
    roles/admin: { can_open: ["*"], can_write: ["*"] }
    org/: { name: "acme" }
```

**2. Admin agrega un empleado** (`admin.rs:34-65`):

```
add_device("bob", "sales", "Bob Martinez") →
  Escribe members/<bob_node_id>: { active: true, role: "sales", person: "Bob" }
  Auto-popula role grants:
    sales: can_open: ["org_data", "org_public", "org_control"], can_write: ["org_data"]
```

**3. Admin comparte tickets Read** (`admin.rs:100-113`):

```
share_org_tickets() →
  Genera Write-mode tickets para control_doc + data_doc
  Envía al peer vía invite protocol (QUIC ALPN /syntrix/invite/1)
```

**4. Control namespace define permisos** (`org_<id>/control`):

```json
{
  "members/<node_id>": { "active": true, "role": "sales" },
  "roles/sales": { "can_open": ["org_data", "org_public"], "can_write": ["org_data"] },
  "namespaces/invoices_bob": {
    "status": "active",
    "writers": ["bob_node_id"],
    "readers": ["alice_node_id", "contabilidad_role"],
    "capabilities": {
      "alice_node_id": "<encrypted ticket>",
      "contabilidad_role": "<encrypted with role key>"
    }
  }
}
```

**5. Validación de escritura** — 3 capas (`syntrix-client/DESIGN.md:159-167`):

- **Layer 1** — `accept_cb`: sin capability → el peer ni siquiera sincroniza
- **Layer 2** — local: consulta `org_control`, el rol `sales` tiene `can_write: ["org_data"]`
- **Layer 3** — otros peers validan y descartan si no coincide

### Cómo funciona el merge (LiveStore/LiveSQL → TanStack DB)

**LiveStore approach (anterior):**

```
LiveStore → SQLite
  invoices_view =
    SELECT * FROM invoices_alice
    UNION ALL
    SELECT * FROM invoices_bob
    UNION ALL ...
```

**TanStack DB approach (recomendado):** más simple, más eficiente.

Opción A — **una sola collection por tipo de dato:**

```
adapter sync() carga TODAS las entradas de invoices_* en UNA collection invoices.
Cada entry tiene campos: { namespace, author, ...payload }

useLiveQuery((q) =>
  q.from({ invoice: invoicesCollection })
   .where(({ invoice }) => eq(invoice.status, 'open'))
   .orderBy(({ invoice }) => invoice.created_at, 'desc')
)
```

El adapter `irohCollectionOptions`:
```typescript
const sync = ({ begin, write, commit, markReady }) => {
  // 1. Abre N namespaces de iroh-docs según org_control
  const namespaces = getReadableNamespaces(orgId, role) 
  // → ["org_data/invoices_alice", "org_data/invoices_bob", ...]

  // 2. Carga inicial: itera entries de todos los namespaces
  begin()
  for (const ns of namespaces) {
    const entries = invoke('sync_pull', { namespace: ns, cursor: null })
    for (const entry of entries) {
      write({ type: 'insert', value: parseEntry(entry, ns) })
    }
  }
  commit()
  markReady()

  // 3. Subscribe a eventos de iroh-docs para tiempo real
  listen('data-changed', (event) => {
    begin()
    if (event.type === 'new_entry') write({ type: 'insert', value: parseEntry(event) })
    commit()
  })
}
```

Opción B — **una collection por namespace** (espeja `decision.md` 1:1):

```typescript
// Cada namespace = una collection separada
const invoicesAlice = createCollection(irohCollectionOptions({ namespace: 'invoices_alice' }))
const invoicesBob   = createCollection(irohCollectionOptions({ namespace: 'invoices_bob' }))

// Query mergeada: carga todas las collections relevantes
useLiveQuery((q) =>
  q.from({ alice: invoicesAlice, bob: invoicesBob })
   .where(...)
   .select(({ alice, bob }) => ({ ... }))
)
```

**Recomendación: Opción A** (una collection por tipo). Es más simple, el adapter carga de N namespaces a 1 collection, y TanStack DB mantiene los índices automáticamente. Los 100k invoices caben en differential dataflow sin problema.

### Por qué este diseño evita problemas

| Problema | Solución |
|----------|----------|
| Revocación de Write | Nadie comparte Write. Si Bob se va, `invoices_bob` queda como archivo. |
| Rotación de namespaces | Innecesaria. El namespace del ex-empleado es histórico. |
| Conductor / SPOF | Inexistente. Cada empleado escribe directo a su namespace. |
| Datos duplicados | Cada entry vive UNA vez. El adapter mergea en lectura. |
| Cambio de rol | Solo afecta Read: se agregan/quitan tickets. |

### Despido: qué pasa

```
1. Admin marca members/bob → { active: false }
2. Se sync a todos los peers vía gossip
3. Peer de Bob: deja de recibir updates de la org
4. Peer de Alice (admin): accept_cb rechaza conexiones de Bob
5. invoices_bob: intacto como archivo histórico, legible por contabilidad
6. Datos de OTRAS orgs de Bob: no se tocan (aislamiento multi-org)
```

## Relación entre Durable Streams y TanStack DB

Son **ortogonales**. No dependen uno del otro, pero se complementan:

| Capa | Durable Streams | TanStack DB |
|------|----------------|-------------|
| ¿Qué es? | Protocolo de transporte (HTTP streams) | Store local de datos |
| ¿Dónde vive? | Servidor HTTP | Cliente (navegador) |
| ¿Qué resuelve? | Sync, replay, resume | Queries, relaciones, optimistic state |
| ¿Pueden combinarse? | ✅ Un adapter de TanStack DB podría usar Durable Streams como sync engine | ✅ |

**Ejemplo de combinación:** ElectricSQL (creadores de Durable Streams) son sponsors de TanStack DB. Su adapter `@tanstack/electric-db-collection` integra ambos. Durable Streams mueve los datos, TanStack DB los almacena y consulta.

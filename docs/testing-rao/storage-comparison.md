# LiveStore vs redb-direct vs SurrealDB — Análisis para Syntrix

## El problema

Syntrix necesita persistencia local + queries + sync P2P. Tenemos iroh-docs (redb) como source of truth para sync. ¿Qué capa de queries ponemos arriba?

## Las tres opciones

### Opción A: LiveStore (event sourcing + WASM SQLite)

```
Frontend React → LiveStore hooks → SQLite WASM → iroh-docs (vía SyncBackend)
```

- **Madurez:** v0.4.0-beta. API cambia entre versiones.
- **Dependencias:** Effect-TS (~50 paquetes), WASM SQLite, web workers.
- **Curva de aprendizaje:** Alta. Effect-TS, schemas, materializers, adapters.
- **Sync:** Diseñado para sync vía WebSocket/HTTP. Adaptar a iroh-docs requiere implementar un `Adapter` custom (cambió de `SyncBackend` en v0.4.0).
- **Offline-first:** Nativo. Event sourcing = siempre podés escribir local.
- **Queries reactivas:** `store.useQuery()` — la UI se actualiza sola cuando cambian los datos.
- **Peso:** ~2MB WASM + ~1MB JS.

**Pros:**
- Reactividad built-in (useQuery)
- Event sourcing alineado con nuestro modelo P2P
- Múltiples stores (device + per-org)

**Contras:**
- Complejidad extrema para MVP (~50 deps Effect)
- No publicado en npm (necesita monorepo linking)
- v0.4.0-beta, API inestable
- Duplica el almacenamiento (SQLite + redb)

### Opción B: redb directo con capa de queries en Rust

```
Frontend React → invoke() → Rust → redb (via iroh-docs store)
```

- **Madurez:** redb es estable (v2.x), iroh-docs lo usa internamente.
- **Dependencias:** Cero. redb ya está en el stack vía iroh-docs.
- **Curva de aprendizaje:** Baja. Rust estándar, queries por key-prefix.
- **Sync:** Directo. iroh-docs ya sincroniza redb entre peers.
- **Offline-first:** Nativo. redb es local-first.
- **Queries reactivas:** Manual. Necesitamos implementar polling o eventos Tauri.
- **Peso:** 0 KB extra (ya está en el binary de Rust).

**Pros:**
- Cero dependencias nuevas
- Sin duplicación de datos (único store)
- Performante (Rust nativo, sin WASM)
- Código simple y mantenible
- Ya tenemos 90% implementado

**Contras:**
- Sin queries SQL (key-value solamente)
- Sin reactividad automática (necesita polling/eventos)
- Sin joins, agregaciones, índices secundarios
- Para queries complejas hay que iterar entries y filtrar en memoria

### Opción C: SurrealDB embebido

```
Frontend React → invoke() → Rust → SurrealDB → iroh-docs (sync manual)
```

- **Madurez:** v2.x stable.
- **Dependencias:** `surrealdb` crate (~30MB binary).
- **Curva de aprendizaje:** Media. SurrealQL (similar a SQL), SDK Rust.
- **Sync:** NO nativo. SurrealDB es single-node. Hay que implementar sync a iroh-docs manualmente.
- **Offline-first:** Sí (embebido), pero el sync es complejo.
- **Queries reactivas:** `LIVE SELECT` — suscripciones en tiempo real.
- **Features extra:** Grafos, full-text search, schemaless, SurrealQL.

**Pros:**
- SQL + Grafos + Full-text search
- LIVE queries (reactividad nativa)
- Embebido, sin servidor
- Schemaless flexible

**Contras:**
- +30MB al binary
- Duplica almacenamiento (SurrealDB + redb)
- Sync a iroh-docs NO es nativo (hay que construir todo el bridge)
- Single-node por diseño (no P2P)
- El modelo de datos de SurrealDB no mapea 1:1 con iroh-docs

## Comparativa

| Dimensión | LiveStore | redb-direct | SurrealDB |
|-----------|-----------|-------------|-----------|
| Peso | ~3MB WASM+JS | 0 KB | ~30MB |
| Dependencias | ~50 npm | 0 | 1 crate Rust |
| Curva aprendizaje | Alta (Effect-TS) | Baja (Rust) | Media (SurrealQL) |
| Sync P2P nativo | No (adapter custom) | ✅ (iroh-docs) | No (sync manual) |
| Offline-first | ✅ | ✅ | ✅ |
| Queries SQL | ✅ (SQLite) | ❌ (KV only) | ✅ (SurrealQL) |
| Grafos | ❌ | ❌ | ✅ |
| Reactividad | ✅ (useQuery) | Manual | ✅ (LIVE SELECT) |
| Duplicación datos | SQLite + redb | Ninguna | SurrealDB + redb |
| Mantenibilidad | Baja (API cambia) | Alta | Media |
| Madurez | Beta | Stable | Stable |

## ¿Qué queries necesitamos realmente?

Veamos las queries de nuestro ERP:

| Query | SQL | redb |
|-------|-----|------|
| "Invoices de hoy" | `SELECT * WHERE date = today()` | Iterar entries con prefix `evt:`, filtrar por fecha en Rust |
| "Total facturado por vendedor este mes" | `SELECT writer, SUM(total) GROUP BY` | Iterar, agrupar en un `HashMap` |
| "Productos con stock < 10" | `SELECT * WHERE stock < 10` | No aplica (productos son pocos, se cachean en memoria) |
| "Clientes que no compraron en 30 días" | JOIN invoices + customers | Iterar invoices, construir set de customers activos, restar del total |

**Conclusión:** para un ERP chico (5 empleados, 100 invoices/mes, 500 productos, 200 clientes), **todas estas queries se pueden hacer iterando entries de redb en Rust.** El dataset completo cabe en memoria (< 1 MB). Las queries toman microsegundos.

SQL solo se vuelve necesario cuando:
- 10,000+ invoices
- Queries ad-hoc del usuario (filtros dinámicos)
- Joins complejos entre múltiples tablas
- Agregaciones con múltiples dimensiones

## Recomendación

**Fase 1 (MVP, HOY): redb directo con capa de queries en Rust.**

Ya lo tenemos. Las queries actuales (`query_invoices`, `query_products`, `query_customers`) son stubs que devuelven `Vec<T>` vacío. Completarlos con iteración real de redb es ~100 líneas de Rust.

Para reactividad, ya tenemos Tauri events funcionando (invite flow). Mismo patrón: cuando iroh-docs recibe un entry nuevo → emitir evento → React actualiza.

**Fase 2 (escalar): si las queries se vuelven complejas, evaluar:**

- **Agregar SQLite como cache de lectura** (no como source of truth). Un proceso en Rust que escucha eventos de iroh-docs y actualiza tablas SQLite. Las queries van contra SQLite, los writes van a iroh-docs. Sin event sourcing, sin LiveStore.
- **O migrar a LiveStore** cuando esté estable (v1.0) y tengamos un equipo más grande.

**No usar SurrealDB.** El overhead de 30MB + sync manual no se justifica para nuestro volumen de datos.

## Arquitectura recomendada (Fase 1)

```
┌─────────────────────────────────────────────┐
│  React UI                                    │
│  invoke("query_invoices", {filter})           │
│  listen("data-changed", callback)             │
├─────────────────────────────────────────────┤
│  Rust (Tauri command)                         │
│  ┌───────────────────────────────────────┐   │
│  │  Query Layer (~100 líneas)             │   │
│  │  - key_prefix scan en redb             │   │
│  │  - filter/deserialize en Rust          │   │
│  │  - cache en memoria (opcional)         │   │
│  └───────────────────────────────────────┘   │
│         ↓ write              ↑ read           │
│  ┌───────────────────────────────────────┐   │
│  │  iroh-docs (redb)                      │   │
│  │  - entries firmadas por HLC            │   │
│  │  - sync P2P automático                 │   │
│  └───────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**Beneficio principal:** un solo source of truth (redb). Sin duplicación. Sin deps extra. Sin complejidad.

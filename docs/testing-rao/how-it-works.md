# Cómo funciona LiveStore

## 1. Event Sourcing

LiveStore usa un **log de eventos** como fuente de verdad. En lugar de mutar estado directamente, las aplicaciones commitean **eventos** (con payloads validados por Effect Schema):

- Los eventos se persisten en un **event log** local
- Se aplican a la base de datos SQLite local mediante **materializers**
- Se sincronizan entre clientes vía el backend de sync

```
Commit event → persistir en event log → aplicar materializers → actualizar SQLite → notificar queries reactivas
```

## 2. Grafo de dependencias reactivo

El core (`@livestore/livestore/src/reactive.ts`) implementa un grafo reactivo inspirado en el paper MiniAdapton:

- **Refs**: celdas mutables donde se asignan valores
- **Thunks**: computaciones puras que dependen de otros átomos
- **Effects**: side effects que se ejecutan cuando los valores cambian

El grafo se refresca en orden topológico, con comparación de igualdad de valores para cortar la propagación.

## 3. Sistema de Schemas

Definido en `@livestore/common/src/schema/`:

- **LiveStoreSchema**: compuesto de definiciones de tablas (SQLite DDL), definiciones de eventos y materializers
- **TableDef**: definiciones de tablas usando Effect Schema
- **EventDef**: definiciones de tipos de eventos con esquemas de payload
- **Materializers**: funciones que aplican eventos a la base de datos SQLite

## 4. Arquitectura de Web Workers (Navegador)

El adaptador de navegador (`@livestore/adapter-web`) usa múltiples workers:

- **Leader Worker**: dueño de la base de datos SQLite y el event log (persistido en OPFS)
- **Shared Worker**: gestiona locks y ruteo entre tabs
- **Client Session**: cada tab se conecta al leader vía MessageChannel

```
Tab A ←→ Shared Worker ←→ Leader Worker (SQLite + OPFS)
Tab B ←→ Shared Worker ↲
```

## 5. Webmesh (Mesh Networking)

El paquete `@livestore/webmesh` provee comunicación tipo red (inspirado en Elixir/OTP Distribution) conectando tabs, workers y ventanas. Soporta:

- **Proxy channels**: comunicación indirecta vía nodos intermedios
- **Direct channels**: con objetos transferibles
- **Broadcast channels**: difusión a todos los nodos

Esto potencia el protocolo de DevTools.

## 6. Protocolo de Sync

Implementado entre `@livestore/common/src/sync/` y `@livestore/sync-cf/`:

- Interfaz **SyncBackend** pluggable
- Sincronización push/pull de eventos
- Detección de **Backend ID mismatch** (para resets en desarrollo)
- **Live pull** para sync en tiempo real (WebSocket o SSE)
- Transporte en chunks para batches grandes de eventos

## 7. Sistema de Queries

LiveStore provee múltiples primitivas de consulta en `@livestore/livestore/src/live-queries/`:

- **signal**: crea una celda reactiva a partir de SQL crudo
- **computed**: deriva valores de otras signals/queries
- **queryDb**: template tag SQL para queries parametrizadas
- **QueryBuilder**: query builder SQL tipado usando las definiciones del schema

## 8. Estándar de escritura (Writes)

Las escrituras en LiveStore siguen el paradigma de **event sourcing**: no se muta el estado directamente, se commitean eventos.

```ts
store.commit(events.todoCreated({ id: nanoid(), text: 'Make coffee' }))
store.commit(
  events.todoCreated({ id: '1', text: 'A' }),
  events.todoCompleted({ id: '1' })
)
```

- Cada evento tiene un **nombre**, **payload validado con Effect Schema** y **sequence number**
- Los eventos se persisten primero en el **event log** (fuente de verdad)
- Luego los **materializers** aplican los eventos a la base SQLite local
- Se soportan transacciones (múltiples eventos atómicos) y `skipRefresh` para batches grandes (`packages/@livestore/livestore/src/store/store.ts:771`)

## 9. Estándar de lectura de sync (Pull/Push)

La interfaz `SyncBackend` (`packages/@livestore/common/src/sync/sync-backend.ts:45`) define el contrato de sincronización:

**Pull** (leer eventos remotos):
```ts
pull(cursor, options?: { live?: boolean }) => Stream<PullResItem>
```
- Recibe un `cursor` con el último `eventSequenceNumber` procesado + metadata opcional
- Retorna un **Stream** de `PullResItem` con `batch` (array de eventos encoded) y `pageInfo` (`MoreKnown`, `MoreUnknown`, `NoMore`)
- Soporta `live: true` para streaming en tiempo real (WebSocket/SSE)

**Push** (enviar eventos locales):
```ts
push(batch: ReadonlyArray<LiveStoreEvent.Global.Encoded>) => Effect<void>
```
- Recibe batches de 1-100 eventos con sequence numbers en orden ascendente
- Errores tipados: `IsOfflineError`, `BackendIdMismatchError`, `ServerAheadError`

## 10. Detección y manejo de cambios de schema

LiveStore detecta cambios de schema automáticamente al iniciar la app mediante **hash-based change detection** (`packages/@livestore/common/src/schema-management/migrations.ts`):

**Tablas de estado (STATE) — seguro modificar:**
- Cada tabla se hashea con `SqliteAst.hash()`
- Los hashes se comparan contra los guardados en la tabla del sistema `__livestore_schema`
- Si hay mismatch → la tabla se recrea y los datos se **rematerializan desde el event log**
- Sin pérdida de datos: el event log es la fuente de verdad

**Tablas de eventlog — NUNCA modificar sin precaución:**
- Cambios al schema de `eventlogMetaTable` o `syncStatusTable` causan "soft reset"
- La tabla vieja queda inaccesible (pérdida efectiva de datos)
- Se requiere incrementar manualmente `liveStoreStorageFormatVersion` (actualmente `6`, en `packages/@livestore/common/src/version.ts:38`)

**Schemas de eventos:**
- Detectados por hash y guardados en `__livestore_schema_event_defs`
- Si el hash cambia, se loguea warning pero se intenta materializar igual
- Si el payload no decodifica con el nuevo schema → error en `rematerializeFromEventlog` (`packages/@livestore/common/src/rematerialize-from-eventlog.ts:74-86`)

**Estrategias de migración:**
- `auto` (default): crea nuevos archivos de DB por cada cambio de schema (ej: `state123.db`, `state456.db`)
- `manual`: reusa siempre el mismo archivo (`statefixed.db`) — útil para migraciones manuales controladas (`packages/@livestore/adapter-web/src/web-worker/common/persisted-sqlite.ts:165`)

La detección ocurre en cada boot de la app, no en tiempo real durante la ejecución.

## 11. Integración con el ecosistema Effect

LiveStore está profundamente integrado con el ecosistema **Effect** (`effect`, `@effect/sql`, `@effect/schema`, `@effect/platform`):

- Todas las operaciones asíncronas usan el tipo `Effect` de Effect
- Validación de schemas vía `@effect/schema` (reexportado como `Schema`)
- Inyección de dependencias vía Effect `Context` / `Layer`
- OpenTelemetry tracing a través de todo el stack

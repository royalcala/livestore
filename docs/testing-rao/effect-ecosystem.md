# Ecosistema Effect y su integración con LiveStore

## ¿Qué es Effect?

**Effect** (`effect` v3.21.2) es una librería de programación funcional para TypeScript que reemplaza las excepciones, promesas y side effects con tipos seguros. Su unidad central es el tipo `Effect<Success, Error, Requirements>`:

- **Success**: lo que produce si sale bien (`A`)
- **Error**: qué errores puede producir (`E`)
- **Requirements**: qué dependencias necesita (inyección de dependencias tipada) (`R`)

Todo el comportamiento asíncrono y concurrente en LiveStore está construido sobre este tipo.

El ecosistema Effect incluye:

| Paquete | Propósito |
|---------|-----------|
| `effect` | Core: `Effect`, `Stream`, `Fiber`, `Queue`, `Deferred`, `Scope`, `Runtime`, `Duration`, `Cause` |
| `@effect/schema` | Schemas tipados para validación, encoding/decoding en runtime (reexportado como `Schema` por LiveStore) |
| `@effect/sql` | Abstracción tipada para bases de datos SQL |
| `@effect/platform` | Primitivas multiplataforma (HTTP, file system, workers) |
| `@effect/opentelemetry` | Instrumentación OpenTelemetry |
| `@effect/rpc` | Llamadas a procedimiento remoto tipadas |
| `@effect/vitest` | Integración con Vitest para testing |

LiveStore re-exporta todo desde `@livestore/utils/effect` (`packages/@livestore/utils/src/effect/mod.ts`), proporcionando una capa unificada de imports.

---

## Cómo LiveStore usa cada concepto de Effect

### 1. `Effect` — operaciones asíncronas tipadas

Toda función asíncrona en LiveStore retorna un `Effect`. Esto permite composición segura sin callbacks ni try/catch.

```ts
// createStore retorna Effect<Store, UnknownError, Scope | OtelTracer>
export const createStore = (...): Effect.Effect<
  Store<TSchema, TContext>,
  UnknownError,
  Scope.Scope | OtelTracer.OtelTracer
>
```
(`packages/@livestore/livestore/src/store/create-store.ts:267`)

### 2. `Effect.gen` — sintaxis async/await tipada

LiveStore usa `Effect.gen` masivamente para escribir código asíncrono que parece sincrónico pero retiene toda la información de tipos:

```ts
Effect.gen(function* () {
  const store = yield* createStore({ schema, storeId, adapter, ... })
  yield* store[StoreInternalsSymbol].boot
  return store
})
```
(`packages/@livestore/livestore/src/store/create-store.ts:301`)

La palabra clave `yield*`:
- Para dependencias: resuelve requisitos del contexto (inyección de dependencias)
- Para otros Effects: ejecuta y obtiene el valor de éxito
- Los errores se propagan automáticamente en el canal `E`

### 3. `Context` y `Layer` — inyección de dependencias

El sistema de dependencias de Effect reemplaza los singletons globales y los service locators. LiveStore lo usa para que el Store esté disponible en todo el árbol de componentes:

```ts
// Definir un Tag (identificador tipado del servicio)
class MainStore extends Store.Tag(schema, 'main') {}

// Crear una Layer (provee el servicio al contexto)
const layer = MainStore.layer({ adapter: webAdapter, batchUpdates: ReactDOM.unstable_batchedUpdates })

// Usar en código Effect
Effect.gen(function* () {
  const { store } = yield* MainStore  // resuelto del contexto
  const todos = yield* MainStore.query(tables.todos.all())
  yield* MainStore.commit(events.todoCreated({ id: '1', text: 'Hola' }))
})
```
(`packages/@livestore/livestore/src/effect/LiveStore.ts:170`)

**Cómo funciona:**
- `Context.Tag('nombre')` → crea un tag (identificador tipado) para un servicio
- `Layer.scoped(Tag, effect)` → crea una capa que provee el servicio
- `Layer.provide(dep)` → compone capas (la de arriba requiere la de abajo)
- `yield* Tag` → extrae el servicio del contexto en `Effect.gen`

**Patrón Deferred**: LiveStore usa `Deferred` + `Context.Tag` para inicialización asíncrona:
```ts
// Se crea un Deferred (promesa de Effect)
const DeferredLayer = Layer.effect(DeferredTag, Deferred.make<RunningType>())
// El store se resuelve después de boot
yield* Deferred.succeed(deferred, ctx)
```
(`packages/@livestore/livestore/src/effect/LiveStore.ts:233`)

### 4. `Schema` — validación tipada en runtime

LiveStore re-exporta `@effect/schema` como `Schema`. Se usa para:

**Schemas de eventos** — validar payloads de eventos:
```ts
const todoCreated = Schema.Struct({
  id: Schema.String,
  text: Schema.String,
  completed: Schema.Boolean,
})
```
(`packages/@livestore/common/src/schema/EventDef/`)

**Schemas de tablas** — definir columnas de SQLite:
```ts
const table = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  createdAt: Schema.DateTimeUtc,
})
```

**Sync payload** — schemas para el payload de sincronización:
```ts
const SyncPayload = Schema.Struct({ authToken: Schema.String })
```

**Hash de schemas** — detección de cambios:
```ts
const schemaHash = Schema.hash(eventDef.schema)
```
(`packages/@livestore/common/src/schema-management/validate-schema.ts:37`)

### 5. `Stream` — datos reactivos y sync en tiempo real

Effect `Stream` es una fuente de valores asíncrona y cancelable. LiveStore lo usa para:

**Sync pull** — streaming de eventos desde el backend:
```ts
type SyncBackend = {
  pull: (cursor) => Stream.Stream<PullResItem, IsOfflineError | BackendIdMismatchError | UnknownError>
}
```
(`packages/@livestore/common/src/sync/sync-backend.ts:50`)

**Rematerialización** — re-aplicar eventos desde el eventlog:
```ts
Stream.unfoldChunk(initial, (item) => {
  // cargar chunks de 100 eventos
  const nextItem = Chunk.fromIterable(stmt.select(...))
  return Option.some([prevItem, nextItem])
}).pipe(Stream.runDrain)
```
(`packages/@livestore/common/src/rematerialize-from-eventlog.ts:102`)

**Devtools messaging** — protocolo basado en Streams.

### 6. `Scope` — manejo de recursos y ciclo de vida

`Scope` es el mecanismo de Effect para liberar recursos. En LiveStore:

```ts
// createStore crea un scope de vida del store
const lifetimeScope = yield* Scope.make()
yield* Effect.addFinalizer(() => Scope.close(lifetimeScope, exit))

// El store y todos sus recursos se liberan al cerrar el scope
// (fibers, queues, database connections, etc.)
```
(`packages/@livestore/livestore/src/store/create-store.ts:292`)

Todo recurso adquirido con `Effect.acquireRelease` se libera automáticamente cuando el scope se cierra.

### 7. `Fiber` — concurrencia ligera

LiveStore usa fibers para ejecutar procesos en background:

```ts
// Forkear el procesamiento de boot status
yield* Queue.take(bootStatusQueue).pipe(
  Effect.forever,
  Effect.forkScoped,  // se cancela al cerrar el scope
)
```
(`packages/@livestore/livestore/src/store/create-store.ts:308`)

En el Store se inician múltiples fibers al boot:
- Sync processor (push/pull de eventos)
- Event stream processor
- DevTools mesh connection

### 8. `Queue` — comunicación entre fibers

Colas usadas para comunicación asíncrona entre componentes:

```ts
const bootStatusQueue = yield* Queue.unbounded<BootStatus>()
const syncPullQueue = yield* Queue.unbounded<LiveStoreEvent.Global.Encoded>()
```
(`packages/@livestore/livestore/src/store/create-store.ts:306`, `packages/@livestore/common/src/sync/mock-sync-backend.ts:51`)

### 9. `Deferred` — promesas tipadas de Effect

Similar a `Promise` pero integrado con el ecosistema Effect:

```ts
// Crear
const storeDeferred = yield* Deferred.make<Store>()

// Resolver (puede fallar)
yield* Deferred.succeed(storeDeferred, store)
yield* Deferred.fail(deferred, error)

// Esperar (bloquea el fiber hasta que se resuelva)
const store = yield* storeDeferred
```
(`packages/@livestore/livestore/src/store/create-store.ts:316`)

### 10. `SubscriptionRef` — estado reactivo compartido

Usado para exponer cambios de estado a suscriptores (ej. estado de conectividad):

```ts
type SyncBackend = {
  isConnected: SubscriptionRef.SubscriptionRef<boolean>
}
```
(`packages/@livestore/common/src/sync/sync-backend.ts:76`)

### 11. `Runtime` — ejecutar Effects fuera del contexto

Cuando se necesita ejecutar código Effect desde un callback no-Effect:

```ts
const runtime = yield* Effect.runtime<Scope.Scope>()
// ... luego desde un callback
Effect.gen(function* () { ... }).pipe(
  Effect.provide(runtime),
  Effect.runFork,
  Fiber.join,
)
```
(`packages/@livestore/livestore/src/store/create-store.ts:324`)

### 12. OpenTelemetry — tracing y observabilidad

LiveStore integra OpenTelemetry mediante `@effect/opentelemetry`:

```ts
Effect.withSpan('createStore', { attributes: { storeId } })
Effect.withSpan('createStore:boot')
Effect.withPerformanceMeasure('livestore:makeAdapter')
```
(`packages/@livestore/livestore/src/store/create-store.ts:438`)

Cada operación significativa tiene su span, permitiendo trazas distribuidas.

---

### 13. `Logger` — registro del estado de los datos

Effect incluye un sistema de logging estructurado que LiveStore usa extensivamente para rastrear el estado interno de los datos:

#### Niveles y funciones de log

| Función | Nivel | Uso en LiveStore |
|---------|-------|------------------|
| `Effect.log(...)` | Info | Estados generales: sync status, inicio/apagado de servicios |
| `Effect.logDebug(...)` | Debug | Migraciones de schema, comunicación mesh, boot/shutdown |
| `Effect.logWarning(...)` | Warning | Schema hash mismatches, operaciones lentas, timeouts |
| `Effect.logError(...)` | Error | Fallos de sync, errores de decodificación, shutdown forzado |

#### Ejemplos reales del código

**Log de estado de sync** — el store expone el estado de sincronización con `printSyncStates()`:
```ts
// packages/@livestore/livestore/src/store/store.ts:1153
printSyncStates: () => {
  Effect.gen(this, function* () {
    const session = yield* this[StoreInternalsSymbol].syncProcessor.syncState
    yield* Effect.log(
      `Session sync state: ${objectToString(session.localHead)} (upstream: ${objectToString(session.upstreamHead)})`,
      session.toJSON(),
    )
    const leader = yield* this[StoreInternalsSymbol].clientSession.leaderThread.syncState
    yield* Effect.log(
      `Leader sync state: ${objectToString(leader.localHead)} (upstream: ${objectToString(leader.upstreamHead)})`,
      leader.toJSON(),
    )
  })
}
```

**Log de migraciones de schema** — al detectar cambios de schema en el boot:
```ts
// packages/@livestore/livestore/src/store/create-store.ts:371
if (migrationsReport.migrations.length > 0) {
  yield* Effect.logDebug(
    '[@livestore/livestore:createStore] migrationsReport',
    ...migrationsReport.migrations.map((m) =>
      m.hashes.actual === undefined
        ? `Table '${m.tableName}' doesn't exist yet. Creating table...`
        : `Schema hash mismatch for table '${m.tableName}' (DB: ${m.hashes.actual}, expected: ${m.hashes.expected}), migrating table...`,
    ),
  )
}
```

**Log de ciclo de vida del store**:
```ts
// packages/@livestore/livestore/src/store/create-store.ts:331
yield* Effect.logWarnIfTakesLongerThan({ label: '@livestore/livestore:shutdown', duration: 500 })
yield* Effect.logDebug('LiveStore shutdown complete')
```

**Log con metadatos estructurados** — `Effect.annotateLogs` agrega contexto a todos los logs:
```ts
// packages/@livestore/livestore/src/store/create-store.ts:439
Effect.annotateLogs({ debugInstanceId, storeId })
// packages/@livestore/livestore/src/store/create-store.ts:262
Effect.annotateLogs({ thread: 'window' })
```

**Log de errores con OpenTelemetry** — `tapCauseLogPretty` agrega spanId y traceId:
```ts
// packages/@livestore/utils/src/effect/Effect.ts:90
yield* Effect.logError(firstErrLine, cause).pipe((_) =>
  span === undefined
    ? _
    : Effect.annotateLogs({ spanId: span.spanContext().spanId, traceId: span.spanContext().traceId })(_),
)
```

#### Configuración de logging

LiveStore expone `LogConfig` (`packages/@livestore/common/src/logging.ts`) para que el usuario controle:

```ts
// Nivel mínimo de log (default: Debug en dev, Info en prod)
const level = resolveLogLevel(config, defaults)
// Logger layer personalizable (default: Logger.prettyWithThread)
const layer = resolveLoggerLayer(config, defaults)
```

El usuario puede pasar `logger` y `logLevel` al crear el store para controlar la verbosidad o usar un logger custom.

---

## Flujo completo: del boot a la query reactiva

```
1. createStore(package/@livestore/livestore/src/store/create-store.ts)
   │  Effect.gen(function* () { ... })
   │  Requiere: Scope.Scope + OtelTracer.OtelTracer
   ▼
2. Scope.make() — crea el lifetimeScope
   │
   ▼
3. Queue.unbounded<BootStatus>() — cola de estado de boot
   │
   ▼
4. Deferred.make<Store>() — promesa del store
   │
   ▼
5. adapter({ schema, storeId, ... }) — inicia el adapter (Web Worker o Cloudflare)
   │  Efecto con span 'createStore:makeAdapter'
   │
   ▼
6. new Store({ clientSession, schema, ... }) — constructor del Store
   │
   ▼
7. store[StoreInternalsSymbol].boot — inicia fibers de background (sync, eventlog, devtools)
   │  forkScoped en el lifetimeScope
   │
   ▼
8. Deferred.succeed(storeDeferred, store) — el store está listo
   │
   ▼
9. Las queries reactivas usan el grafo de dependencias (reactive.ts)
   │  Las dependencias se rastrean automáticamente con getters
   │  Los thunks se recomputan en orden topológico cuando los refs cambian
   │
   ▼
10. Al hacer shutdown: Scope.close(lifetimeScope, exit)
    │  Todos los fibers, queues, conexiones se liberan
    ▼
    Store cerrado limpiamente
```

---

## El ecosistema Effect como runtime de LiveStore

LiveStore no solo "usa Effect" — **Effect es el runtime de LiveStore**. Provee:

- **Concurrencia estructurada**: fibers + scopes garantizan que no haya leaks
- **Manejo de errores tipado**: el canal `E` del Effect documenta todos los errores posibles (`UnknownError`, `MaterializeError`, `BackendIdMismatchError`, etc.)
- **Cancelación**: cualquier Effect puede cancelarse limpiamente vía scopes
- **Tracing**: cada operación es rastreable extremo a extremo con OpenTelemetry
- **Inyección de dependencias**: en lugar de singletons globales, los servicios se pasan por contexto
- **Testing**: Effect facilita mockear dependencias mediante layers alternativas

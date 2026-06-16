# LiveStore

LiveStore es una **capa de datos reactiva centrada en el cliente** para aplicaciones web y móviles. Reemplaza librerías tradicionales de manejo de estado (Redux, MobX, Zustand) con una arquitectura basada en una **base de datos SQLite embebida y reactiva** (compilada a WebAssembly). Los datos se sincronizan entre clientes en tiempo real usando **event sourcing**.

## Qué ofrece

- **Consultas reactivas instantáneas** a una base de datos SQLite local embebida (query builder tipado + SQL crudo)
- **Event sourcing** como backbone de sincronización — los cambios se persisten como eventos y se aplican mediante materializers
- **Offline-first** — los datos se persisten localmente y se sincronizan cuando hay conectividad
- **Sincronización en tiempo real** entre clientes con backends pluggables (Cloudflare Durable Objects, D1, custom)
- **Sincronización cross-tab** vía arquitectura de Web Workers (SharedWorker + Leader Worker)
- **Resolución de conflictos de merge personalizable**
- **Integraciones de framework**: React (principal), Vue, Expo, Node.js, Solid (en desarrollo)
- **DevTools** con capa de comunicación mesh entre ventanas/tabs/workers
- **Servidor MCP** para tooling de AI/automatización
- **Instrumentación OpenTelemetry** en todo el stack

## Paquetes principales

| Paquete | Propósito |
|---------|-----------|
| `@livestore/livestore` | Core: `createStore`, grafo reactivo, live queries, query builder |
| `@livestore/common` | Tipos compartidos, schemas, primitivas de sync, eventos, materializers |
| `@livestore/react` | Integración React: hooks (`useStore`, `useLiveQuery`), providers |
| `@livestore/adapter-web` | Adaptador navegador: Web Workers + OPFS para persistencia |
| `@livestore/adapter-cloudflare` | Adaptador Cloudflare Workers/Durable Objects |
| `@livestore/sync-cf` | Proveedor de sync para Cloudflare (cliente + servidor) |
| `@livestore/sqlite-wasm` | Bindings de SQLite WebAssembly por plataforma |
| `@livestore/webmesh` | Mesh networking para comunicación entre tabs/workers/windows |
| `@livestore/utils` | Utilidades compartidas (Effect, deepEqual, nanoid, etc.) |
| `@livestore/peer-deps` | Dependencias peer empaquetadas (Effect ecosystem, OpenTelemetry) |

## Documentos de análisis

| Documento | Contenido |
|-----------|-----------|
| [Cómo funciona LiveStore](./how-it-works.md) | Arquitectura interna: event sourcing, grafo reactivo, schemas, Web Workers, webmesh, protocolo de sync, sistema de queries, API de escritura (`store.commit`), API de sync (`SyncBackend.pull`/`push`), detección de cambios de schema. |
| [Ecosistema Effect](./effect-ecosystem.md) | Qué es Effect, sus paquetes principales, y cómo LiveStore integra cada concepto: `Effect.gen`, `Context`/`Layer` (inyección de dependencias), `Schema` (validación), `Stream` (sync en tiempo real), `Scope` (ciclo de vida), `Fiber` (concurrencia), `Queue`, `Deferred`, `SubscriptionRef`, `Runtime`, OpenTelemetry, `Logger`. |
| [Sync providers self-hosted](./sync-self-hosted.md) | Arquitecturas open source para el servidor de sync: Bun + SQLite + WebSocket (la más simple), PostgreSQL + LISTEN/NOTIFY, Turso/libsql, Go + BadgerDB, LiteFS, NATS JetStream. Con código completo para cada opción, tabla comparativa y recomendaciones según etapa del proyecto. |
| [ERP multi-sucursal con Iroh](./erp/iroh/erp-iroh-architecture.md) | Arquitectura P2P para ERP distribuido. Documento hub que enlaza a las 7 capas de detalle. |
| └─ [Diseño de red ERP](./erp/design.md) | Topología jerárquica: tenants aislados, malla privada por tenant, Tauri como nodo. |
| └─ [Contrato de sync de LiveStore](./erp/livestore-sync-contract.md) | Qué necesita LiveStore del backend: formato wire, cursor, push/pull payloads, errores, mapeo a Iroh Docs. |
| └─ [Iroh Doc como source of truth](./erp/iroh/erp-iroh-iroh-doc.md) | Estructura del doc por tenant, inicialización, ventajas sobre SQLite standalone. |
| └─ [Hybrid Logical Clock](./erp/iroh/erp-iroh-hlc.md) | Orden total sin servidor central, reglas del HLC, comparación con alternativas. |
| └─ [SyncBackend sobre Iroh](./erp/iroh/erp-iroh-sync-backend.md) | Adaptación de la interfaz de LiveStore para leer/escribir del Iroh Doc. |
| └─ [Permisos y autorización](./erp/iroh/erp-iroh-permissions.md) | Validación de eventos contra estado actual de SQLite, comparación con Jazz. |
| └─ [Ruteo geográfico](./erp/iroh/erp-iroh-routing.md) | Cómo el cliente encuentra el backend más cercano (LAN, DNS, cloud fallback). |
| └─ [Backend empaquetable](./erp/iroh/erp-iroh-backend.md) | Despliegue del backend en sucursal, estructura de archivos, comunicación con clientes. |
| └─ [Flujos y comparativa](./erp/iroh/erp-iroh-flows.md) | Diagramas de secuencia (con/sin internet, usuario remoto), tabla comparativa, riesgos. |

## Estado actual

- **Versión**: 0.4.0 (Junio 2026)
- **Estado**: Beta — se esperan breaking changes antes de 1.0
- **Licencia**: Apache 2.0
- **Repositorio**: `github.com/livestorejs/livestore`
- **Documentación**: `docs.livestore.dev`

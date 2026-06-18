# LiveStore v0.4.0 — Integración en Syntrix

## Estado actual

LiveStore v0.4.0-beta está en el monorepo `upstream-forks/livestore/`. Los paquetes no están publicados en npm. Se importan directamente desde source vía pnpm workspace linking.

## Paquetes necesarios

| Paquete | Propósito |
|---------|-----------|
| `@livestore/livestore` | Core: schemas, eventos, `createStore`, `StoreRegistry`, `store.useQuery()`, `store.commit()` |
| `@livestore/react` | React: `useStore()`, `StoreRegistryProvider`, `store.useQuery()` |
| `@livestore/sqlite-wasm` | WASM SQLite (vía `wa-sqlite`) |
| `@livestore/adapter-web` | OPFS adapter para Tauri WebView (`makeSingleTabAdapter`) |
| `@livestore/wa-sqlite` | Engine SQLite compilado a WASM |
| `@livestore/peer-deps` | Todas las peer deps de Effect-TS en un solo paquete |

## Arquitectura de stores

```
Device Store (local, sin sync)
├── notifications    ← invites, cambios de rol, alertas
├── orgs             ← lista de orgs donde soy miembro
└── settings         ← preferencias del dispositivo

Org Store (sync vía iroh-docs)
├── invoices         ← sync P2P entre miembros
├── products         ← catálogo compartido
└── customers        ← clientes compartidos
```

## Plan de integración

### Paso 1 — Workspace linking

Agregar los paquetes al `pnpm-workspace.yaml` de syntrix-client:

```yaml
packages:
  - .
  - ../upstream-forks/livestore/packages/@livestore/livestore
  - ../upstream-forks/livestore/packages/@livestore/react
  - ../upstream-forks/livestore/packages/@livestore/sqlite-wasm
  - ../upstream-forks/livestore/packages/@livestore/adapter-web
  - ../upstream-forks/livestore/packages/@livestore/wa-sqlite
  - ../upstream-forks/livestore/packages/@livestore/common
  - ../upstream-forks/livestore/packages/@livestore/utils
  - ../upstream-forks/livestore/packages/@livestore/peer-deps
  - ../upstream-forks/livestore/packages/@livestore/framework-toolkit
```

Dependencias en `package.json`:

```json
{
  "@livestore/livestore": "workspace:^",
  "@livestore/react": "workspace:^",
  "@livestore/sqlite-wasm": "workspace:^",
  "@livestore/adapter-web": "workspace:^",
  "@livestore/wa-sqlite": "workspace:^"
}
```

### Paso 2 — Worker para OPFS

```ts
// src/livestore.worker.ts
import { deviceSchema } from './schemas/device.schema'
import { makeWorker } from '@livestore/adapter-web/worker'

makeWorker({ schema: deviceSchema })
```

### Paso 3 — Device store

```tsx
import { makeSingleTabAdapter } from '@livestore/adapter-web'
import { storeOptions, StoreRegistry } from '@livestore/livestore'
import { useStore } from '@livestore/react'
import { unstable_batchedUpdates as batchUpdates } from 'react-dom'

const registry = new StoreRegistry()
const adapter = makeSingleTabAdapter({ storage: { type: 'opfs' } })

const deviceStoreOpts = storeOptions({
  storeId: 'device',
  schema: deviceSchema,
  adapter,
  batchUpdates,
})

export const useDeviceStore = () => useStore(deviceStoreOpts)
```

### Paso 4 — Org store (con sync iroh-docs)

El `SyncBackend` de LiveStore se implementa contra iroh-docs. El diseño original en `erp-iroh-sync-backend.md` sigue vigente pero hay que actualizarlo para la API de v0.4.0.

### Paso 5 — Reemplazar `invoke()` por `store.useQuery()` y `store.commit()`

Reemplazar los `invoke()` en las pantallas de React por llamadas directas al store de LiveStore.

## Bloqueantes conocidos

1. **Peer deps Effect-TS**: `@livestore/peer-deps` instala `effect 3.21.2` y ~50 paquetes del ecosistema. Impacto en build time.
2. **WASM SQLite**: necesita `wa-sqlite/dist/wa-sqlite.mjs` servido por Vite. Configurar asset serving.
3. **Worker thread**: Vite necesita bundlear el worker file por separado (`?worker` import).
4. **Microsoft/TypeScript#55689**: `node_modules/@livestore/peer-deps` → `@effect/platform` tiene peer `@effect/experimental` que pnpm puede interpretar como peer dep faltante.
5. **Strict tsconfig**: El tsconfig de livestore usa `target: ES2024`, `module: NodeNext`. Nuestro proyecto usa `ES2020`, `bundler`. Puede haber conflictos.

## Recomendación

Para MVP, mantener el enfoque actual (`invoke()`) y planificar la migración a LiveStore como un sprint dedicado cuando tengamos:
- Schema de datos estable
- Flujos de usuario definidos
- El setup de workspace linking verificado

El `livestore-sync-iroh` adapter existente se adaptará a la API v0.4.0 (cambió de `SyncBackend` a `Adapter` + `makeClientSession`).

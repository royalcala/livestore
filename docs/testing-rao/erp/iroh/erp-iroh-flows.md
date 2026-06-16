# Flujos de secuencia y comparativa

## Escenario: Compra en sucursal con internet

```
User A1          Backend A         Iroh Doc        Backend B       User B1
  │                 │                 │                │              │
  │ store.commit()  │                 │                │              │
  │────────────────►│                 │                │              │
  │                 │ tick() → HLC    │                │              │
  │                 │────────────────►│                │              │
  │    OK (local)   │ doc.setBytes()  │                │              │
  │◄────────────────│                 │─── sync ──────►│              │
  │                 │                 │                │ materialize  │
  │                 │                 │                │ filter       │
  │                 │                 │                │──────────────►│
  │                 │                 │                │  WS push     │
```

## Escenario: Compra en sucursal sin internet

```
User A1          Backend A         Iroh Doc (local)     Backend B (offline)
  │                 │                    │                     │
  │ store.commit()  │                    │                     │
  │────────────────►│                    │                     │
  │                 │ tick() → HLC       │                     │
  │                 │───────────────────►│                     │
  │    OK (local)   │                    │                     │
  │◄────────────────│                    │                     │
  │                 │                    │                     │
  │  ... 2 horas sin internet ...       │                     │
  │                 │                    │                     │
  │                 │  vuelve internet   │                     │
  │                 │───────────────────►│────── sync ────────►│
  │                 │                    │   (delta eventos)   │ materialize
  │                 │                    │◄────────────────────│ filter → User B1
```

## Escenario: Usuario remoto (fuera de sucursales)

```
User Remoto        Cloud Backend        Iroh Doc         Backend A       Backend B
  │                     │                   │                │               │
  │ conecta a cloud     │                   │                │               │
  │────────────────────►│                   │                │               │
  │                     │ subscribe         │                │               │
  │                     │──────────────────►│◄───────────────│ (sync activo) │
  │   eventos visibles  │                   │                │               │
  │◄────────────────────│ (filtrados)       │                │               │
  │                     │                   │                │               │
  │ commit              │                   │                │               │
  │────────────────────►│ tick() → HLC      │                │               │
  │                     │──────────────────►│──── sync ─────►│ materialize   │
  │                     │                   │──── sync ─────►│ materialize   │
```

## Tabla comparativa con otras arquitecturas

| Aspecto | Server central único | Cloudflare DO | LiveStore + Iroh |
|---------|---------------------|--------------|-------------------|
| **Sin internet** | No funciona | No funciona | Sucursal opera completo |
| **Latencia en sucursal** | Internet → server | Internet → CF edge | LAN (sub-ms) |
| **Costo operativo** | VPS $20+/mes | DO $5+/mes | Solo relays Iroh ($0) |
| **Cifrado de datos** | TLS, texto plano en DB | TLS, texto plano en DO | End-to-end, ni relays ven datos |
| **Permisos granulares** | Código de app | Código de app | Backend filter por usuario |
| **Escala** | Tu servidor | Límite DO | Cada sucursal suma cómputo |
| **Vendor lock-in** | Cloud provider | Cloudflare | Ninguno |
| **Multi-tenant** | Schemas/DBs | Un DO por tenant | Un Iroh Doc por tenant |
| **Backups** | Manual | Integrado | Cada nodo tiene copia completa |
| **Deploy en sucursal** | Difícil | No aplica | Un binario |

## Lo que hay que construir

| Componente | Complejidad | Dependencias |
|-----------|-------------|-------------|
| HLC | Baja | Ninguna |
| SyncBackend sobre Iroh | Media | `@n0-computer/iroh` (JS binding) |
| Filtro de permisos | Media | Modelo de roles del ERP |
| Ruteo geográfico | Baja | DNS + mDNS opcional |
| Backend empaquetable | Baja | Bun, Iroh binary |
| Provisioning de tenants | Baja | API de Iroh |
| LiveStore en backend | Nula | Ya existe |
| WebSocket server para clientes | Baja | Bun.serve |

## Riesgos y mitigaciones

| Riesgo | Mitigación |
|--------|-----------|
| **HLC collision** (dos nodos generan mismo HLC) | HLC incluye `node` ID como tiebreaker; virtualmente imposible con node IDs únicos |
| **Iroh JS binding inmaduro** | Usar Iroh como sidecar process (binario Rust) con FFI o HTTP local |
| **Doc crece indefinidamente** | Compactación: eventos > 90 días se mueven a cold storage (S3) y se borran del doc |
| **Nodo malicioso escribe eventos no autorizados** | Validación de schema en cada backend al recibir; capabilities en el doc definen quién escribe qué prefijo |
| **Divergencia de relojes entre sucursales** | HLC maneja clock skew; si es extremo (>5s), el orden total sigue siendo determinista aunque no coincida con el orden físico real |

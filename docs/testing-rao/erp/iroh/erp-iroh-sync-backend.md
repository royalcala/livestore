# SyncBackend sobre Iroh

El `SyncBackend` de LiveStore se adapta para que en vez de comunicarse vía HTTP/WebSocket con un servidor central, lea y escriba directamente del Iroh Doc de cada org. La sincronización P2P entre dispositivos la maneja Iroh transparentemente.

## Mapeo de la interfaz

| Operación SyncBackend | Sobre Iroh Doc |
|-----------------------|----------------|
| `push(batch)` | Asigna HLC a cada evento y escribe `doc.setBytes("evt:<ts>:<count>:<node>", event)` en `org_<id>/data` |
| `pull(cursor, {live})` | `doc.getMany("evt:")`, filtra por cursor. Si `live: true`, suscribe a cambios |
| `ping` | `iroh.net.ping(doc.id())` o verifica `doc.status()` |
| `isConnected` | El endpoint iroh reporta peers conectados en el swarm |

## Namespace: uno por org

Cada org tiene su propio Iroh Doc para datos transaccionales. El SyncBackend opera sobre el doc de la org activa:

```
org_acme/data/
  evt:00001781655456:00000001:a1b2c3d4...  →  { type: "invoice", hlc: {...}, payload: {...} }
  evt:00001781655457:00000001:a1b2c3d4...  →  { type: "order", ... }
  evt:00001781655458:00000002:e5f6a7b8...  →  { type: "invoice", ... }   ← de otro dispositivo
```

El prefijo `evt:` es suficiente porque el doc ya está aislado por org (`org_acme/data`). No hace falta prefijar con el org_id dentro de las keys.

## Push: escribir eventos locales

Cuando LiveStore commitea un batch de eventos:

```
1. SyncBackend recibe batch = [{name, args, seqNum, clientId, ...}]
2. Para cada evento:
   a. Genera HLC { ts: now_us, count: counter++, node: node_id_short }
   b. Serializa key: "evt:<ts>:<count>:<node>"
   c. Escribe value: { type: name, hlc: {...}, payload: args, seqNum, clientId }
   d. doc.setBytes(key, value)
3. Retorna inmediatamente (escritura local, cero latencia)
```

Como la key incluye `<node>` (dispositivo que escribe), dos dispositivos nunca colisionan en la misma key. Sin conflictos CRDT.

## Pull: leer eventos syncronizados

```
1. SyncBackend recibe cursor = { hlc_ts, hlc_count, hlc_node }
2. Itera doc.getMany("evt:").key_prefix_from("evt:", cursor_key)
3. Para cada entry encontrada:
   a. Deserializa el value JSON
   b. Emite en el stream de pull: { name, args, seqNum, clientId }
   c. Avanza el cursor
4. Si live: true, se suscribe a doc.subscribe() y emite nuevos eventos en tiempo real
```

## Cursor: HLC posicional

El cursor es el último HLC procesado. Las keys son naturalmente ordenables en el B-tree de Iroh porque el timestamp va primero, luego el contador, luego el node_id:

```
evt:00001781655456:00000001:a1b2c3d4
evt:00001781655456:00000002:a1b2c3d4   ← mismo ts, count mayor
evt:00001781655457:00000001:e5f6a7b8   ← ts mayor
```

Esto permite `key_prefix_from("evt:", cursor_key)` para saltar directamente al punto de sincronización. Nuestro fork de iroh-docs (`branch syntrix`) ya incluye el PR `key_prefix_from`.

## Evento completo (wire format)

Cada entry en Iroh Doc contiene el evento serializado más metadatos de sync:

```json
{
  "type": "invoice",
  "hlc": { "ts": 1781655456, "count": 1, "node": "a1b2c3d4e5f6a7b8" },
  "payload": { "customer_id": "cust-1", "total": 150.0 },
  "seqNum": 42,
  "clientId": "client-session-abc",
  "sessionId": "sess-xyz"
}
```

- `seqNum` y `clientId` son requeridos por LiveStore para rebase y detección de duplicados
- `hlc` es el orden total sin autoridad central
- `type` y `payload` son el evento de negocio en sí

## Seguridad en capas

El SyncBackend opera dentro de un sistema con tres capas de defensa:

| Capa | Mecanismo | Efecto |
|------|-----------|--------|
| **accept_cb** | `iroh-syntrix-docs::accept::make_accept_cb` | Bloquea handshake de dispositivos inactivos. Sin sync, sin datos. |
| **Capability** | `NamespaceSecret` del doc | Sin capability, no se abre el namespace. El doc ni siquiera se sincroniza. |
| **Validación local** | `NamespaceRegistry::can_write` | Entradas de writers no autorizados se descartan al materializar. |

El SyncBackend no necesita implementar estas capas — ya están en el stack. Solo lee y escribe del doc.

## Flujo completo

```
Dispositivo A (Alice crea invoice)
  → LiveStore commit → Eventlog SQLite + State DB SQLite
  → SyncBackend.push() → doc.setBytes("evt:...", event)
  → Iroh sync propaga a otros peers vía gossip

Dispositivo B (Admin ve invoices)
  → Iroh recibe entry nueva → doc.subscribe() emite evento
  → SyncBackend.pull() recibe el evento → lo emite como stream
  → LiveStore materializa en State DB SQLite de B
  → UI de B reacciona (query reactiva)

Dispositivo C (Bob, revocado — active: false)
  → accept_cb rechaza su handshake
  → No recibe sync de ningún namespace
  → Lo que Bob escriba localmente se queda en su disco
```

## Lo que NO hace el SyncBackend

- **No maneja autenticación** — el accept_cb y los capabilities ya lo hacen
- **No maneja autorización por tipo de evento** — la validación local ya lo hace
- **No hace retry con backoff** — LiveStore ya tiene `backgroundBackendPushing` con exponential backoff
- **No multiplexa orgs** — opera sobre el doc de la org activa. Cambiar de org = cambiar de doc

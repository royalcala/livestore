# SyncBackend sobre Iroh

El `SyncBackend` de LiveStore se adapta para que en vez de comunicarse vía HTTP/WebSocket con un servidor central, lea y escriba directamente del Iroh Doc del tenant.

## Cómo se adapta la interfaz

La interfaz `SyncBackend` define cuatro operaciones. Así se mapean a Iroh:

| Operación | En servidor tradicional | Sobre Iroh Doc |
|-----------|------------------------|----------------|
| `push(batch)` | POST HTTP al server | Asigna HLC a cada evento y escribe `doc.setBytes("eventlog/evt:<hlc>:<node>", event)` |
| `pull(cursor)` | GET HTTP o stream WS | Itera `doc.getMany("eventlog/")`, filtra por cursor, emite en stream |
| `live: true` | WebSocket al server | Suscripción a cambios en el doc: eventos nuevos emiten automáticamente |
| `ping` | Health check HTTP | `iroh.net.ping(doc.id())` |

## Cursor posicional sobre Iroh Docs

Las keys usan el formato `evt:<HLC_serializado>:<node_id>`, donde el HLC va primero. Esto hace que las keys sean **naturalmente ordenables** en el B-tree de Iroh.

La API pública actual de iroh-docs no expone `KeyFilter::PrefixFrom { prefix, cursor }` (solo `Prefix`, `Exact` y `Any`). Esto significa que para saltar al cursor, `Query::all().key_prefix("evt:")` itera desde el inicio y descarta entradas hasta llegar al offset.

**Impacto real:**

- El B-tree de Iroh **sí soporta seek posicional a bajo nivel** (`ByKeyBounds`, `RecordsBounds`) — la capacidad existe
- Para exponerla en la API pública, se necesita extender `QueryBuilder` con `key_prefix_from(prefix, cursor)`. Cambio acotado (~50 líneas en `query.rs`, `bounds.rs`, `store.rs`)
- Para **MVP con <10k eventos por tenant**, el offset client-side es aceptable: ~1-2ms extra por cada 1000 entradas salteadas

**Estrategia:** usar `offset` en MVP, contribuir `key_prefix_from` a iroh-docs para producción.

## Push: escribir eventos locales al Iroh Doc

Cuando el cliente (vía el backend) commitea eventos, el SyncBackend asigna un HLC nuevo, serializa la clave y escribe en el doc. Como dos nodos nunca escriben la misma clave (el `node_id` es parte de la key), **no hay conflictos CRDT que resolver** — el doc se usa como KV store ordenado.

## Pull: leer eventos del Iroh Doc

El cursor es el último HLC procesado. Se escanea el prefijo `eventlog/` descartando entradas anteriores al cursor. Para modo `live: true`, se suscribe a cambios en el doc y emite eventos nuevos automáticamente.

## Claves de diseño

- **El SyncBackend no hace requests HTTP**: lee y escribe localmente en el Iroh Doc. La sincronización entre nodos la maneja Iroh transparentemente
- **La latencia es cero para escrituras**: `push` escribe al doc local y retorna inmediato
- **El orden total lo da el HLC**: las keys son ordenables lexicográficamente
- **Sin conflictos CRDT**: keys únicas por diseño (HLC + node_id), no hay escrituras concurrentes a la misma key

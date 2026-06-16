# Iroh Doc como source of truth

Cada tenant recibe un `app_id` que se traduce en un Iroh Doc. El doc es un key-value store CRDT sincronizado P2P entre todos los nodos autorizados.

## Estructura del Iroh Doc

```
doc/<tenant_id>/
  │
  ├── eventlog/
  │   ├── "evt:<hlc_serializado>"  → EventEncoded (JSON)
  │   └── ...
  │
  ├── capabilities/
  │   ├── "user:<user_id>/roles"   → ["admin", "contabilidad"]
  │   ├── "user:<user_id>/scopes"  → ["sucursal-A:*", "sucursal-B:read"]
  │   ├── "role:admin/permissions" → ["event:*", "query:*"]
  │   └── ...
  │
  ├── head/
  │   ├── "head:<node_id>"         → último HLC procesado
  │   └── ...
  │
  └── nodes/
      ├── "node:backend-a"         → { url, ubicación }
      ├── "node:backend-b"         → { url, ubicación }
      └── ...
```

## Por qué Iroh Doc y no solo el eventlog en SQLite

- **CRDT nativo**: múltiples nodos escriben concurrentemente, Iroh resuelve conflictos
- **Cifrado end-to-end**: ni siquiera los Iroh relays ven los datos
- **Sync P2P**: sin servidor central, los backends se sincronizan directo o vía relay si hay NAT traversal necesario
- **Offline-first**: cada nodo tiene una copia completa del doc, escribe local y sincroniza cuando se reconecta

## Inicialización del doc por tenant

Al provisionar un nuevo tenant se crea un Iroh Doc con las entradas iniciales del sistema (tenantId, createdAt, rol admin por defecto) y se genera un ticket para compartir con nuevos nodos. Cada backend que se une al tenant usa ese ticket para obtener acceso de escritura al doc.

Las claves se organizan por prefijo (`eventlog/`, `capabilities/`, `head/`, `nodes/`) para permitir suscripciones selectivas y queries por rango.

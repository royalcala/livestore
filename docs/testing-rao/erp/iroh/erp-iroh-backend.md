# Backend empaquetable

Cada sucursal recibe un backend autocontenido que incluye LiveStore, SQLite e Iroh.

## Estructura de archivos

```
backend-sucursal/
  ├── server              # Binario Bun + WebSocket + LiveStore
  ├── iroh-sidecar        # Proceso Iroh (Rust, binario estático)
  ├── data/
  │   ├── sqlite.db       # SQLite local de LiveStore
  │   └── iroh/           # Datos de Iroh (doc replicado)
  └── config.json         # tenant_id, node_id, ticket del doc
```

## Despliegue

Una sola máquina en la sucursal (puede ser un Mini PC, Raspberry Pi, o una VM existente). El backend se distribuye como:

- **Script de instalación**: `curl | bash` que descarga el binario correcto para la arquitectura
- **Docker Compose**: para entornos que ya usan contenedores
- **Binario estático**: si se compila con Bun standalone + Iroh sidecar

## Lo que corre dentro

El backend ejecuta:

1. **Servidor WebSocket**: acepta conexiones de clientes (navegadores en la sucursal)
2. **Instancia de LiveStore**: materializa eventos en SQLite local, ejecuta queries reactivas
3. **SyncBackend sobre Iroh**: lee/escribe del Iroh Doc del tenant
4. **Filtro de permisos**: por cada cliente conectado, filtra eventos según sus roles y scopes
5. **Iroh sidecar**: maneja la sincronización P2P con otros nodos y relays

## Comunicación con el cliente

El cliente se conecta al backend vía WebSocket. Cada conexión incluye un token JWT que identifica al usuario. El backend carga los roles y scopes del usuario desde el Iroh Doc y crea un SyncBackend filtrado específico para esa conexión.

Los eventos que el cliente commitea pasan por:
1. Validación local en frontend (`canCommit` compartido)
2. Validación en backend (`canCommit` revalidado)
3. Escritura al Iroh Doc (vía `IrohWriteGate`)
4. Sincronización P2P automática a otros nodos

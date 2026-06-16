# ERP multi-sucursal: Arquitectura con LiveStore + Iroh

## Visión general

ERP multi-sucursal donde cada sucursal opera con independencia total (incluso sin internet), los datos se sincronizan P2P entre sucursales y hacia una nube central opcional, y cada usuario solo ve los eventos que sus permisos le autorizan.

```
Iroh Doc (única fuente de verdad por tenant)
  │
  ├── Sucursal A ──► Backend A (LiveStore + SQLite + filtro)
  ├── Sucursal B ──► Backend B (LiveStore + SQLite + filtro)
  └── Cloud ────────► Backend Cloud (LiveStore + PostgreSQL + filtro)
```

**Principios de diseño:**

- Cada sucursal tiene su propio backend (binario único: Bun + SQLite + Iroh sidecar)
- El Iroh Doc es la única fuente de verdad del tenant
- El event log es append-only y cifrado end-to-end
- Los backends sincronizan entre sí vía Iroh P2P (sin servidor central)
- Cada backend filtra los eventos antes de enviarlos al cliente según permisos
- Los clientes se conectan al backend más cercano geográficamente
- Sin internet = cada sucursal sigue operando con datos locales

## Documentos

| Documento | Contenido |
|-----------|-----------|
| [Iroh Doc como source of truth](./erp-iroh-iroh-doc.md) | Estructura del doc por tenant, inicialización, ventajas sobre SQLite standalone |
| [Hybrid Logical Clock (HLC)](./erp-iroh-hlc.md) | Orden total sin servidor central, reglas del HLC, comparación con otras alternativas |
| [SyncBackend sobre Iroh](./erp-iroh-sync-backend.md) | Adaptación de la interfaz de LiveStore para leer/escribir del Iroh Doc |
| [Permisos y autorización](./erp-iroh-permissions.md) | Filtrado de eventos por usuario, Policy-as-Code compartido, Query-based Authorization |
| [Ruteo geográfico](./erp-iroh-routing.md) | Cómo el cliente encuentra el backend más cercano |
| [Backend empaquetable](./erp-iroh-backend.md) | Despliegue del backend en sucursal, estructura de archivos |
| [Flujos y comparativa](./erp-iroh-flows.md) | Diagramas de secuencia, tabla comparativa con otras arquitecturas, riesgos |

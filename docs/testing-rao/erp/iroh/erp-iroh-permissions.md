# Permisos y autorización

LiveStore emite eventos. El backend recibe esos eventos vía el Iroh Doc. Antes de persistirlos, valida contra el estado actual de los datos en SQLite: ¿este evento es válido dado lo que existe en la base de datos hoy?

Mismo principio en el frontend: antes de hacer `store.commit(event)`, valida contra su SQLite local. Si pasa, commitea. Si no, feedback inmediato.

## Modelo

No hay un sistema de permisos separado. La validación es simplemente una query SQL que pregunta: "¿los datos actuales permiten esta operación?"

```
Evento: facturaUpdated { id: "inv-1", amount: 5000 }
        │
        ▼
Validator: SELECT 1 FROM invoices
           WHERE id = 'inv-1'
             AND assignedTo = '<current_user_id>'
             AND status != 'closed'
             AND sucursalId IN (<user_sucursales>)
        │
   ┌────┴────┐
   │ Row OK  │ Row no existe → rechazar
   │ → OK    │
   └─────────┘
```

### Validaciones por tipo de evento

Cada tipo de evento tiene su propia query de validación. Ejemplos:

| Evento | Query de validación |
|--------|-------------------|
| `facturaUpdated` | ¿La factura existe, el usuario es `assignedTo`, no está cerrada, pertenece a su sucursal? |
| `facturaClosed` | ¿La factura existe, el usuario es `assignedTo` o admin, está en estado válido para cerrar? |
| `pagoApproved` | ¿El pago existe, está pendiente, el monto no excede el `approvalLimit` del usuario? |
| `productoUpdated` | ¿El usuario tiene rol admin o gerente en la sucursal del producto? |
| `clienteUpdated` | ¿El cliente existe, el usuario lo creó o pertenece a su sucursal? |

Los roles y scopes del usuario no son un sistema separado — son columnas en las tablas de la BD (`users.role`, `users.approvalLimit`, `user_sucursales`) y se consultan en el JOIN de la misma query.

## Implementación

Un solo archivo con funciones puras. Frontend y backend importan el mismo código.

Cada validador recibe el evento y el store (para consultar el estado actual) y retorna `boolean`.

Para validar updates, se consulta el estado anterior y el nuevo en la misma query (el SQLite local tiene ambos porque el evento aún no se materializó).

## Frontend: feedback instantáneo

Antes de `store.commit()`, se corre la query de validación contra el SQLite local del navegador. Si falla, se muestra feedback inmediato sin esperar al backend.

Las queries se pueden suscribir reactivamente con `useLiveQuery()` para que los botones se habiliten/deshabiliten automáticamente según el estado actual de los datos.

## Backend: autoridad final

El backend recibe eventos del Iroh Doc. Corre la misma validación contra su SQLite local. Si pasa, materializa el evento. Si falla, lo descarta y loguea el intento para auditoría.

El backend es la autoridad final porque aunque el frontend valide, un cliente malicioso podría modificar el código. Pero la validación del backend usa exactamente la misma lógica, importada del mismo archivo.

## Comparación con Jazz

[Jazz](https://jazz.tools/docs/auth/permissions) define permisos con un DSL declarativo (`policy.table.allowRead.where({ column: value })`, `allowedTo.read("parent")`). Es más conciso para CRUD simple y ofrece magic columns como `$canEdit`/`$canDelete`.

Pero para reglas de negocio de un ERP — donde un pago solo se aprueba si el monto es menor al `approvalLimit` del usuario, que está en otra tabla, y la factura relacionada no está cerrada, y pertenece a la sucursal correcta — el DSL es insuficiente. Ahí SQL directo contra el estado actual de los datos es la herramienta correcta.

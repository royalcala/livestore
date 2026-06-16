# Ruteo geográfico

El cliente debe conectarse al backend más cercano para minimizar latencia y maximizar disponibilidad.

## Secuencia de resolución

1. **Backend local** (misma red): el cliente intenta descubrir un backend en la LAN vía mDNS/Bonjour o hostname predecible (`erp-<tenant>.local`)
2. **DNS geográfico**: si no hay backend local, consulta un servicio de ruteo que devuelve el backend más cercano según la ubicación del cliente (Cloudflare Workers, Fly.io, o un DNS simple)
3. **Cloud central**: fallback si ningún backend de sucursal está accesible

## Registro de backends

Cada backend publica su endpoint y ubicación en el Iroh Doc bajo el prefijo `nodes/`:

```
nodes/node:backend-a → { wsUrl: "ws://192.168.1.10:3000", location: { lat, lon } }
```

Los backends actualizan su entrada periódicamente (heartbeat). Si un backend deja de responder, se marca como offline y el ruteo lo omite.

## Sin internet

Si un usuario está en una sucursal sin internet, el cliente se conecta al backend local (LAN) y opera normalmente. Los eventos se persisten localmente en el Iroh Doc y en SQLite. Cuando vuelve internet, Iroh sincroniza los deltas automáticamente con los otros nodos.

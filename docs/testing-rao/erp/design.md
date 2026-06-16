# Diseño de red ERP

## Topología general

La red crece desde un nodo central (nuestra nube) hacia abajo. Cada tenant es una malla aislada que conecta su nodo cloud con sus sucursales. Los tenants no se ven entre sí.

```
Nuestra infraestructura
  │
  ├── Iroh Relays (públicos, solo NAT traversal, no almacenan datos)
  │
  └── Cloud Central (nuestro DC / VPS)
        │
        ├── Tenant "acme-corp"
        │     ├── Nodo Cloud (LiveStore + SQLite, siempre online)
        │     ├── Sucursal A  (backend local, Mini PC en oficina)
        │     ├── Sucursal B  (backend local, Mini PC en oficina)
        │     └── Usuarios remotos → se conectan al Nodo Cloud
        │
        ├── Tenant "globex-inc"
        │     ├── Nodo Cloud (LiveStore + SQLite, siempre online)
        │     ├── Sucursal C  (backend local)
        │     └── Usuarios remotos → Nodo Cloud
        │
        └── Tenant "initech"
              └── Nodo Cloud (mismo patrón)
```

## Aislamiento entre tenants

Cada tenant tiene su propio **Iroh Doc** independiente, identificado por `tenant_id`. Dos tenants nunca comparten doc, nunca ven los eventos del otro, nunca comparten storage.

```
Iroh Doc "acme-corp"        Iroh Doc "globex-inc"
  ├── eventlog/...            ├── eventlog/...
  ├── capabilities/...        ├── capabilities/...
  └── head/...                └── head/...
       │                            │
  Solo nodos de              Solo nodos de
  acme-corp tienen           globex-inc tienen
  el ticket de este doc      el ticket de este doc
```

El aislamiento lo garantiza Iroh: sin el ticket del doc, un nodo no puede leer ni escribir en él. El cifrado es end-to-end.

## Dentro de un tenant: malla privada

Los nodos de un mismo tenant (cloud + sucursales) forman una malla P2P donde **todos se sincronizan con todos** vía el Iroh Doc compartido. No hay jerarquía interna: la sucursal A puede sincronizar directo con la sucursal B si tienen conectividad, sin pasar por el nodo cloud.

```
Tenant "acme-corp" — Iroh Doc compartido
  │
  ├── Nodo Cloud ◄══════════════► Sucursal A
  │       │                            │
  │       └──────────────► Sucursal B ◄┘
  │
  └── Iroh Relay (si hay NAT entre sucursales)
```

**Caso sin internet en sucursal**: la sucursal A escribe localmente en el doc. Cuando recupera internet, Iroh sincroniza los eventos nuevos con los demás nodos automáticamente.

**Caso usuario remoto**: se conecta al nodo cloud (siempre online), que tiene una copia completa del doc y del SQLite del tenant. El nodo cloud le sirve eventos filtrados por permisos.

## Flujo de provisionamiento

1. Cliente contrata el servicio → se genera `tenant_id` único
2. Nuestra nube crea un Iroh Doc para ese tenant y un nodo cloud dedicado
3. Se comparte el ticket del doc con el cliente
4. El cliente instala backends en sus sucursales, cada uno configurado con `tenant_id` y el ticket
5. Cada backend se une al doc, recibe la copia completa y empieza a sincronizar

## Flujo de eventos dentro de un tenant

```
Usuario en Sucursal A
  │
  │ store.commit(evento)
  ▼
Backend Sucursal A
  │
  │ 1. Valida contra SQLite local (¿datos actuales permiten este evento?)
  │ 2. Asigna HLC
  │ 3. Escribe en Iroh Doc local
  │ 4. Materializa en SQLite local
  │
  ▼
Iroh Doc (local → sync automático)
  │
  ├──► Nodo Cloud (recibe evento, valida, materializa)
  │      │
  │      └──► Usuarios remotos conectados al cloud (filtrado por permisos)
  │
  └──► Sucursal B (recibe evento, valida, materializa)
         │
         └──► Usuarios en sucursal B (filtrado por permisos)
```

## Qué NO es esta arquitectura

- **No es una malla plana global** — los tenant están aislados
- **No es serverless** — hay un nodo cloud por tenant (para usuarios remotos y disponibilidad)
- **No es cliente↔cliente directo** — los clientes (navegadores) siempre hablan con un backend (sucursal o cloud)
- **No es multi-tenant en el mismo doc** — cada tenant tiene su propio Iroh Doc

## Por qué esta topología

| Necesidad | Cómo se resuelve |
|-----------|-----------------|
| Aislamiento de datos entre clientes | Iroh Docs separados por tenant, cifrados con tickets distintos |
| Sucursal opera sin internet | Backend local con SQLite + Iroh Doc local; sync cuando vuelve conectividad |
| Usuario remoto (fuera de sucursal) | Nodo cloud siempre online sirve eventos filtrados |
| Escala | Cada tenant nuevo = un Iroh Doc nuevo + un nodo cloud nuevo, sin afectar a los demás |
| Costo | Relays Iroh gratuitos para NAT traversal, nodos cloud ligeros (SQLite, no PostgreSQL) |

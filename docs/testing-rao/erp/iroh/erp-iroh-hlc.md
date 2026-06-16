# Hybrid Logical Clock (HLC)

Sin autoridad central, los eventos necesitan un orden total determinista para que todos los nodos procesen el event log en el mismo orden. El HLC combina reloj físico + contador lógico + ID del nodo.

## Estructura y reglas

```
HLC = { ts: number, count: number, node: string }

Reglas:
1. tick():    ts = max(prev.ts, now()), count = (ts == prev.ts ? prev.count + 1 : 1)
2. receive(): ts = max(prev.ts, remote.ts, now()), count se propaga
3. compare(): ts → count → node (tiebreaker determinista)
```

**Serialización**: el HLC se serializa como string ordenable lexicográficamente para usarse como clave en el Iroh Doc (ej: `"00001700000000001:002:suc-a"`).

## Ejemplo

```
Evento 1 (Suc A): { ts: 1700000000, count: 1, node: "suc-a" }
Evento 2 (Suc B): { ts: 1700000001, count: 1, node: "suc-b" }  → después de 1
Evento 3 (Suc A): { ts: 1700000002, count: 1, node: "suc-a" }  → después de 2
Evento 4 (Suc B): { ts: 1700000001, count: 2, node: "suc-b" }  → mismo ts que 2, mayor count

Orden total: 1 → 2 → 4 → 3
```

## Por qué HLC y no otras alternativas

| Mecanismo | Problema |
|-----------|----------|
| `Date.now()` | Relojes pueden divergir segundos entre nodos, NTP no es perfecto |
| `crypto.randomUUID()` | No hay orden, imposible saber qué evento fue primero |
| Contador secuencial | No escala a múltiples nodos (cada uno empieza en 1) |
| **HLC** | Orden total determinista, captura causalidad, tolera clock skew |

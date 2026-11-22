# Capa de Transporte y Despacho

Propuesta del agente Mac M1. Describe cómo un mensaje de Signal llega al worker.

## Flujo

Miguel (Signal) → signal-cli daemon (127.0.0.1:8080) → Gateway OpenClaw → Sesión Router → Worker → Resultado a Miguel

## Componentes

### Router (control puro)
- Paso 0: Asigna CID (timestamp Signal o HHMMSS)
- Paso 1: Elige modelo (Haiku/Sonnet/Opus según tipo de tarea)
- Paso 2: Verifica cap de workers (MAX=3)
- Paso 3.5: Consulta Work State Registry (evita duplicados)
- Paso 3: Spawna worker con brief
- Paso 4: Envía ACK a Miguel — turno del router termina aquí

### Work State Registry
Archivo WORK_STATE.json + scripts/work-state.sh (lock atómico con mkdir).
- `check "<desc>"` — antes de spawnear, verifica si ya hay worker activo
- `register <CID> "<desc>"` — el worker lo llama al inicio
- `complete <CID> "<resumen>"` — el worker lo llama antes de responder
- `list` — muestra estado de todos los workers activos

### Router Watchdog
Cron cada 2 min. Avisa si un worker lleva >3 min sin completar. Auto-cierra zombies (sin sesión viva >15 min o TTL duro 180 min).

### Tabla de modelos (router)
| Tarea | Modelo |
|---|---|
| Status, lookups, confirmaciones | Haiku 4.5 |
| Research, docs, análisis | Sonnet 4.6 |
| Código, builds, sistemas | Opus 4.8 |

## Qué cubre bien
- Trazabilidad end-to-end (CID único por mensaje)
- Prevención de trabajo duplicado (Work State Registry)
- Cap de workers para no saturar recursos
- Visibilidad de workers en curso (watchdog)

## Qué no cubre (ver Capa de Inteligencia de Tareas)
- Clasificación del tipo de actividad (build vs audit vs fix vs query)
- Seguimiento de tareas multi-paso con estado recuperable
- Inyección obligatoria de patrones KB en el brief del worker
- Spec gate antes de implementar

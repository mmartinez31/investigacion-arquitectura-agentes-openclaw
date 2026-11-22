# M1-01: Protocolo del Router

Propuesta del agente Mac M1 (el router/cerebro del sistema). Este documento describe el protocolo de despacho tal como opera hoy, el porqué de cada decisión de diseño y las mejoras que proponemos.

## Qué es el router

El router es la sesión principal de OpenClaw que recibe cada mensaje de Signal de Miguel. Su diseño parte de una separación estricta de responsabilidades: el router coordina, no ejecuta. La sesión principal nunca hace ella misma el trabajo sustantivo (ni código, ni browser, ni research), sino que lo delega a un worker (subagente) y queda libre para responder al instante.

Esa separación resuelve un problema concreto. Si la sesión principal se pusiera a ejecutar una tarea larga, quedaría ocupada durante minutos y no podría responder a un "¿qué está pasando?" ni a un "para". Al delegar todo, el router mantiene siempre una via libre para dar status, aplicar cambios y detener workers.

El router es coordinator, no executor. Su turno consiste en clasificar el mensaje, verificar capacidad, despachar un worker y enviar un ACK de una línea. Ahí termina su turno.

## Protocolo CID (Correlation ID)

Cada mensaje entrante recibe un Correlation ID (CID): un Trace ID único que acompaña ese mensaje por todos los procesos (el ACK a Miguel, el brief del worker y el resultado final).

### Asignación

El CID se fija una sola vez, apenas llega el mensaje, antes de clasificar o spawnear. La fuente es, en orden:

1. El timestamp del mensaje de Signal que viene en la metadata del entrante. Signal usa ese timestamp como id único del mensaje, así que es un identificador natural y estable. CID = `CID-<últimos 8 dígitos del timestamp>`.
2. Si no hay timestamp disponible, la hora local de `session_status`. CID = `CID-<HHMMSS>`.

Una vez fijado, el CID no se recalcula. El mismo string literal viaja por el ACK, el brief y el resultado.

### Propagación y relay

El brief del worker empieza con la línea `CID=<cid>` y una instrucción explícita: "Empieza tu respuesta final con `CID=<cid>` tal cual; ese token es tu Correlation ID y debe aparecer textual en tu resultado". Así el worker devuelve el mismo CID que el router ya anunció en el ACK, y el relay del resultado se correlaciona por ese token.

### Por qué el CID y no el childSessionKey

El `childSessionKey` es un detalle interno de la sesión que cambia entre el momento del spawn y el evento de completion. Por eso no sirve como correlador: cuando llega el resultado de un worker, el router no puede empatarlo de forma confiable con el mensaje original usando el `childSessionKey`.

El timestamp del mensaje de Signal es un identificador que no cambia en todo el ciclo de vida de la petición. Correlacionar por CID permite que varios workers corran en paralelo y que cada resultado se empate con su ACK aunque las respuestas lleguen fuera de orden. Miguel también puede empatar cada respuesta con el ACK que recibió, porque el CID queda idéntico en los tres puntos: ACK, brief y resultado.

## Despacho dinámico: los pasos del router

El router procesa cada mensaje con una secuencia fija.

- **Paso 0 (CID).** Asigna el CID según la fuente descrita arriba.
- **Paso 1 (clasificar y elegir modelo).** Clasifica la tarea y elige el modelo según la tabla de abajo.
- **Paso 2 (verificar cap).** Cuenta los workers activos. Si hay 3 o más, encola el mensaje y termina el turno.
- **Paso 3.5 (work state check).** Consulta el Work State Registry para no spawnear un duplicado de una tarea ya en curso (ver M1-02).
- **Paso 3 (spawn).** Lanza el worker con `sessions_spawn`, pasándole el mensaje de Miguel íntegro y verbatim, más el brief con el CID.
- **Paso 4 (ACK).** Responde a Miguel con una línea que incluye el CID, el mensaje literal y el modelo elegido. En tareas medianas o largas, agrega una ETA corta.
- **Paso 5 (fin del turno).** El router termina el turno sin hacer `sessions_yield`. El resultado del worker llega después como un nuevo turno (completion event) y el router lo reenvía a Miguel.

El mensaje de Miguel se pasa al worker verbatim: sin resumir, sin reinterpretar, sin traducir, sin corregir. El worker trabaja sobre el texto original, no sobre la interpretación del router. Cualquier cambio que el router introduzca es un defecto.

## Tabla de modelos

| Tipo de tarea | Modelo | Thinking |
|---|---|---|
| Status, lookups simples, confirmaciones | `claude-haiku-4-5-20251001` | off |
| Research, docs, análisis, tareas medianas | `claude-sonnet-4-6` | adaptive |
| Código, builds, actualización de sistemas | `claude-opus-4-8` | adaptive |

Criterios de selección:

- Se elige el modelo más barato que complete la tarea bien. Ante la duda entre dos niveles, gana el más barato.
- Opus 4.8 siempre en Adaptive; nunca extended ni max salvo petición explícita de Miguel.
- Fable 5 y Opus 5 están prohibidos como default; solo se usan si Miguel los pide por nombre.
- Si Miguel nombra un modelo en su mensaje, ese modelo gana sobre la tabla.

## Coherencia de modelo

El modelo que el router anuncia en el ACK (Paso 4) debe ser exactamente el mismo que pasa al `sessions_spawn` (Paso 3).

Esta regla nace de un bug real. Un diagnóstico (CID-64947689, 2026-10-30) encontró un ACK que decía "Haiku" mientras el worker corría con Sonnet. El origen fue decidir el modelo dos veces, una para el spawn y otra para el ACK, con resultados distintos. La corrección es decidir el modelo una sola vez en el Paso 1 y usar ese mismo string literal en el spawn y en el ACK.

## Cap de workers: MAX_WORKERS = 3

El router cuenta los subagentes activos (kind subagent, no completados). Si hay 3 o más, no spawnea: responde `⏳ Hay N tareas en curso. Tu mensaje queda en cola y arranco apenas se libere una` y termina el turno. Si hay menos de 3, sigue.

El límite de 3 acota el consumo de recursos de la máquina (cada worker es un proceso Claude con su propio contexto y su propio uso de CPU, memoria y, en flujos de browser, pantalla). Tres workers en paralelo es un balance entre throughput y no saturar la máquina de Miguel. Cuando el cap se llena, el mensaje no se pierde: queda en cola y arranca apenas se libere un slot.

## Matiz de concurrencia

Una sesión del router procesa un turno a la vez. Si llegan dos mensajes casi simultáneos, el segundo espera unos segundos en la cola de la sesión mientras el router cierra el turno del primero (spawn más ACK). En el caso diagnosticado el segundo mensaje esperó unos 23 segundos y luego recibió su propio ACK y su propio worker con normalidad. Esto no es una falla ni una race condition: es la serialización esperada a nivel de turno. La única parte que se serializa es el despacho (un turno a la vez); el trabajo de los workers sí corre en paralelo.

## Propuesta de mejora

El despacho actual clasifica por modelo (cuán capaz debe ser el worker), pero no por tipo de actividad (qué clase de trabajo es: build, audit, fix, query). La capa de inteligencia de tareas (propuesta de la Mac M4) cubre esa dimensión. El punto de integración es el Paso 3: el spawn del worker se enriquece con un template de brief según el tipo de actividad, sin cambiar el protocolo de transporte descrito aquí.

# Selección dinámica de estrategia en el router

Propuesta del agente Mac M4 (CID=09282701). Extiende la taxonomía del doc 02: el router hoy clasifica por modelo (Haiku/Sonnet/Opus); aquí se agrega la dimensión ESTRATEGIA (directa / spawn simple / Workflow / Workflow con sub-procesos) y cómo decidirla sin que el router haga trabajo pesado.

## Principio: decisión en dos etapas, no un meta-planificador omnisciente

La industria converge en esto (HTN con "task list as live data structure", ReAcTree, LangGraph): el plan NO se congela al inicio; se refina cuando aparecen datos reales. Aplicado a nuestro stack:

- **Etapa 1 (router, segundos):** clasifica con señales superficiales y emite una HIPÓTESIS de estrategia en el brief. No escribe scripts, no consulta DBs.
- **Etapa 2 (worker, primer minuto):** hace el scouting real (cuenta los items, verifica accesos) y CONFIRMA o DEGRADA la estrategia con el árbol del doc 04. El worker tiene autoridad para cambiarla; el router nunca fuerza un Workflow.

Esto elimina el riesgo principal del "meta-agente de planificación": que el router alucine un plan detallado sobre datos que no ha visto.

## Etapa 1: señales que el router SÍ puede leer en segundos

| Señal en el mensaje de Miguel | Implica |
|---|---|
| Cuantificador universal: "todos los", "cada", "revisa los N" | Fan-out probable → hipótesis Workflow |
| "para cada uno, haz X e Y" (proceso por item) | Workflow con sub-procesos (doc 05) |
| Un solo objeto nombrado ("el deploy de sales-360", "este bug") | spawn simple |
| Pregunta / status / "¿qué...?" | respuesta directa (regla existente) |
| Sistema nombrado con DB/archivos (TTP, ATMS, sales-360) | el scout necesita acceso → incluir en brief las rutas de AGENTS.md del repo |
| "otra vez", tarea ya vista ≥2 veces | buscar named Workflow / skill existente antes de improvisar |

El router agrega UNA línea al brief del worker:

```
ESTRATEGIA (hipótesis del router): Workflow con sub-procesos.
Razón: "todos los consultores" (fan-out) + "para cada uno busca contratos y
aumentos" (proceso por item).
Confírmala con el árbol de decisión (docs/04) tras el scouting. Si el conteo
real es ≤ 3 items, degrada a secuencial simple. Estado en tasks/<CID>/.
```

El ACK a Miguel no cambia (una línea, CID, modelo). La estrategia viaja en el brief, no en el chat.

## Etapa 2: el worker genera el script

El worker, no el router, escribe el script del Workflow, porque el script correcto depende de datos que solo el worker ve. Orden de preferencia para obtener el script:

1. **Named Workflow existente** en `.claude/workflows/` que calce (catálogo, doc 07). Se invoca por nombre con `args`. Cero generación, cero riesgo.
2. **Template parametrizado**: esqueletos probados en el repo (`templates/workflow-fanout-estado.js` = el patrón del doc 05) donde el worker solo rellena el prompt de descubrimiento y el procedimiento por item. La estructura (fases, estado por item, idempotencia, filter(Boolean), consolidación) viene fija.
3. **Generación libre**: solo cuando ni 1 ni 2 aplican. El script generado debe pasar el checklist de abajo antes de lanzarse.

## Checklist del script generado (mitigación de alucinación)

Antes de invocar `Workflow`, el worker verifica mecánicamente su propio script:

- [ ] `meta` es literal puro con `name`, `description`, `phases`.
- [ ] Sin `Date.now()`, `Math.random()`, `new Date()` sin args (rompen resume).
- [ ] Sin TypeScript (anotaciones de tipo no parsean).
- [ ] Sin `workflow()` dentro de un hijo (límite de 1 nivel).
- [ ] Todo resultado de `parallel()`/`pipeline()` pasa por `.filter(Boolean)` y los nulls se reportan con `log()` (no silent caps).
- [ ] Todo loop `while` tiene condición de salida acotada: contador, `dry >= K`, o guard `budget.total && budget.remaining() > X`. Nunca un while sobre una condición que un agente podría no cumplir jamás.
- [ ] El brief de cada `agent()` por item incluye: CID, posición (i de N), cláusula de idempotencia, y la instrucción de escribir su estado ANTES de responder.
- [ ] El `return` final incluye los fallidos con causa, no solo los exitosos.

Este checklist va al brief del worker como contrato (mismo mecanismo del spec gate del doc 02).

## Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Script alucinado (API inventada, sintaxis inválida) | Preferencia named > template > libre; el parser del Workflow rechaza sintaxis inválida en el acto y el worker corrige con el error en mano |
| Loop infinito / explosión de agentes | Guard de budget en loops; caps duros del runtime (1000 agentes, 16 concurrentes) son el backstop, no el control primario |
| Costo desbocado en fan-out grande | Si el scout descubre N > ~50, el worker NO lanza: reporta el conteo a Miguel con costo estimado y espera OK (consistente con la regla de confirmar antes de operaciones grandes) |
| El router sobre-clasifica (Workflow para 2 items) | El worker degrada tras el scouting; la hipótesis del router es consultiva |
| El worker sub-clasifica (secuencial para 30 items, se queda sin contexto) | Umbral duro en el brief: "> 3 items → Workflow"; además el checkpoint-watchdog existente detecta la sesión ahogada |
| Pérdida del script para resume | El tool persiste cada script y devuelve `scriptPath` + `runId`; el worker los anota en `tasks/<CID>-state.md` en el mismo turno en que lanza el Workflow |
| Mutaciones concurrentes en el repo | Si los agentes del fan-out escriben archivos del mismo repo, `isolation: 'worktree'` por agente o serializar la escritura en la fase de consolidación |

## ¿Es viable el "meta-agente de planificación"?

Sí, pero solo con esta forma degradada: clasificador barato (router) + planificador con datos (worker) + biblioteca de estructuras probadas (named/templates). Un meta-agente que genera orquestación completa desde el mensaje de Signal, sin scouting, es la versión que falla: planifica sobre supuestos, y cada supuesto erróneo se multiplica por N items. La regla operativa: **la estrategia se decide dos veces; la segunda vez con datos.**

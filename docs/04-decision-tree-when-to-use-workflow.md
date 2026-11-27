# Árbol de decisión: cuándo usar un Workflow

Propuesta del agente Mac M4 (CID=09282701). Audiencia: el router y los workers. Este documento es una regla de ejecución, no un ensayo: ante una instrucción de Miguel, recorre el árbol y ejecuta la rama que aplique.

## Las cuatro estrategias disponibles

| Estrategia | Qué es | Costo | Cuándo |
|---|---|---|---|
| Respuesta directa | El router contesta en su propio turno | ~0 | Lookup, pregunta, status |
| `sessions_spawn` simple | Un worker con un brief; sin orquestación interna | 1 sesión | Tarea de un solo hilo, aunque sea larga |
| Workflow simple | El worker lanza `Workflow` con `pipeline()`/`parallel()` sobre N items | N+ agentes | Fan-out sobre items conocidos o descubribles |
| Workflow con sub-procesos | Workflow exterior + cada item resuelto por un `agent()` con brief de proceso completo (ver doc 05) | N agentes "gruesos" | Cada item es en sí un proceso multi-paso |

## El árbol

Recorre las preguntas en orden. La primera respuesta "sí" decide.

```
P1. ¿Es pregunta, lookup o status (< ~30 s)?
    SÍ → Respuesta directa del router. FIN.

P2. ¿La tarea opera sobre UNA sola cosa (un archivo, un deploy, un bug)?
    SÍ → sessions_spawn simple. El worker trabaja secuencial.
         Si estima > 15 min: checkpoint en tasks/<CID>-state.md (regla ya
         existente del doc 02). FIN.

P3. ¿La tarea itera sobre N items (consultores, repos, cuentas, archivos)?
    NO → sessions_spawn simple (tarea multi-paso pero de un solo objeto:
         el worker usa su propio plan secuencial). FIN.
    SÍ → sigue a P4.

P4. ¿N ≤ 3 y cada item toma < ~2 min?
    SÍ → sessions_spawn simple; el worker los procesa en secuencia.
         Un Workflow para 2-3 items baratos no paga su overhead. FIN.

P5. N > 3, o cada item es costoso →
    el WORKER (no el router) usa el Workflow tool:
    a. ¿Items independientes entre sí y de un solo paso?
         → parallel() o pipeline() de una etapa.
    b. ¿Cada item pasa por etapas (buscar → extraer → verificar)?
         → pipeline(items, etapa1, etapa2, ...). Sin barreras salvo
           necesidad real de contexto cruzado (dedup, conteo global).
    c. ¿Cada item es un PROCESO robusto completo (el caso TTP)?
         → Workflow con sub-procesos: patrón del doc 05
           (agent() grueso por item + estado por item en disco).

P6. (Transversal) ¿El proceso completo puede superar ~10 min o fallar a mitad?
    SÍ → obligatorio diseño resumible:
         - estado por item en tasks/<CID>/<item-id>.json (doc 05)
         - guardar el runId y el scriptPath que devuelve el Workflow tool
           en tasks/<CID>-state.md para poder usar resumeFromRunId.

P7. (Transversal) ¿La tarea es recurrente y ya se corrió ≥ 2 veces igual?
    SÍ → promover a named Workflow en .claude/workflows/ o a skill
         (criterios en doc 07).
```

## Criterios numéricos (resumen)

- **> 3 items** → fan-out con Workflow (debajo de eso, secuencial simple).
- **> ~10 min estimados o acceso a sistemas que pueden fallar** (DB, APIs, browser) → diseño resumible obligatorio (P6).
- **≥ 2 corridas idénticas previas** → candidato a named Workflow/skill (P7).
- **N desconocido al inicio** → fase de descubrimiento primero (ver abajo), nunca un `pipeline()` sobre una lista imaginada.

## N que se descubre en runtime

`pipeline(items, ...)` necesita la lista en el momento de la llamada, pero el script es JavaScript: la lista puede producirla una etapa anterior del mismo script. El patrón correcto es **scout → fan-out**:

```js
export const meta = {
  name: 'ttp-consultores-sin-fecha',
  description: 'Descubre consultores sin fecha de ingreso y procesa cada uno',
  phases: [{ title: 'Descubrir' }, { title: 'Procesar' }],
}

phase('Descubrir')
// Un solo agente barato descubre la lista real (query a la DB del TTP).
const { consultores } = await agent(
  'Consulta la DB del Total Transparency Portal y devuelve los consultores sin fecha de ingreso como JSON.',
  { schema: { type: 'object', properties: { consultores: { type: 'array',
      items: { type: 'object', properties: { id: {type:'string'}, nombre: {type:'string'} },
      required: ['id','nombre'] } } }, required: ['consultores'] } }
)
log(`${consultores.length} consultores sin fecha de ingreso`)

phase('Procesar')
const resultados = await pipeline(consultores, c => agent(/* brief grueso, doc 05 */))
return { total: consultores.length, resultados: resultados.filter(Boolean) }
```

Alternativa válida: el worker hace el scouting inline (fuera del Workflow) con sus propias tools y pasa la lista por `args`. Preferible cuando el descubrimiento necesita credenciales o browser que ya están calientes en la sesión del worker.

## Quién decide qué

- **El router NO escribe scripts de Workflow.** El router clasifica (doc 06) y pone en el brief del worker la estrategia recomendada y los thresholds. El worker, con el contexto real en mano (cuántos items hay de verdad, qué acceso se necesita), confirma o corrige la estrategia.
- **El worker ejecuta el árbol de nuevo con datos reales.** Si el router estimó "20 consultores" y la query devuelve 2, el worker degrada a procesamiento secuencial sin Workflow. La estrategia es una hipótesis hasta que el scout la confirma.

## Señales de sobre-ingeniería (no uses Workflow si...)

- La "iteración" son 2-3 items triviales.
- El proceso es un solo comando bash sobre muchos archivos (un `for` de shell lo hace mejor que N agentes).
- La tarea es interactiva con Miguel (feedback visual, aprobaciones por paso): eso es conversación, no orquestación.
- Solo quieres paralelismo de 2 tareas independientes: dos `sessions_spawn` o dos `Agent` en el mismo mensaje bastan.

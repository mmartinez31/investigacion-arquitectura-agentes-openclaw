# Patrón para "workflow de workflows" con límite de un nivel

Propuesta del agente Mac M4 (CID=09282701). Resuelve: cómo componer procesos robustos anidados cuando el Workflow tool de Claude Code solo permite UN nivel de anidamiento (`workflow()` dentro de un child lanza error).

## Hechos del Workflow tool (verificados contra la spec de la herramienta)

1. **Anidamiento = 1 nivel.** Un script puede llamar `workflow(nameOrRef, args)` para correr un workflow hijo inline; el hijo NO puede llamar `workflow()` a su vez. El hijo comparte cap de concurrencia, contador de agentes, señal de abort y budget de tokens con el padre.
2. **Resume por journal de prefijo.** `Workflow({scriptPath, resumeFromRunId})` re-usa la caché del prefijo más largo de llamadas `agent()` sin cambios (mismo prompt + mismas opts). Primera llamada editada o nueva en adelante corre en vivo. Mismo script + mismos args = 100% cache hit. Por eso `Date.now()`/`Math.random()` están prohibidos en el script: romperían la igualdad de prompts.
3. **Fallas por item, no por lote.** Un stage de `pipeline()` que lanza excepción deja ese item en `null` y salta sus etapas restantes; los demás items siguen. Un thunk de `parallel()` que falla resuelve a `null`; la llamada nunca rechaza. Un `agent()` cuyo subagente muere con error terminal devuelve `null`. Conclusión: `.filter(Boolean)` + registro de los nulls es obligatorio (regla "no silent caps").
4. **Caps:** concurrencia min(16, cores-2) por workflow; 1000 agentes de por vida; 4096 items por llamada a pipeline/parallel.
5. **Budget compartido:** `budget.remaining()` cubre main loop + todos los workflows del turno; al llegar al total, `agent()` lanza.

## La regla central: el segundo nivel es un agente, no un workflow

El impulso natural es: "workflow exterior itera consultores, workflow interior procesa un consultor". Eso choca con el límite de 1 nivel y además es innecesario. La composición correcta:

- **Nivel 1 (orquestación):** un único Workflow. Dueño del fan-out, del journal, del budget y de las fases.
- **Nivel 2 (proceso por item):** un `agent()` "grueso": un subagente con un brief que contiene el proceso COMPLETO del item (buscar contratos, buscar aumentos, validar, escribir estado). Un subagente de Claude Code es un agente con todas sus tools: puede leer archivos, consultar la DB, llamar APIs y ejecutar su propio plan interno de 10 pasos. No necesita ser un workflow para ser robusto; necesita un brief que sea un procedimiento.
- **Nivel 3 (si existiera):** no existe como orquestación. Si el proceso del item tiene a su vez sub-partes paralelizables pesadas, eso es señal de que el fan-out está mal particionado: aplana (ver "Aplanar la jerarquía" abajo).

Esto coincide con el patrón de la industria: en Temporal, los child workflows son unidades reutilizables de UN nivel de descomposición y la recomendación es no anidar sagas profundas; en LangGraph, los subgraphs heredan el checkpointer del padre en vez de tener orquestación propia; en HTN, las tareas compuestas se descomponen hasta primitivas y la primitiva aquí es "un subagente con un procedimiento".

## El hilo de estado: archivos por item, no contexto

El journal del Workflow cachea por (prompt, opts) y solo sobrevive dentro de la sesión. Para robustez real entre sesiones y para visibilidad, el estado vive en disco:

```
tasks/<CID>/
  _manifest.json          # lista descubierta + runId + scriptPath + timestamp
  <item-id>.json          # resultado por item: {status, data, error, evidencia}
```

Reglas:

- **El agente del item escribe su propio archivo de estado como ÚLTIMO paso de su proceso** (análogo a la regla work-state.sh: completar ANTES de redactar la respuesta final).
- **El script del Workflow trata el archivo como idempotencia:** el brief del agente empieza con "si `tasks/<CID>/<id>.json` existe con status=done, no repitas el trabajo; devuelve su contenido". Así el resume funciona por DOS vías independientes: el journal del Workflow (caché de prefijo) y el estado en disco (si el journal se perdió o el script cambió).
- **Estado pequeño, punteros no payloads** (lección LangGraph): el JSON guarda rutas e IDs, no los PDFs ni los dumps. Kilobytes, no megabytes.

## Ejemplo completo: TTP, 20 consultores sin fecha de ingreso

Instrucción original: "revisa todos los consultores que no tengan fecha de ingreso, y para cada uno busca los contratos y aumentos".

```js
export const meta = {
  name: 'ttp-consultores-contratos-aumentos',
  description: 'Para cada consultor sin fecha de ingreso: contratos + aumentos, con estado por item',
  phases: [
    { title: 'Descubrir', detail: 'query de consultores sin fecha de ingreso' },
    { title: 'Procesar', detail: 'un agente por consultor: contratos + aumentos' },
    { title: 'Consolidar', detail: 'reporte + items fallidos' },
  ],
}

// args = { cid: '09282701', stateDir: 'tasks/09282701' }  -- el worker los pasa
const { cid, stateDir } = args

phase('Descubrir')
const { consultores } = await agent(
  `Consulta el Total Transparency Portal (lee su AGENTS.md para el acceso a la DB)
   y lista los consultores SIN fecha de ingreso. Escribe la lista en
   ${stateDir}/_manifest.json y devuélvela como JSON.`,
  { schema: {
      type: 'object',
      properties: { consultores: { type: 'array', items: { type: 'object',
        properties: { id: { type: 'string' }, nombre: { type: 'string' } },
        required: ['id', 'nombre'] } } },
      required: ['consultores'] } }
)
log(`Descubiertos ${consultores.length} consultores sin fecha de ingreso`)

phase('Procesar')
const procesados = await pipeline(
  consultores,
  (c, _orig, i) => agent(
    `CID=${cid}. Ítem ${i + 1}/${consultores.length}: consultor ${c.nombre} (id=${c.id}).

     IDEMPOTENCIA: si ${stateDir}/${c.id}.json existe con "status":"done",
     NO repitas el trabajo: lee el archivo y devuelve su contenido.

     PROCESO (completo, en orden):
     1. Busca TODOS los contratos del consultor (DB del TTP + carpeta de
        contratos; el AGENTS.md del repo TTP documenta ambas fuentes).
     2. Busca el historial de aumentos salariales.
     3. Si una fuente no responde, intenta la alternativa antes de marcar error.
     4. Escribe ${stateDir}/${c.id}.json con:
        { "status": "done" | "error", "consultor": {...},
          "contratos": [{ruta, fecha, tipo}], "aumentos": [{fecha, monto}],
          "faltantes": [...], "error": "..." }
        Guarda RUTAS a los documentos, no su contenido.
     5. SOLO después de escribir el archivo, devuelve el mismo JSON.`,
    { label: `consultor:${c.id}`, phase: 'Procesar',
      schema: { type: 'object', properties: {
        status: { type: 'string', enum: ['done', 'error'] } },
        required: ['status'] } }
  )
)

phase('Consolidar')
const ok = procesados.filter(r => r && r.status === 'done')
const fallidos = consultores.filter((c, i) => !procesados[i] || procesados[i].status !== 'done')
if (fallidos.length) log(`FALLIDOS (${fallidos.length}): ${fallidos.map(f => f.id).join(', ')}`)

const resumen = await agent(
  `Lee todos los JSON de ${stateDir}/ y redacta el reporte final para Miguel:
   tabla consultor / # contratos / # aumentos / faltantes, y la lista de items
   fallidos con su causa. Español, sin em dash.`,
  { phase: 'Consolidar' }
)
return { total: consultores.length, ok: ok.length, fallidos: fallidos.map(f => f.id), resumen }
```

### Qué pasa si falla en el consultor 7 de 20

- **Falla el item 7, los demás siguen:** `pipeline()` deja ese item en `null`; la fase Consolidar lo reporta como fallido con causa. No se pierde el lote.
- **Se muere el Workflow entero (crash, abort, output limit):** dos caminos de recuperación, en orden:
  1. `Workflow({scriptPath, resumeFromRunId})`: los `agent()` ya completados vuelven de caché al instante (mismo prompt/opts), solo corren los pendientes.
  2. Si no hay journal (otra sesión, script editado): se relanza el Workflow normal. La cláusula de IDEMPOTENCIA hace que los consultores con `<id>.json` status=done cuesten un turno corto de verificación en vez del proceso completo.
- **Reintento selectivo de fallidos:** relanzar el mismo script; los done salen por idempotencia, los error se reprocesan porque su archivo no dice done.

## Aplanar la jerarquía (cuando el item "necesitaría" su propio workflow)

Si el proceso por item es tan grande que un solo subagente se queda sin contexto (p.ej. "por cada consultor, por cada contrato, extraer 15 campos y verificar firmas"), NO anides: aplana el fan-out al grano más fino.

```js
// En vez de: pipeline(consultores, consultor => workflow('procesar-contratos', ...))  // PROHIBIDO (nivel 2)
// Aplana: descubre los pares (consultor, contrato) y haz pipeline sobre ESO.
phase('Descubrir contratos')
const pares = (await pipeline(consultores,
  c => agent(`Lista los contratos de ${c.id}; solo rutas e IDs.`, { schema: LISTA })
)).filter(Boolean).flatMap((r, i) => r.contratos.map(k => ({ consultor: consultores[i], contrato: k })))

phase('Extraer')
await pipeline(pares, p => agent(/* proceso de UN contrato, con estado en
  tasks/<CID>/<consultor>/<contrato>.json */))
```

La jerarquía conceptual (consultor → contrato) se conserva en el ESTADO (estructura de carpetas) y en la consolidación, no en la orquestación. Un solo nivel de workflow puede expresar cualquier árbol si el fan-out itera sobre las hojas.

## Cuándo sí usar `workflow()` (el único nivel de anidamiento)

Reservar `workflow(nameOrRef, args)` para invocar un **named Workflow probado** como fase de un workflow mayor (p.ej. `workflow('find-consultant-files', {id})` dentro de un proceso de auditoría), sabiendo que: comparte budget y cap con el padre, y sus hijos no pueden anidar más. Es composición de bloques de catálogo (doc 07), no un mecanismo de recursión.

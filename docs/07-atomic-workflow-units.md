# Unidades atómicas: named Workflow vs Skill vs agent() inline

Propuesta del agente Mac M4 (CID=09282701). Responde: para un sub-proceso robusto como "buscar los archivos de un consultor", ¿qué forma debe tener el bloque reutilizable?

## Los tres recipientes

| Recipiente | Qué es | Quién lo ejecuta | Estado/resume propio |
|---|---|---|---|
| **agent() inline** | Un brief escrito en el momento dentro de un script o un spawn | Un subagente efímero | No (solo el estado en disco que su brief le ordene) |
| **Skill (SKILL.md)** | Un playbook versionado que cualquier agente lee y sigue | El agente que la invoca, en su propio contexto | No (el procedimiento puede ordenar checkpoints) |
| **Named Workflow (`.claude/workflows/*.js`)** | Orquestación determinista versionada, invocable por nombre o con `workflow(name, args)` | El runtime del Workflow tool | Sí (journal + resumeFromRunId + budget + fases) |

Analogía con la industria: agent() inline = activity de Temporal (unidad de trabajo, sin orquestación); Skill = runbook/procedimiento; named Workflow = child workflow de Temporal (unidad de ORQUESTACIÓN reutilizable con historia propia).

## Criterio de selección

Pregunta en orden; la primera que aplica decide:

```
1. ¿El bloque ORQUESTA varios agentes (fan-out interno, fases, loop-until-dry,
   budget propio)?
     SÍ → named Workflow. Es el único recipiente con esas primitivas.
     Ejemplos: 'review-changes', 'ttp-consultores-contratos-aumentos' (doc 05).

2. ¿El bloque es un PROCEDIMIENTO que un solo agente sigue (pasos, fuentes,
   formatos, trampas conocidas) y se repite entre tareas distintas?
     SÍ → Skill. El conocimiento se versiona una vez y lo consume cualquier
     agente: el worker directo, un agent() de un pipeline, o un subagente
     spawneado. Ejemplos existentes: contact-intel-apollo-linkedin-lusha,
     upwork-job-hunting.

3. ¿Es una sub-tarea de un solo propósito, específica de ESTA corrida?
     SÍ → agent() inline con brief completo. No contamines el catálogo con
     bloques de un solo uso.
```

Regla de promoción (misma del doc 04, P7): un agent() inline cuyo brief copias por segunda vez se convierte en Skill; una estructura de orquestación que copias por segunda vez se convierte en named Workflow. La primera corrida siempre puede ser inline: el catálogo se construye por extracción de cosas que funcionaron, no por diseño anticipado.

## El caso "buscar archivos de un consultor" (TTP)

Respuesta concreta: **Skill + agent() inline, NO named Workflow.**

- El proceso (dónde está la DB, qué carpetas de contratos existen, cómo se llaman los archivos, qué hacer si falta el documento) es conocimiento procedimental de UN agente trabajando solo → `SKILL.md` (p.ej. `ttp-consultant-files`) con las fuentes, el orden de búsqueda y el formato del JSON de estado.
- El fan-out sobre los 20 consultores vive en el Workflow exterior (doc 05); cada `agent()` del pipeline recibe un brief corto: "sigue la skill ttp-consultant-files para el consultor X; estado en tasks/<CID>/<id>.json".
- Convertirlo en named Workflow sería incorrecto: no orquesta nada (es un agente leyendo fuentes en secuencia) y como `workflow()` hijo quemaría el único nivel de anidamiento disponible, impidiendo que el proceso padre lo use dentro de otra composición.

La skill además corrige el problema de duplicación: si el procedimiento vive inline en el script del Workflow, cada ajuste (una carpeta nueva de contratos) obliga a editar scripts; en la skill se corrige una vez.

## Granularidad correcta de un bloque

Señales de que el bloque está bien cortado (coinciden con la guía de child workflows de Temporal y con la primitiva de HTN):

- **Un sustantivo + un verbo:** "buscar archivos de un consultor", "generar dossier de una cuenta". Si el nombre necesita "y" ("buscar archivos Y actualizar la DB Y notificar"), son dos bloques o es orquestación.
- **Entrada/salida serializables y pequeñas:** IDs y rutas entran, JSON corto sale. Si necesita recibir "todo el contexto de la conversación", no es un bloque atómico.
- **Idempotente:** correrlo dos veces con la misma entrada es seguro (verifica su archivo de estado antes de trabajar). Esto es lo que hace componible al bloque dentro de cualquier resume.
- **Falla contenida:** reporta `{status: "error", causa}` en vez de tumbar al orquestador. El orquestador decide reintento/compensación (patrón saga: la compensación es del nivel de arriba, no del bloque).
- **Sin decisiones de escala:** el bloque no decide cuántos de sí mismo corren; eso es del orquestador (budget, caps, confirmación de Miguel para N grande).

Señal de grano demasiado fino: el brief + el overhead de spawn superan el trabajo (p.ej. un agente para leer un solo campo). Agrupa: un agente por consultor, no un agente por campo del consultor.

## Estructura del catálogo propuesta

```
.claude/workflows/           # orquestaciones nombradas, probadas
  ttp-consultores-contratos-aumentos.js
  review-changes.js
skills/                      # procedimientos de un agente
  ttp-consultant-files/SKILL.md
  phidimensions-account-dossier/SKILL.md
templates/                   # esqueletos de scripts para generación (doc 06)
  workflow-fanout-estado.js
```

Cada named Workflow documenta en su `meta.whenToUse` cuándo invocarlo, y cada skill mantiene la regla existente del Skill Workshop: cambios via `skill_workshop`, nunca edición directa de proposals.

## Resumen operativo

- Orquesta (fases, fan-out, resume, budget) → **named Workflow**.
- Procedimiento repetible de un agente → **Skill**.
- Sub-tarea de un solo uso → **agent() inline**.
- Segunda copia de algo = promoción al recipiente superior.
- El bloque atómico es idempotente, pequeño en I/O, con falla contenida; la escala y la compensación son siempre del orquestador.

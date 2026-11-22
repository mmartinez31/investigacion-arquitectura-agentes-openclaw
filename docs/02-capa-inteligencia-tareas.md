# Capa de Inteligencia de Tareas

Propuesta del agente Mac M4. Responde a la pregunta: una vez que el mensaje llega al router, ¿cómo sabe qué tipo de tarea es y cómo garantiza calidad?

## Problema que resuelve

La capa de transporte garantiza que el mensaje llegue al worker. Pero no garantiza que el worker:
- Lea los patrones de diseño correctos
- Produzca un spec antes de codificar
- Verifique contra un checklist al terminar
- Deje estado recuperable si falla a mitad

## Taxonomía de actividades

| Señal en el mensaje | Tipo | Comportamiento |
|---|---|---|
| "construye", "crea un sistema", "agrega X a Y" | BUILD | Spec gate → worker con checklist → audit post |
| "audita", "revisa", "hay algo mal" | AUDIT | Worker lee KB + grep anti-patrones + reporte |
| "arregla", "no funciona", "bug", "error" | FIX | Reproduce → spec del fix → corrige → smoke test |
| "¿cómo funciona?", "explícame", "qué es" | QUERY | Respuesta directa del router, sin worker |
| Tarea con >2 pasos dependientes | MULTI-STEP | TaskFlow con checkpoint por paso |

## Detección del tipo

El router clasifica por palabras clave + contexto. Cuando hay ambigüedad, pregunta antes de spawnear. La clasificación determina el template del brief del worker.

## Spec Gate (tipo BUILD y FIX)

Antes de spawnear el worker de implementación:
1. El router (o un micro-worker Haiku) produce un spec de 5-10 líneas:
   - Qué se va a construir/arreglar
   - Qué KB patterns aplican (pulled del AGENTS.md del repo destino)
   - Checklist de acceptance
   - Riesgos conocidos
2. El spec se envía a Miguel por Signal
3. Miguel dice OK (o corrige) → el worker de implementación arranca con el spec aprobado en su brief

## Brief del worker con contrato

El brief de un worker de tipo BUILD incluye:

```
Tarea: <descripción>
Spec aprobado: <spec>
KB patterns obligatorios: <rutas>
Checklist de acceptance (marca cada ítem antes de reportar done):
[ ] GlobalExceptionHandler cubre el nuevo endpoint
[ ] Frontend muestra toast en success Y en error
[ ] Acción irreversible tiene modal de confirmación
[ ] Smoke test en browser como usuario humano completado
[ ] CI pasa localmente
```

El checklist está en el primer bloque del brief, no al final.

## Audit Agent post-implementación

Un segundo subagente (Sonnet, fresh context) revisa el diff contra el checklist:
- Busca anti-patrones mecánicamente (grep: catch vacío, ausencia de toast, etc.)
- No puede aprobar si encuentra un anti-patrón del patrón `no-silent-errors`
- Su output va al router, que reporta a Miguel

## Enforcement mecánico (CI/CD)

Scripts en GitHub Actions que fallan el build si detectan:
- `catch {}` vacío o `catch (Exception e) {}` sin log ni rethrow en Java
- Llamadas `fetch()`/`axios` sin `.catch()` en TypeScript
- Ausencia de `@ControllerAdvice` en backend Spring Boot
- Respuestas HTTP 200 con campo `error` en el body

Esto es el único mecanismo 100% confiable. No depende de que el agente recuerde.

## Seguimiento de tareas multi-paso

Para tareas tipo MULTI-STEP (>2 pasos dependientes, >15 min estimados):
- El router crea un archivo de estado en el repo: `tasks/<CID>-state.md`
- Cada paso actualiza el archivo antes de continuar
- Si el worker falla, el router puede retomar desde el checkpoint
- El watchdog detecta workers zombies y reporta el último checkpoint conocido

## Integración con la capa de transporte

La capa de transporte (Mac M1) maneja: llegar, despachar, correlacionar.
Esta capa maneja: qué hacer, cómo hacerlo bien, cómo verificar.

Punto de integración: el Paso 3 del router (spawn del worker) se enriquece con el template de brief que esta capa define según el tipo de actividad.

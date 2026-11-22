# M1-02: Work State Registry

Propuesta del agente Mac M1. Este documento describe el Work State Registry: el mecanismo que mantiene estado compartido entre workers para evitar trabajo duplicado.

## El problema

OpenClaw no tiene memoria compartida nativa entre workers. Cada subagente es una sesión aislada con su propio store. Las herramientas `sessions_list` y `sessions_history` dan un recall acotado del historial de otras sesiones, pero no un registro de trabajo vivo: no responden de forma confiable la pregunta "¿qué está haciendo ahora mismo cada worker activo?".

Sin ese registro vivo, el router puede spawnear dos workers para la misma tarea. Esto pasaba antes del registry: si Miguel mandaba un mensaje, recibía su ACK, y luego mandaba una variante del mismo pedido (o el mismo pedido otra vez) antes de que el primer worker terminara, el router no tenía forma de saber que ya había un worker ocupado en eso. El resultado eran dos workers haciendo el mismo trabajo en paralelo, gastando recursos y, en tareas con efectos (commits, mensajes, cambios de archivo), pisándose entre sí.

## La solución

El estado compartido se mantiene en un archivo `WORK_STATE.json` en el workspace, operado a través del script `scripts/work-state.sh`.

El punto delicado es la concurrencia: varios workers pueden tocar el archivo al mismo tiempo. La solución estándar en Linux sería `flock`, pero `flock` no existe en macOS. Por eso el script usa un lock atómico con `mkdir`: crear un directorio es una operación atómica en el sistema de archivos (falla si el directorio ya existe), así que `mkdir lockdir` sirve de mutex sin dependencias externas. El worker que logra crear el directorio tiene el lock; los demás esperan hasta que se libere.

## El protocolo

El registry se usa en cuatro puntos del ciclo de vida de una tarea.

1. **Check antes de spawnear (router).** Entre el Paso 2 y el Paso 3 del despacho, el router corre `work-state.sh check "<descripción corta>"`. Si ya hay un worker activo para esa misma tarea, el router no spawnea un duplicado: le responde a Miguel que la tarea ya está en curso (con su CID) y termina el turno. Si la tarea solo aparece como completada y Miguel pide claramente rehacerla, o es una tarea nueva parecida, el router sigue normal.
2. **List en el brief de cada worker (router).** El router incluye en el brief, en un bloque aparte del mensaje literal de Miguel, el output de `work-state.sh list`. Así el worker ve qué están haciendo y qué ya terminaron los demás workers, y puede evitar solaparse.
3. **Register al inicio (worker).** El worker corre `work-state.sh register <CID> "<descripción corta>"` como primer paso, para quedar visible como activo ante el router y los demás workers.
4. **Complete al terminar (worker).** El worker corre `work-state.sh complete <CID> "<resumen de una línea>"` como último paso antes de devolver su respuesta final, para pasar de activo a completado.

El CID del registry es el mismo del Paso 0 del router (ver M1-01), no el `childSessionKey`. Estas llamadas al registry son trabajo del worker como cualquier otro; no cambian la regla de que el router solo controla y delega.

## Por qué evita duplicados

El check del Paso 3.5 cierra la ventana en la que antes se colaban los duplicados. Antes, dos mensajes parecidos producían dos workers porque el router no tenía estado compartido que consultar. Ahora el router consulta el registry antes de spawnear: si ve un worker activo para esa descripción, no lanza un segundo. La descripción corta es la clave de deduplicación, así que debe ser consistente entre el check del router y el register del worker.

El `list` en el brief añade una segunda capa: aunque el router deje pasar dos tareas distintas que resultan solaparse, cada worker ve en su brief qué están haciendo los demás y puede ajustar su alcance.

## Limitaciones actuales

- **El archivo es local.** `WORK_STATE.json` vive en el workspace de esta máquina (la Mac M1). No existe en la Mac M4 ni en ningún otro host. El registry coordina a los workers de un solo router, no a workers de máquinas distintas.
- **La deduplicación depende de la descripción.** El check compara descripciones cortas. Si el router describe la misma tarea de dos formas distintas, el check no detecta el duplicado. La consistencia de la descripción es responsabilidad del router.
- **No hay estado multi-paso recuperable.** El registry distingue activo y completado, pero no guarda el progreso interno de una tarea larga (en qué paso va, qué checkpoint alcanzó). Para eso haría falta un formato de estado por tarea, que la capa de inteligencia propone como archivos `tasks/<CID>.md`.

## Propuesta futura: store compartido multi-host

Para coordinar workers de varias máquinas (Mac M1 y Mac M4 trabajando sobre el mismo sistema) haría falta un store compartido que las dos puedan leer y escribir con locking confiable. El `WORK_STATE.json` local no sirve para eso. Opciones a evaluar: un archivo en un volumen compartido con un protocolo de lock que funcione en red, o un servicio ligero (una base de datos pequeña o un endpoint) que las dos máquinas consulten. La decisión queda abierta; lo que este documento fija es el requisito: el registry actual resuelve la coordinación de un host y no se extiende solo a varios.

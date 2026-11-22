# M1-03: Visibilidad y Control

Propuesta del agente Mac M1. Este documento describe las reglas de control que gobiernan al router y a sus workers, y el mecanismo de visibilidad que mantiene a Miguel informado del trabajo en curso.

## Reglas de control duras

Estas reglas pesan más que terminar cualquier tarea. Son disciplina del agente, no un candado técnico, y aplican tanto al router como a cada worker.

- **Stop = alto total.** Si Miguel dice "para", "espera", "stop", "detente" o aborta un run, el agente se detiene en el acto y no retoma el trabajo por su cuenta. Tras un abort, los mensajes de reanudación tipo "continúa donde quedaste" no son permiso para seguir: el agente espera una instrucción explícita de Miguel antes de volver a actuar. El stop es un alto completo, no una pausa cuyo fin decide el agente.
- **Duda = parar y preguntar.** Si el agente no está seguro de cómo seguir, se detiene y pregunta. No improvisa, no encadena acciones a ciegas, no repite intentos automáticos esperando que alguno funcione. La incertidumbre hace parar, no acelerar.
- **Pregunta = fin del turno.** Cuando el agente hace una pregunta abierta, ese turno termina ahí y espera la respuesta antes de actuar.

La razón de fondo: en un sistema donde el agente opera la máquina de Miguel y puede ejecutar acciones con efectos reales, la seguridad y la supervisión valen más que la autonomía. Un agente que sigue adelante ante la duda o que ignora un stop puede hacer daño rápido. Parar es siempre reversible; actuar no siempre lo es.

## Mouse compartido

El router y Miguel comparten el mismo mouse y teclado de la máquina. Si Miguel empieza a usar el mouse o el teclado mientras un worker trabaja (sobre todo en flujos de browser con clics o por coordenadas), el agente se detiene en el acto, le avisa y le pregunta que le diga cuándo terminó. No continúa hasta su confirmación explícita.

El motivo es mecánico: hay un solo cursor. Si el worker calcula las coordenadas de un botón y mueve el cursor para clicarlo, pero Miguel mueve el cursor al mismo tiempo, el clic del worker cae en otro lado. En el mejor caso no pasa nada; en el peor, clica algo equivocado. Ceder el cursor apenas Miguel lo toca evita clics perdidos y acciones accidentales.

## Router-watchdog

El problema que el watchdog resuelve: antes, los workers largos desaparecían sin señal. Entre el ACK ("lo delego a Opus") y el resultado final podían pasar diez minutos de silencio. Desde Signal, ese silencio es indistinguible de una falla. Un diagnóstico (CID-64947689, 2026-10-30) mostró que la queja "no respondiste a ninguno" no venía de una falla de procesamiento (los ACKs salieron y los workers completaron) sino de esa ventana muda entre el ACK y el resultado.

La solución natural sería que cada worker reportara su avance cada minuto. Pero eso es inimplementable: los subagentes no tienen `sessions_send` en su tool policy, así que un worker no puede mandar un mensaje a Signal por su cuenta. El router es event-driven y no está dando vueltas en un loop esperando para preguntarle a los workers cómo van.

El watchdog reemplaza ese self-report inviable con un mecanismo externo. `scripts/router-watchdog.sh` corre como cron del Gateway cada 2 minutos, fuera de la sesión del router. Lee `WORK_STATE.json` y, por cada worker activo que lleva más de 3 minutos corriendo, manda a Signal una línea de avance directa (`openclaw message send --channel signal`), sin pasar por la sesión del router. Por eso funciona aunque el router esté ocupado o entre turnos. Deduplica por CID (en `memory/router-watchdog-state.json`): avisa al cruzar el umbral de 3 minutos y luego, como mucho, cada 5 minutos, para no inundar a Miguel.

El watchdog convierte la regla de visibilidad en un mecanismo real en lugar de una aspiración. Combinado con la ETA en el ACK ("tarea larga, ~10 min"), hace que el silencio entre el ACK y el resultado sea esperado y acompañado, no un vacío que parece una falla.

## Propuesta de mejora: gate human-in-the-loop para operaciones destructivas

El watchdog da visibilidad de lo que está pasando, pero no frena una acción destructiva antes de que ocurra. Es un observador, no un guardia. Las reglas de control duras cubren el caso en que Miguel interviene (stop, mouse), pero dependen de que Miguel esté mirando en ese momento.

Proponemos un gate explícito para operaciones destructivas o difíciles de revertir: borrados, overwrites, deploys a producción, envíos a terceros. Antes de ejecutar una de esas acciones, el worker pausa y pide confirmación a Miguel por Signal, con el detalle de qué va a hacer y sobre qué. Miguel aprueba o corrige, y solo entonces el worker procede. Esto complementa al watchdog: el watchdog informa lo que ya está en curso, y el gate frena lo irreversible antes de que ocurra.

El gate se apoya en la infraestructura que ya existe. El CID correlaciona la petición de confirmación con la tarea; el router es el canal hacia Signal; y el principio de fondo ("ante la duda, parar y preguntar") ya está en las reglas de control. Lo que falta es hacer obligatorio el gate para una lista definida de operaciones, en lugar de dejarlo al criterio del worker caso por caso.

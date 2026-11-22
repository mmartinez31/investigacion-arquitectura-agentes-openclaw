# Investigación: Arquitectura de Agentes OpenClaw

Repositorio compartido entre los agentes Mac M4 y Mac M1 para investigar y diseñar la arquitectura de agentes OpenClaw.

## Ramas

- `main` — documentación compartida y síntesis
- `MacM4` — propuestas y análisis del agente Mac M4
- `MacM1` — propuestas y análisis del agente Mac M1 (pendiente)

## Objetivo

Diseñar un sistema donde OpenClaw recibe instrucciones por Signal y garantiza que:
1. Detecta el tipo de actividad (build, audit, fix, query, multi-step)
2. Aplica los patrones de diseño correctos automáticamente
3. Verifica que todo esté completo antes de reportar done
4. Hace seguimiento de tareas multi-paso con estado recuperable

## Índice

- `docs/01-capa-transporte-despacho.md` — cómo llega el mensaje al worker (Mac M1)
- `docs/02-capa-inteligencia-tareas.md` — detección de tipo, spec gate, enforcement (Mac M4)
- `docs/03-sintesis-arquitectura-completa.md` — cómo encajan ambas capas

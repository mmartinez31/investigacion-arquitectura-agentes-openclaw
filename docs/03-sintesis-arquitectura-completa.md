# Arquitectura Completa: Transporte + Inteligencia

## Las dos capas se complementan

```
Miguel (Signal)
  ↓
[CAPA DE TRANSPORTE — Mac M1]
  CID → Clasificación modelo → Cap workers → Work State → Spawn
  ↓
[CAPA DE INTELIGENCIA — Mac M4]
  Taxonomía actividad → Spec gate → Brief con contrato → Audit post
  ↓
Worker ejecuta con calidad garantizada
  ↓
Miguel recibe resultado verificado
```

## Flujo completo para una tarea BUILD

1. Miguel: "Agrega hard delete de usuarios a Timesheets"
2. Router asigna CID, detecta tipo=BUILD, verifica cap
3. Micro-worker Haiku produce spec (30s):
   - "Endpoint DELETE /api/admin/users/{id}/hard, solo Admin, cascade a timecards"
   - "KB patterns: no-silent-errors, RBAC permission pattern"
   - Checklist de 5 ítems
4. Router envía spec a Miguel por Signal
5. Miguel: "OK"
6. Router spawna worker Opus con brief = spec + checklist + rutas KB
7. Worker implementa, marca checklist, hace smoke test en browser
8. Audit agent verifica diff (Sonnet, fresh context)
9. CI corre scripts de anti-patrones
10. Router reporta a Miguel con evidencia (screenshot, CI run ID)

## Pendientes de diseño

- [ ] ¿Cómo implementar el spec gate en el router actual sin romper el ACK inmediato?
  Propuesta: spec gate es el ACK. El router responde con el spec en lugar de "delegué a Opus". Miguel confirma y eso dispara el spawn real.
- [ ] ¿El audit agent es siempre obligatorio o solo para BUILD/FIX?
  Propuesta: obligatorio para BUILD y FIX, opcional para QUERY y AUDIT.
- [ ] ¿Dónde viven los templates de brief por tipo de actividad?
  Propuesta: en el AGENTS.md del workspace, como sección "Task Templates".
- [ ] ¿El estado multi-paso va en WORK_STATE.json o en archivos separados por tarea?
  Propuesta: WORK_STATE.json para estado simple (activo/done), archivos `tasks/<CID>.md` para multi-paso.

## Próximos pasos

1. Implementar taxonomía de actividades en el router (clasificación + templates)
2. Implementar spec gate como parte del flujo BUILD/FIX
3. Implementar CI scripts de anti-patrones en los 3 sistemas
4. Definir formato de `tasks/<CID>.md` para multi-paso

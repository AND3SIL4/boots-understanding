---
description: Construye o actualiza el grafo canónico de un proyecto con lo que hay en _staging/, y limpia el staging
agent: graph-writer-agent
---

**No es un commit de git.** Toma lo que la ingestión dejó en
`obsidian-brain/_staging/` para un proyecto, lo escribe como grafo canónico
en `obsidian-brain/proyectos/<slug>/documentacion/` y deja el staging
vacío.

Proyecto: $ARGUMENTS

Antes de empezar:

1. Si no se indicó slug, lista los proyectos que tienen lotes en
   `obsidian-brain/_staging/` — cuántas notas y de qué fecha — y pregunta
   cuál escribir. Solo hace falta el slug: los lotes (`_staging/<slug>-*/`)
   los localizas y ordenas tú.
2. Verifica que `obsidian-brain/proyectos/<slug>/documentacion/` existe. Si
   no, detente e infórmalo — el proyecto se crea con `/nuevo-proyecto`, no
   al vuelo.
3. Si no hay ningún lote para ese proyecto, dilo: no hay nada que escribir,
   el grafo ya está al día.

No hace falta que el usuario haya revisado todo el staging. Reporta cuánto
quedó sin revisar y deja que el usuario decida si continuar; lo que entre
sin revisión humana queda marcado como ambiguo en el grafo, no se descarta
ni se da por bueno.

Trabaja en cinco fases: inventario (sin escribir) → escritura →
verificación → limpieza de staging → `Pendientes`. Lo no negociable:

- **Nada de `_staging/` se pierde.** Solo borras lo que verificaste presente
  en el vault; lo demás se queda en staging y lo reportas.
- Las notas conservan su `estado`. **Nunca** promuevas `propuesto` a
  `verificado`: esa marca la pone el humano en Obsidian.
- Una nota `verificado` en el vault no se sobrescribe con información que la
  contradiga: se conserva y se crea una `Inconsistencia`.
- Todo lo que no resolvió un humano —notas sin revisar, enlaces colgantes,
  candidatos a fusión, contradicciones— queda listado en
  `Pendientes - <slug>`.
- Los `_INFORME*.md` y `_CANDIDATOS-FUSION.md` se archivan en
  `documentacion/_ingestas/<lote>/` junto con el ledger de la escritura.

Al terminar, reporta: notas escritas por tipo, cuántas entraron sin revisar,
conflictos conservados, enlaces colgantes, cómo cambió la cobertura en
`Pendientes`, y **qué quedó en staging y por qué**.

---
description: Construye o actualiza el grafo canónico de un proyecto con lo que hay en _staging/, y limpia el staging
argument-hint: <slug-del-proyecto>
---

**No es un commit de git.** Este comando toma lo que la ingestión dejó en
`obsidian-brain/_staging/` para un proyecto, lo escribe como grafo canónico
en `obsidian-brain/proyectos/<slug>/documentacion/` y deja el staging
vacío.

Delega en el subagente `graph-writer-agent`:

**Proyecto:** $1

Antes de delegar:

1. Si `$1` está vacío, lista los proyectos que tienen lotes en
   `obsidian-brain/_staging/` — cuántas notas y de qué fecha — y pregunta
   cuál escribir. Solo hace falta el slug: el agente localiza los lotes
   (`_staging/<slug>-*/`) y los ordena por fecha él mismo.
2. Verifica que `obsidian-brain/proyectos/$1/documentacion/` existe. Si no,
   detente e infórmalo — el proyecto se crea con `/nuevo-proyecto <slug>`,
   no al vuelo.
3. Si no hay ningún lote para ese proyecto, dilo: no hay nada que escribir,
   el grafo ya está al día.

No hace falta que el usuario haya revisado todo el staging. El agente
reporta cuánto quedó sin revisar y el usuario decide si continuar; lo que
entre sin revisión humana queda marcado como ambiguo en el grafo, no se
descarta ni se da por bueno.

El agente trabaja en cinco fases: inventario (sin escribir) → escritura →
verificación → limpieza de staging → `Pendientes`. Los puntos que no son
negociables:

- **Nada de `_staging/` se pierde.** Solo se borra lo que se verificó
  presente en el vault; lo demás se queda en staging y se reporta.
- Las notas conservan su `estado`. El agente **nunca** promueve `propuesto`
  a `verificado`: esa marca la pone el humano en Obsidian.
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

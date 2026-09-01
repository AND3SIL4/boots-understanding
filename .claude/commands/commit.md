---
description: Toma un lote de _staging/ ya revisado y lo escribe al vault canónico
argument-hint: <lote-en-_staging> [slug-del-proyecto]
---

**No es un commit de git.** Este comando mueve notas revisadas de
`obsidian-brain/_staging/` al vault canónico.

Delega en el subagente `graph-writer-agent`:

**Lote:** $1
**Proyecto:** $2

Antes de delegar:

1. Si `$1` está vacío, lista los lotes en `obsidian-brain/_staging/` con
   cuántas notas tiene cada uno y pregunta cuál aprobar.
2. Confirma con el usuario que **ya revisó** ese lote. Este comando existe
   para ejecutar una decisión humana, no para tomarla: si el usuario no ha
   mirado el staging, dile que lo revise primero.
3. Si `$2` está vacío, dedúcelo del frontmatter de las notas del lote o de
   la carpeta de origen. Si no es deducible, pregúntalo.

El agente debe, además de mover las notas:

- Cambiar `estado: propuesto` → `verificado` y actualizar
  `last_verified_date`.
- Regenerar `_overview.md` y actualizar `Proyecto`, `Arquitectura`,
  `MejoresPracticas` y `Pendientes` del proyecto.
- Verificar que todo wikilink apunte a una nota existente; los que no,
  registrarlos en `Pendientes` como nodos faltantes.

Al terminar, reporta qué se escribió, qué enlaces quedaron colgando y cómo
cambió la tabla de cobertura en `Pendientes`.

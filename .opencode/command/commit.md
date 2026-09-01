---
description: Toma un lote de _staging/ ya revisado y lo escribe al vault canónico
agent: graph-writer-agent
---

**No es un commit de git.** Mueve notas revisadas de
`obsidian-brain/_staging/` al vault canónico.

Lote y proyecto: $ARGUMENTS

Antes de empezar:

1. Si no se indicó lote, lista los que hay en `obsidian-brain/_staging/`
   con cuántas notas tiene cada uno y pregunta cuál aprobar.
2. Confirma que el usuario **ya revisó** ese lote. Este comando ejecuta una
   decisión humana, no la toma.
3. Si no se indicó proyecto, dedúcelo del frontmatter de las notas o de la
   carpeta de origen; si no es deducible, pregúntalo.

Además de mover las notas: cambia `estado: propuesto` → `verificado`,
actualiza `last_verified_date`, regenera `_overview.md` y actualiza
`Proyecto`, `Arquitectura`, `MejoresPracticas` y `Pendientes`. Verifica que
todo wikilink apunte a una nota existente y registra en `Pendientes` los
que queden colgando.

---
description: Procesa un documento crudo de _inbox/ y propone notas en obsidian-brain/_staging/
argument-hint: <ruta-en-_inbox> [slug-del-proyecto]
---

Delega en el subagente `ingestion-agent` la ingestión de:

**Fuente:** $1
**Proyecto:** $2

Antes de delegar:

1. Si `$1` está vacío, lista qué hay en `_inbox/` y pregunta cuál procesar.
   No elijas por tu cuenta.
2. Si `$1` no existe en disco, dilo y muestra qué sí hay en `_inbox/` — no
   busques un archivo "parecido".
3. Si `$2` está vacío, dedúcelo de la ruta (`_inbox/<slug>/...`). Si la
   ruta no lo revela, **pregúntalo**: sin slug de proyecto el
   `graph-writer-agent` no sabrá bajo qué `obsidian-brain/proyectos/<slug>/`
   ubicar las notas después.
4. Si la fuente es una carpeta con muchos archivos, procesa un documento
   por lote. Un lote gigante es un lote que nadie revisa de verdad.

El agente escribe únicamente en `obsidian-brain/_staging/<lote>/`. Al
terminar, reporta:

- Cuántas notas de cada tipo se propusieron.
- Qué bloques de documentación quedaron **sin cubrir** por esta fuente
  (esto va a `Pendientes`) — es tan importante como lo que sí se extrajo.
- Qué contradicciones se detectaron contra notas ya existentes (van como
  `Inconsistencia`, sin resolver).

Recuérdale al usuario que el siguiente paso es revisar el staging antes de
aprobar, y dile la ruta exacta del lote.

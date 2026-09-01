---
description: Procesa un documento crudo de _inbox/ — o un proyecto completo — y propone notas en obsidian-brain/_staging/
argument-hint: <ruta-en-_inbox | raíz-del-proyecto> [slug-del-proyecto]
---

Delega en el subagente `ingestion-agent` la ingestión de:

**Fuente:** $1
**Proyecto:** $2

## Antes de delegar

1. Si `$1` está vacío, lista qué hay en `_inbox/` y pregunta cuál procesar.
   No elijas por tu cuenta.
2. Si `$1` no existe en disco, dilo y muestra qué sí hay en `_inbox/` — no
   busques un archivo "parecido".
3. Si `$2` está vacío, dedúcelo de la ruta (`_inbox/<slug>/...`). Si la
   ruta no lo revela, **pregúntalo**: sin slug de proyecto el
   `graph-writer-agent` no sabrá bajo qué `obsidian-brain/proyectos/<slug>/`
   ubicar las notas después.
4. **Verifica que el proyecto ya existe en el vault** y detente en el
   primer punto que falle — informa, no crees nada:
   - `obsidian-brain/proyectos/<slug>/` debe existir. Si no, dilo, lista
     los proyectos que sí hay y recuerda que se crea con
     `/nuevo-proyecto <slug>`.
   - `obsidian-brain/proyectos/<slug>/documentacion/` debe existir. Si no,
     **informa y detente**: el proyecto no se creó desde
     `proyecto-plantilla`. No crees la carpeta tú.
5. Decide el modo y díselo al agente explícitamente:
   - **`$1` es un archivo o una carpeta pequeña** → modo archivo/lote, un
     documento a la vez.
   - **`$1` es la raíz de un proyecto** (`_inbox/<slug>/`, con subcarpetas
     de export, base de datos, documentación...) → **modo proyecto
     completo**: todo el árbol en una sola pasada, respetando el orden de
     precedencia y la lista de exclusión de la skill `ingestion-pipeline`.

## Modo proyecto completo

Recuérdale al agente, porque es lo que define este modo:

- **Orden de precedencia** — no empezar un nivel sin cerrar el anterior:
  1. export de Automation Anywhere (verdad técnica: qué hace el bot),
  2. DDL de la base de datos (verdad de datos: tablas y campos),
  3. documentación funcional y de negocio (verdad del porqué),
  4. resto legible (logs, transcripciones) como contexto, `confidence: baja`.
  Cuando un nivel inferior contradice a uno superior → nota
  `Inconsistencia` con ambas citas, sin elegir ganador.
- **Se omiten sin abrirlos** (ni un intento de lectura): videos y audios,
  archivos `.zip` y demás comprimidos, todo lo que cuelgue de una carpeta
  `dml*` (de la base de datos solo interesa el DDL), y las capturas de
  pantalla dentro del árbol del export (`*Metadata/*.png` y similares).
- **Lo omitido se reporta**, con el conteo por regla, en el `_INFORME.md`
  del lote.

Antes de que el agente lea el primer archivo, muéstrale al usuario el
inventario del árbol — qué se va a procesar en cada nivel y qué se omite
por cuál regla — y **espera su confirmación**. Un proyecto completo es caro
de ingerir y más caro de revisar a ciegas.

## Al terminar

El agente escribe únicamente en `obsidian-brain/_staging/<lote>/`. Reporta:

- Cuántas notas de cada tipo se propusieron (en modo proyecto completo,
  desglosadas por nivel de precedencia).
- Qué archivos se omitieron y por cuál regla.
- Qué bloques de documentación quedaron **sin cubrir** por estas fuentes
  (esto va a `Pendientes`) — es tan importante como lo que sí se extrajo.
- Qué contradicciones se detectaron, entre niveles o contra notas ya
  existentes (van como `Inconsistencia`, sin resolver).

Recuérdale al usuario que el siguiente paso es revisar el staging antes de
aprobar, y dile la ruta exacta del lote.

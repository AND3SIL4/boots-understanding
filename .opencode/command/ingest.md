---
description: Procesa un documento crudo de _inbox/ — o un proyecto completo — y propone notas en obsidian-brain/_staging/
agent: ingestion-agent
---

Ingiere la siguiente fuente: $ARGUMENTS

El primer argumento es la ruta dentro de `_inbox/` — un archivo, una
carpeta, o la raíz de un proyecto; el segundo, opcional, es el slug del
proyecto.

## Antes de empezar

1. Si no se indicó ruta, lista qué hay en `_inbox/` y pregunta cuál
   procesar. No elijas por tu cuenta.
2. Si la ruta no existe, dilo y muestra qué sí hay en `_inbox/` — no
   busques un archivo "parecido".
3. Si no se indicó proyecto, dedúcelo de la ruta (`_inbox/<slug>/...`). Si
   la ruta no lo revela, pregúntalo: sin slug de proyecto no se sabrá bajo
   qué `obsidian-brain/proyectos/<slug>/` ubicar las notas después.
4. **Verifica que el proyecto ya existe en el vault** y detente en el
   primer punto que falle — informa, no crees nada:
   - `obsidian-brain/proyectos/<slug>/` debe existir. Si no, dilo, lista
     los proyectos que sí hay y recuerda que se crea con
     `/nuevo-proyecto <slug>`.
   - `obsidian-brain/proyectos/<slug>/documentacion/` debe existir. Si no,
     **informa y detente**: el proyecto no se creó desde
     `proyecto-plantilla`. No crees la carpeta tú.
5. Decide el modo:
   - **Archivo o carpeta pequeña** → modo archivo/lote, un documento a la vez.
   - **Raíz de un proyecto** (`_inbox/<slug>/`, con subcarpetas de export,
     base de datos, documentación...) → **modo proyecto completo**: todo el
     árbol en una sola pasada, con el orden de precedencia y la lista de
     exclusión de la skill `ingestion-pipeline`.

## Modo proyecto completo

- **Orden de precedencia** — no empieces un nivel sin cerrar el anterior:
  1. export de Automation Anywhere (verdad técnica: qué hace el bot),
  2. DDL de la base de datos (verdad de datos: tablas y campos),
  3. documentación funcional y de negocio (verdad del porqué),
  4. resto legible (logs, transcripciones) como contexto, `confidence: baja`.
  Cuando un nivel inferior contradice a uno superior → nota
  `Inconsistencia` con ambas citas, sin elegir ganador.
- **Omite sin abrirlos** (ni un intento de lectura): videos y audios,
  archivos `.zip` y demás comprimidos, todo lo que cuelgue de una carpeta
  `dml*` (de la base de datos solo interesa el DDL), y las capturas de
  pantalla dentro del árbol del export (`*Metadata/*.png` y similares).
- **Lo omitido se reporta**, con el conteo por regla, en el `_INFORME.md`
  del lote.

Antes de leer el primer archivo, muestra el inventario del árbol — qué vas
a procesar en cada nivel y qué omites por cuál regla — y **espera
confirmación**. Un proyecto completo es caro de ingerir y más caro de
revisar a ciegas.

## Al terminar

Escribe únicamente en `obsidian-brain/_staging/<lote>/`. Reporta cuántas
notas de cada tipo propusiste (desglosadas por nivel si fue proyecto
completo), qué archivos omitiste y por cuál regla, qué bloques de
documentación quedaron sin cubrir, y qué contradicciones detectaste entre
niveles o contra notas ya existentes. Indica la ruta exacta del lote para
que el usuario lo revise antes de aprobarlo.

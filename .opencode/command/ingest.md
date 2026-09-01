---
description: Procesa un documento crudo de _inbox/ y propone notas en obsidian-brain/_staging/
agent: ingestion-agent
---

Ingiere la siguiente fuente: $ARGUMENTS

El primer argumento es la ruta dentro de `_inbox/`; el segundo, opcional,
es el slug del proyecto.

Antes de empezar:

1. Si no se indicó ruta, lista qué hay en `_inbox/` y pregunta cuál
   procesar. No elijas por tu cuenta.
2. Si la ruta no existe, dilo y muestra qué sí hay en `_inbox/` — no
   busques un archivo "parecido".
3. Si no se indicó proyecto, dedúcelo de la ruta (`_inbox/<slug>/...`). Si
   la ruta no lo revela, pregúntalo: sin slug de proyecto no se sabrá bajo
   qué `obsidian-brain/proyectos/<slug>/` ubicar las notas después.
4. Si la fuente es una carpeta con muchos archivos, procesa un documento
   por lote. Un lote gigante es un lote que nadie revisa de verdad.

Escribe únicamente en `obsidian-brain/_staging/<lote>/`. Al terminar
reporta cuántas notas de cada tipo propusiste, qué bloques de
documentación quedaron sin cubrir por esta fuente, y qué contradicciones
detectaste contra notas ya existentes. Indica la ruta exacta del lote para
que el usuario lo revise antes de aprobarlo.

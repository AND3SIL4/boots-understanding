---
name: ingestion-pipeline
description: "Use this when processing a raw document (pdf, docx, xlsx, txt, notes) from _inbox/ into candidate notes for the vault. Defines the manual (no MCP, no vectorization) pipeline: read, extract, propose to staging, never write directly to the canonical vault."
---

# Pipeline de ingestión (versión CLI, sin MCP)

Objetivo: convertir un documento crudo en notas candidatas siguiendo el
esquema de `vault-conventions`, sin escribir nunca directamente al vault
canónico.

## Pasos

0. **Ubicar el insumo y el proyecto.** Los insumos viven en `_inbox/`, en
   la raíz de ejecución y fuera del vault; pueden estar en subcarpetas
   (`_inbox/export/` para exports de Automation Anywhere,
   `_inbox/<slug-del-proyecto>/` cuando hay varios proyectos). Antes de
   empezar, confirma a qué proyecto pertenece el documento: es el
   `<slug-del-proyecto>` bajo el que terminarán las notas en
   `obsidian-brain/proyectos/`. Si no se puede deducir, pregúntalo.

1. **Leer el documento** desde `_inbox/`, usando la skill de lectura
   correspondiente a su tipo — no leas el archivo crudo sin pasar primero
   por la skill adecuada, cada una sabe qué buscar y qué riesgos evitar
   para ese formato:
   - `.pdf` → skill `read-pdf`.
   - `.docx` → skill `read-docx`.
   - `.xlsx` / `.csv` → skill `read-xlsx`.
   - `.md`, `.txt`, notas informales → skill `read-plaintext-notes`.
   - `.atmx`, `.mbot`, `.aapkg`, o un XML/TXT convertido desde alguno de
     esos → skill `read-automation-anywhere-export`. Este tipo de fuente
     es la más confiable para "qué hace el bot realmente" — si contradice
     a un manual de negocio ya ingerido, no elijas una fuente en silencio,
     marca la discrepancia explícitamente (ver esa skill para el detalle).
   - Si el formato no encaja en ninguna de las anteriores, dilo
     explícitamente al usuario y pide una conversión antes de continuar,
     en vez de adivinar el contenido.
2. **Identificar candidatos a nodo**: leer el texto buscando procesos,
   pasos, reglas ("si... entonces..."), sistemas mencionados, campos de
   datos, excepciones, historias de usuario y bots — según los tipos
   definidos en `vault-conventions`.
3. **Redactar cada nodo** como una nota siguiendo su plantilla en
   `obsidian-brain/_templates/`, con `estado: propuesto` y `confidence` honesta
   (`alta` si es una cita casi literal del documento, `baja` si es una
   inferencia).
4. **Completar relaciones** (secciones tipadas) solo cuando el documento
   las sustenta explícita o casi explícitamente. Si una relación parece
   probable pero no está clara en el texto, no inventarla — dejar una nota
   en la sección `## Extraído de` mencionando la ambigüedad en vez de crear
   el link.
5. **Escribir todo en `obsidian-brain/_staging/<lote>/`**, nunca en la ubicación
   final del vault. `<lote>` = nombre del documento + fecha, ej.
   `obsidian-brain/_staging/manual-conciliacion-2026-08-28/`.
6. **Nunca copiar el insumo al vault.** El archivo se queda en `_inbox/`;
   la nota de tipo `Documento` guarda su ruta relativa (ej.
   `_inbox/export/manifest.json`). El vault es solo markdown.
7. **Resumir al usuario**: cuántas notas de cada tipo se propusieron, y
   marcar explícitamente cualquier zona del documento que no se pudo
   interpretar con confianza.

La revisión humana del staging y el paso a vault final es responsabilidad
del `graph-writer-agent`, no de este pipeline.

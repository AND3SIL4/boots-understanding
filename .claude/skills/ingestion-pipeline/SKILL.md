---
name: ingestion-pipeline
description: "Use this when processing a raw document (pdf, docx, xlsx, txt, notes) from vault/_inbox/ into candidate notes for the vault. Defines the manual (no MCP, no vectorization) pipeline: read, extract, propose to staging, never write directly to the canonical vault."
---

# Pipeline de ingestión (versión CLI, sin MCP)

Objetivo: convertir un documento crudo en notas candidatas siguiendo el
esquema de `vault-conventions`, sin escribir nunca directamente al vault
canónico.

## Pasos

1. **Leer el documento** desde `vault/_inbox/`.
   - `.md`, `.txt` → leer directo.
   - `.pdf`, `.docx`, `.xlsx` → si el entorno tiene skills de documentos
     disponibles (pdf/docx/xlsx), usarlas para extraer texto y tablas antes
     de continuar. Si no están disponibles, pedir al usuario que convierta
     el archivo a texto/markdown primero.
2. **Identificar candidatos a nodo**: leer el texto buscando procesos,
   pasos, reglas ("si... entonces..."), sistemas mencionados, campos de
   datos, excepciones, historias de usuario y bots — según los tipos
   definidos en `vault-conventions`.
3. **Redactar cada nodo** como una nota siguiendo su plantilla en
   `vault/_templates/`, con `estado: propuesto` y `confidence` honesta
   (`alta` si es una cita casi literal del documento, `baja` si es una
   inferencia).
4. **Completar relaciones** (secciones tipadas) solo cuando el documento
   las sustenta explícita o casi explícitamente. Si una relación parece
   probable pero no está clara en el texto, no inventarla — dejar una nota
   en la sección `## Extraído de` mencionando la ambigüedad en vez de crear
   el link.
5. **Escribir todo en `vault/_staging/<lote>/`**, nunca en la ubicación
   final del vault. `<lote>` = nombre del documento + fecha, ej.
   `vault/_staging/manual-conciliacion-2026-08-28/`.
6. **Resumir al usuario**: cuántas notas de cada tipo se propusieron, y
   marcar explícitamente cualquier zona del documento que no se pudo
   interpretar con confianza.

La revisión humana del staging y el paso a vault final es responsabilidad
del `graph-writer-agent`, no de este pipeline.

---
name: read-pdf
description: "Use when a raw document in _inbox/ is a .pdf. Extracts text, tables, and flags scanned/image-only content that needs OCR, so ingestion-pipeline can process it like any other source document."
---

# Leer documentos PDF

## Cómo extraer el contenido

1. Si el entorno ya trae una skill dedicada de PDF (algunos entornos de
   Claude Code la incluyen), úsala primero — suele dar mejor extracción de
   tablas que un extractor genérico.
2. Si no, extrae texto con herramientas de línea de comandos estándar
   (`pdftotext -layout` conserva mejor la estructura de tablas que la
   extracción por defecto).
3. Si el PDF es un **escaneo** (texto no seleccionable, o `pdftotext`
   devuelve vacío/basura): no lo trates como texto extraído. Repórtalo al
   `ingestion-agent` como "requiere OCR" y, si hay herramienta de OCR
   disponible en el entorno, úsala; si no, indícale al usuario que el
   documento necesita conversión manual antes de poder ingerirse con
   confianza.

## Qué buscar específicamente en manuales RPA en PDF

- Diagramas de flujo del proceso (suelen ser imágenes — si no se pueden
  leer como texto, descríbelos en prosa a partir de lo que sí se pueda leer
  alrededor, y marca `confidence: baja` en cualquier nota que dependa de
  interpretar la imagen).
- Tablas de reglas de negocio (muy comunes en anexos) — extraerlas fila por
  fila, no como bloque de texto suelto, para que cada fila pueda volverse
  una `ReglaDeNegocio` separada.
- Pies de página/encabezados con número de versión y fecha del documento —
  van al frontmatter de la nota `Documento` correspondiente.

No inventes contenido de páginas que no se pudieron extraer — repórtalas
como no leídas en vez de rellenarlas.

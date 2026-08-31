---
name: read-docx
description: "Use when a raw document in _inbox/ is a .docx. Extracts headings, paragraphs, tables, and comments/tracked changes so ingestion-pipeline can process it like any other source document."
---

# Leer documentos Word (.docx)

## Cómo extraer el contenido

1. Si el entorno ya trae una skill dedicada de docx, úsala primero.
2. Si no, extrae el texto preservando la jerarquía de encabezados (Título
   1/2/3) — la jerarquía suele reflejar la jerarquía real del proceso
   (Título 1 = Proceso, Título 2 = Paso, por ejemplo), así que consérvala
   en vez de aplanar todo a un solo bloque de texto.
3. Extrae las tablas por separado, fila por fila — igual que en PDF, son
   candidatas frecuentes a `ReglaDeNegocio` o `CampoDeDato`.

## Qué no ignorar

- **Comentarios y cambios con seguimiento**: en documentación RPA suelen
  contener la discusión real de por qué una regla es como es, o
  correcciones que nunca se aplicaron al cuerpo del texto. Si el documento
  los tiene, extráelos y trátalos como fuente de menor `confidence` que el
  cuerpo aprobado del documento, pero no los descartes.
- **Notas al pie**: frecuentemente contienen la excepción a una regla que
  el cuerpo del texto no menciona.

No inventes contenido de secciones dañadas o ilegibles — repórtalas como no
leídas.

---
name: read-xlsx
description: "Use when a raw document in vault/_inbox/ is a .xlsx (or .csv). Extracts each sheet as structured rows so ingestion-pipeline can turn tabular business rules into individual notes."
---

# Leer hojas de cálculo (.xlsx / .csv)

## Cómo extraer el contenido

1. Procesa **cada hoja por separado** — en documentación RPA, cada hoja
   suele ser una tabla temáticamente distinta (una hoja de reglas, otra de
   excepciones, otra de mapeo de campos).
2. Trata la primera fila como encabezados y cada fila siguiente como un
   registro individual — no resumas la hoja en prosa, preserva la
   granularidad fila por fila.
3. Identifica el propósito de la hoja por su nombre y sus encabezados de
   columna antes de decidir a qué tipo de nodo mapea cada fila:
   - Hoja de reglas/condiciones → cada fila candidata a `ReglaDeNegocio`.
   - Hoja de campos/diccionario de datos → cada fila candidata a
     `CampoDeDato`.
   - Hoja de excepciones/casos especiales → cada fila candidata a
     `Excepcion`.
   - Hoja de trazabilidad HU → sistema → candidata a relaciones
     `Implementada por` / `Interactúa con`.

## Cuidado con

- Celdas combinadas o fórmulas: reporta el valor calculado, no la fórmula,
  salvo que la fórmula en sí sea la regla de negocio (ej. una columna que
  calcula el umbral de aprobación).
- Filas ocultas o con formato de "obsoleto"/tachado: repórtalas aparte,
  marcadas como posiblemente desactualizadas (`confidence: baja`), no las
  mezcles sin distinción con las filas vigentes.

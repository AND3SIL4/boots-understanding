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
2. **Barrer el documento contra la lista completa de tipos.** No leer
   buscando "lo importante" — recorrer el texto una vez por bloque, porque
   lo que se documenta mal es justamente lo que nadie va a buscar. Para
   cada bloque, o se extraen notas, o se anota en `Pendientes` que la
   fuente no lo cubre:

   | Bloque | Qué buscar en el texto | Tipo |
   | --- | --- | --- |
   | Proceso y flujo | qué hace, en qué orden | `Proceso`, `PasoDeProceso` |
   | Reglas | "si... entonces", umbrales, aprobaciones | `ReglaDeNegocio` |
   | Validaciones | formatos, obligatoriedad, rangos, cruces | `Validacion` |
   | Sistemas | plataforma RPA, SAP, BD, apps, servicios | `Sistema` (usa `categoria`) |
   | Ambientes y red | dev/QA/prod, VPN, servidores, orquestador | `Ambiente` |
   | Seguridad | usuarios de servicio, perfiles, permisos | `Acceso` |
   | Datos | campos de negocio que fluyen | `CampoDeDato` |
   | Configuración | variables, parámetros, valores por ambiente | `Parametro` |
   | Entradas | archivos, colas, correos que consume | `Insumo` |
   | Salidas | reportes, archivos, registros que produce | `Salida` |
   | Comunicación | correos, alertas, plantillas | `Notificacion` |
   | Casos borde | qué pasa si falla, reprocesos | `Excepcion` |
   | Fallas reales | incidentes ocurridos, stoppers | `Incidencia` |
   | Amenazas | lo que puede romperse y no está cubierto | `Riesgo` |
   | Requisitos | "como X quiero Y", criterios de aceptación | `HistoriaDeUsuario` |
   | Personas | roles, áreas, dueño del proceso, quién aprueba | `Actor` |
   | Bot | herramienta, dónde vive el código | `Robot` |
   | Medición | KPIs, SLAs, metas, tiempos comprometidos | `Indicador` |
   | Pruebas | casos UAT, evidencias, resultados | `PruebaUAT` |
   | Decisiones | por qué se hizo así, qué se descartó | `Decision` |
   | Fechas | puestas en producción, cambios, migraciones | `Hito` |
   | Vocabulario | términos del dominio, siglas, sinónimos | `Termino` |
   | Dudas recurrentes | lo que el documento aclara porque siempre preguntan | `PreguntaFrecuente` |

   Y las notas singleton del proyecto: `ResumenEjecutivo` (`_overview.md`),
   `Proyecto` (contexto, alcance, objetivos, beneficios), `Arquitectura`,
   `MejoresPracticas`, `Pendientes`. Si ya existen en el vault, se
   **proponen como edición**, no como nota nueva — el `graph-writer-agent`
   fusiona.

2b. **Registrar contradicciones, no resolverlas.** Si el documento
   contradice algo ya ingerido — típicamente un export de Automation
   Anywhere contra un manual de negocio — crear una nota `Inconsistencia`
   con la cita literal de ambas fuentes. No elegir una fuente en silencio.

2c. **Registrar los huecos.** Todo bloque de la tabla que el documento no
   cubra va a `Pendientes`, con la pregunta concreta y a quién habría que
   preguntarle. Un hueco declarado vale más que un campo rellenado por
   inferencia.
3. **Redactar cada nodo** como una nota siguiendo su plantilla en
   `obsidian-brain/_templates/`, con `estado: propuesto` y `confidence` honesta
   (`alta` si es una cita casi literal del documento, `baja` si es una
   inferencia).
3b. **Nunca transcribir un secreto.** Si el documento trae contraseñas,
   tokens, cadenas de conexión con credenciales o llaves, la nota `Acceso`
   registra que el acceso existe y dónde se custodia — nunca el valor. El
   secreto expuesto se reporta en `Pendientes` como hallazgo.

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
7. **Resumir al usuario**: cuántas notas de cada tipo se propusieron,
   qué bloques de la tabla del paso 2 quedaron sin cubrir, y cualquier
   zona del documento que no se pudo interpretar con confianza. El resumen
   de lo que FALTA es tan importante como el de lo que se extrajo.

La revisión humana del staging y el paso a vault final es responsabilidad
del `graph-writer-agent`, no de este pipeline.

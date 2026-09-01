---
name: ingestion-pipeline
description: "Use this when processing a raw document (pdf, docx, xlsx, txt, notes) from _inbox/ into candidate notes for the vault. Defines the manual (no MCP, no vectorization) pipeline: read, extract, propose to staging, never write directly to the canonical vault."
---

# Pipeline de ingestión (versión CLI, sin MCP)

Objetivo: convertir un documento crudo en notas candidatas siguiendo el
esquema de `vault-conventions`, sin escribir nunca directamente al vault
canónico.

## Dos modos de ingestión

| Modo | Cuándo aplica | Alcance |
| --- | --- | --- |
| **Archivo o lote** | La ruta recibida es un archivo suelto, o una carpeta con pocos documentos | Un documento a la vez |
| **Proyecto completo** | La ruta recibida es la raíz de un proyecto en `_inbox/<slug>/` | Todo el árbol de una sola pasada, en el orden de precedencia de abajo |

El modo proyecto completo **no relaja ninguna regla** del pipeline: sigue
siendo un documento a la vez, con su propia extracción y sus propias notas.
Lo que agrega es un orden obligatorio, una lista de exclusión y una
precondición sobre el vault.

### Precondición: el proyecto ya existe en el vault

Antes de abrir un solo archivo, verificar en este orden y **detenerse** en
el primero que falle — informar al usuario, no crear nada:

1. `obsidian-brain/proyectos/<slug>/` existe. Si no, el proyecto no está
   creado: decirlo, listar los proyectos que sí existen, y recordar que se
   crea con `/nuevo-proyecto <slug>`. Nunca ingerir contra un proyecto
   inexistente.
2. `obsidian-brain/proyectos/<slug>/documentacion/` existe. Si no,
   informarlo y detenerse — **no crear la carpeta**: su ausencia significa
   que el proyecto no se creó desde `proyecto-plantilla` y hay que
   revisarlo antes de escribir nada.

Todo lo que produce la ingestión termina bajo `documentacion/`. Las demás
carpetas del proyecto (`control-cambios/`, `desarrollo/`, `reuniones/`,
`soporte/`, `notas-generales.md` y la nota raíz del proyecto) son trabajo
humano: la ingestión no escribe ahí ni propone cambios ahí.

### Orden de precedencia (cuál fuente manda)

Procesar en este orden y no empezar un nivel sin haber terminado el
anterior — lo que se ingiere primero fija el marco contra el que se lee
todo lo demás:

| # | Nivel | Dónde suele estar | Autoridad |
| --- | --- | --- | --- |
| 1 | Export de Automation Anywhere | `export-aa/`, `export-automation-anywhere/`, `export/` | **Verdad técnica**: qué hace el bot realmente |
| 2 | DDL de base de datos | `ddl*`, `database-*/ddl*.sql` | **Verdad de datos**: tablas, campos, tipos, llaves |
| 3 | Documentación funcional y de negocio | `documentacion-general/` (docx, pdf, xlsx, pptx) | Verdad del **por qué**: intención, alcance, reglas de negocio |
| 4 | Resto legible | `logs-ejemplos/`, `transcripciones/`, `.txt`, `.md` sueltos | Contexto. `confidence: baja` salvo cita literal |

Reglas de precedencia:

- Un nivel inferior **nunca sobrescribe en silencio** a uno superior. Si el
  manual (3) contradice al bot (1) o al DDL (2), se crea una nota
  `Inconsistencia` citando ambas fuentes y diciendo cuál es la autoridad
  para ese tipo de afirmación. No se elige ganador.
- Un nivel inferior sí puede **completar** lo que el superior no dice —
  típicamente el porqué de negocio detrás de un paso técnico.
- Para lo técnico (pasos, variables, tablas, campos), si (1) o (2) lo
  cubren, esa es la nota; el documento de negocio se cita como contexto,
  no como origen del dato.

### Lista de exclusión (no leer, no intentar leer)

Estos archivos se **omiten sin abrirlos**. No es "intentar y si falla
seguir": ni siquiera se intenta.

| Qué | Cómo reconocerlo | Por qué |
| --- | --- | --- |
| Video y audio | `.mp4`, `.avi`, `.mov`, `.mkv`, `.wmv`, `.webm`, `.mp3`, `.wav`, `.m4a`, `.ogg` | No hay skill de lectura; transcribirlos es otro proceso, aparte |
| Comprimidos | `.zip` — y por la misma razón `.rar`, `.7z`, `.tar`, `.gz` | El contenido ya está descomprimido al lado en el export; abrirlos duplica |
| DML de base de datos | Cualquier ruta con un segmento `dml*` (ej. `database-sqlserver/dml-cuenta-transitoria/`) | De la base de datos solo interesa la **estructura** (DDL), no los datos cargados |
| Capturas del export | Imágenes (`.png`, `.jpg`, `.jpeg`, `.gif`, `.bmp`) dentro del árbol del export — típicamente en carpetas `*Metadata/` | Son screenshots de captura de objetos del bot, no lógica |

Lo omitido **se reporta, no se esconde**: el informe del lote dice cuántos
archivos se saltaron y por cuál regla. Si algo omitido resulta ser la única
fuente posible de un tema, eso va a `Pendientes` como hueco declarado.

## Pasos

0. **Ubicar el insumo y el proyecto.** Los insumos viven en `_inbox/`, en
   la raíz de ejecución y fuera del vault; pueden estar en subcarpetas
   (`_inbox/export/` para exports de Automation Anywhere,
   `_inbox/<slug-del-proyecto>/` cuando hay varios proyectos). Antes de
   empezar, confirma a qué proyecto pertenece el documento: es el
   `<slug-del-proyecto>` bajo el que terminarán las notas en
   `obsidian-brain/proyectos/`. Si no se puede deducir, pregúntalo.

0b. **Si la ruta es la raíz de un proyecto** (`_inbox/<slug>/`), estás en
   modo proyecto completo: aplica primero la precondición del vault, arma
   el inventario del árbol marcando cada archivo con su nivel de
   precedencia o su regla de exclusión, y **muestra ese inventario antes de
   leer nada**. Un inventario que el usuario no vio es una ingestión que
   nadie puede auditar. Recién ahí se recorre el árbol, nivel por nivel.

1. **Leer el documento** desde `_inbox/`, usando la skill de lectura
   correspondiente a su tipo — no leas el archivo crudo sin pasar primero
   por la skill adecuada, cada una sabe qué buscar y qué riesgos evitar
   para ese formato:
   - `.pdf` → skill `read-pdf`.
   - `.docx` → skill `read-docx`.
   - `.xlsx` / `.csv` → skill `read-xlsx`.
   - `.md`, `.txt`, notas informales → skill `read-plaintext-notes`.
   - `.atmx`, `.mbot`, `.aapkg`, un XML/TXT convertido desde alguno de
     esos, o los archivos **sin extensión** que cuelgan del árbol del
     export (son JSON de Automation 360, uno por bot)
     → skill `read-automation-anywhere-export`. Este tipo de fuente
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

   En modo proyecto completo el lote es uno solo, `<slug>-completo-<fecha>`,
   y por dentro se separa por nivel de precedencia para que la revisión
   humana pueda leerlo en el mismo orden en que se construyó:

   ```
   obsidian-brain/_staging/<slug>-completo-YYYY-MM-DD/
   ├── _INFORME.md        ← inventario: qué se leyó, qué se omitió y por cuál regla
   ├── 01-export-aa/
   ├── 02-ddl/
   ├── 03-documentacion/
   └── 04-otros/
   ```

   `_INFORME.md` no es opcional: es lo que permite responder "¿de dónde
   salió esto?" y "¿qué quedó afuera?" sin volver a recorrer `_inbox/`.
6. **Nunca copiar el insumo al vault.** El archivo se queda en `_inbox/`;
   la nota de tipo `Documento` guarda su ruta relativa (ej.
   `_inbox/export/manifest.json`). El vault es solo markdown.
7. **Resumir al usuario**: cuántas notas de cada tipo se propusieron,
   qué bloques de la tabla del paso 2 quedaron sin cubrir, y cualquier
   zona del documento que no se pudo interpretar con confianza. El resumen
   de lo que FALTA es tan importante como el de lo que se extrajo. En modo
   proyecto completo, agregar además: archivos leídos por nivel, archivos
   omitidos por cada regla de exclusión, y las `Inconsistencia` abiertas
   entre niveles.

La revisión humana del staging y el paso a vault final es responsabilidad
del `graph-writer-agent`, no de este pipeline.

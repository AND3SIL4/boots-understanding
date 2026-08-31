---
name: vault-conventions
description: "Use this before reading or writing ANY note inside obsidian-brain/. Defines the note types, required frontmatter fields, folder layout, and the typed relation sections that make the vault behave like a graph instead of a pile of markdown files. Any agent that touches obsidian-brain/ must follow this."
---

# Convenciones del vault

El vault es un grafo de conocimiento almacenado como notas markdown de Obsidian.
Cada nota es un nodo. Las relaciones son secciones con encabezado fijo (no
wikilinks sueltos en prosa) para que sean identificables de forma consistente.

## Tipos de nodo válidos

`Proceso`, `PasoDeProceso`, `ReglaDeNegocio`, `Sistema`, `CampoDeDato`,
`Excepcion`, `HistoriaDeUsuario`, `Robot`, `Documento`.

Las plantillas de cada tipo están en `obsidian-brain/_templates/`. Siempre partir de
la plantilla correspondiente, nunca escribir una nota desde cero.

## Frontmatter obligatorio (todas las notas)

```yaml
tipo: <uno de los tipos válidos>
id: <PREFIJO-NNN, ej. RN-003, PASO-012>
source_doc: "[[Documento - nombre]]"
confidence: alta | media | baja
estado: propuesto | verificado
last_verified_date: YYYY-MM-DD
```

`confidence` refleja qué tan literal es la extracción del documento fuente.
`estado: propuesto` significa que viene de ingestión sin revisar; solo pasa a
`verificado` cuando un humano lo confirma.

## Secciones de relación permitidas (encabezados fijos)

Usar exactamente estos encabezados cuando apliquen — son los que el resto
del sistema busca para reconstruir el grafo:

- `## Depende de`
- `## Aplica en` / `## Aplica regla`
- `## Interactúa con`
- `## Ocurre en`
- `## Sigue a`
- `## Implementada por` / `## Automatiza`
- `## Excepciones conocidas`
- `## Usado en` / `## Usado por`
- `## Extraído de` (obligatoria en toda nota — trazabilidad al documento fuente)

Cada ítem de estas listas debe ser un wikilink `[[...]]` a otra nota del
vault, no texto libre.

## Layout de carpetas

Los agentes se ejecutan desde la raíz del proyecto. Desde ahí hay dos
árboles separados, y la separación es deliberada:

```
<raíz de ejecución>/
├── _inbox/            ← insumos crudos, CUALQUIER tipo de archivo
└── obsidian-brain/    ← el vault de Obsidian (un solo vault, solo markdown)
    ├── _templates/
    ├── _staging/
    └── proyectos/
        └── <slug-del-proyecto>/
```

- **`_inbox/` vive en la raíz, FUERA del vault.** Ahí van los insumos tal
  como llegan: `.pdf`, `.docx`, `.xlsx`, `.csv`, `.txt`, `.md`, exports de
  Automation Anywhere (`_inbox/export/`), videos, audios, capturas. El
  vault nunca contiene binarios — así Obsidian abre un grafo limpio de
  markdown y el inbox puede ser tan heterogéneo como haga falta.
- **`obsidian-brain/` es el vault**, y es uno solo. Se abre esa carpeta
  (no la raíz) como vault en la app de Obsidian.
- **`obsidian-brain/proyectos/<slug-del-proyecto>/`** es el grafo
  verificado de cada proyecto RPA. Un solo vault, una carpeta por proyecto.
- Si hay varios proyectos en curso, agrupar también los insumos por
  proyecto: `_inbox/<slug-del-proyecto>/...`.

## Naming y ubicación de archivos

- Nombre de archivo: `<Tipo> - <nombre-corto-descriptivo>.md`
  (ej. `ReglaDeNegocio - aprobacion-gerencial-monto-alto.md`).
- Ubicación: `obsidian-brain/proyectos/<slug-del-proyecto>/<tipo-en-plural>/archivo.md`
  (ej. `obsidian-brain/proyectos/conciliacion-facturas/reglas/ReglaDeNegocio - ....md`).
- Nunca escribir directamente en `obsidian-brain/proyectos/<proyecto>/`:
  las notas nuevas o propuestas van primero a
  `obsidian-brain/_staging/<lote>/` (ver skill `ingestion-pipeline`) y solo
  se mueven a su ubicación final cuando un humano las aprueba.
- Nunca copiar un insumo de `_inbox/` dentro del vault. La nota de tipo
  `Documento` referencia la ruta del archivo en `_inbox/`; el archivo se
  queda donde está.

## Regla de oro

Si una nota afirma algo que no está en el documento fuente, no se escribe.
Mejor un campo vacío o una nota "el documento no especifica esto" que un
dato inventado — este vault es la fuente de verdad para procesos que pueden
tocar pagos, RRHH o cumplimiento.

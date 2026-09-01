---
name: vault-conventions
description: "Use this before reading or writing ANY note inside obsidian-brain/. Defines the note types, required frontmatter fields, folder layout, and the typed relation sections that make the vault behave like a graph instead of a pile of markdown files. Any agent that touches obsidian-brain/ must follow this."
---

# Convenciones del vault

El vault es un grafo de conocimiento almacenado como notas markdown de Obsidian.
Cada nota es un nodo. Las relaciones son secciones con encabezado fijo (no
wikilinks sueltos en prosa) para que sean identificables de forma consistente.

## Tipos de nodo válidos

Las plantillas de cada tipo están en `obsidian-brain/_templates/`. Siempre
partir de la plantilla correspondiente, nunca escribir una nota desde cero.

### Multi-instancia — el grafo (26 tipos)

Todas las carpetas de esta tabla cuelgan de
`obsidian-brain/proyectos/<slug-del-proyecto>/documentacion/`.

| Tipo | Prefijo id | Carpeta | Qué documenta |
| --- | --- | --- | --- |
| `Proceso` | `PROC` | `procesos/` | El proceso de negocio completo |
| `PasoDeProceso` | `PASO` | `pasos/` | Cada paso del flujo de trabajo |
| `ReglaDeNegocio` | `RN` | `reglas/` | Condición → acción: qué *hacer* |
| `Validacion` | `VAL` | `validaciones/` | Criterio de aceptación de un dato: si es *válido* |
| `Sistema` | `SIS` | `sistemas/` | Plataforma RPA, ERP/SAP, base de datos, aplicación, servicio |
| `Ambiente` | `AMB` | `ambientes/` | Dev/QA/UAT/prod, red y conectividad |
| `Acceso` | `ACC` | `accesos/` | Usuarios de servicio, permisos, custodia de secretos |
| `CampoDeDato` | `CD` | `campos/` | Dato de negocio que el proceso maneja |
| `Parametro` | `PAR` | `parametros/` | Variable o configuración técnica del bot |
| `Insumo` | `INS` | `insumos/` | Archivo o fuente de entrada |
| `Salida` | `SAL` | `salidas/` | Archivo, registro o reporte producido |
| `Notificacion` | `NOT` | `notificaciones/` | Correos, alertas, mensajes |
| `Excepcion` | `EXC` | `excepciones/` | Caso borde que el proceso YA contempla |
| `Incidencia` | `INCID` | `incidencias/` | Falla real ocurrida (incluye stoppers) |
| `Riesgo` | `RSG` | `riesgos/` | Amenaza que puede no estar cubierta |
| `HistoriaDeUsuario` | `HU` | `historias/` | Requisito en formato historia |
| `Actor` | `ACT` | `actores/` | Roles, áreas, dueños del proceso |
| `Robot` | `BOT` | `robots/` | El bot que automatiza el proceso |
| `Indicador` | `KPI` | `indicadores/` | KPIs y SLAs |
| `PruebaUAT` | `UAT` | `pruebas/` | Casos de prueba ejecutados y su resultado |
| `Decision` | `DEC` | `decisiones/` | Decisión tomada, alternativas, consecuencias |
| `Hito` | `HITO` | `cronologia/` | Evento fechado en la vida del proyecto |
| `Termino` | `TER` | `glosario/` | Término del dominio tal como se usa aquí |
| `PreguntaFrecuente` | `FAQ` | `faq/` | Pregunta recurrente + respuesta citada |
| `Inconsistencia` | `INCON` | `inconsistencias/` | Dos fuentes que se contradicen, sin resolver |
| `Documento` | `DOC` | `documentos/` | Fuente, referencia externa o anexo |

### Singleton — una sola nota por proyecto, en la raíz de `documentacion/`

| Tipo | Archivo | Qué documenta |
| --- | --- | --- |
| `ResumenEjecutivo` | `_overview.md` | Punto de entrada: entender el proceso en 5 minutos |
| `Proyecto` | `Proyecto - <slug>.md` | Contexto, alcance, objetivos, beneficios |
| `Arquitectura` | `Arquitectura - <slug>.md` | Vista técnica de conjunto e integraciones |
| `MejoresPracticas` | `MejoresPracticas - <slug>.md` | Cómo tocar ESTE proyecto sin romperlo |
| `Pendientes` | `Pendientes - <slug>.md` | Cobertura de documentación y preguntas abiertas |

Si un tipo singleton ya existe para el proyecto, se **edita**, nunca se
crea una segunda nota del mismo tipo.

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

`source_doc` admite `n/a` solo en notas derivadas del propio vault
(`Pendientes`), y una lista de dos o más documentos en `Inconsistencia`.

### Campos adicionales por tipo

Se agregan al frontmatter obligatorio, no lo reemplazan:

- `Sistema` → `categoria: plataforma-rpa | erp | base-de-datos | aplicacion | servicio | archivo`
- `Hito`, `Decision` → `fecha: YYYY-MM-DD`
- `Incidencia` → `fecha`, `severidad: stopper | alta | media | baja`, `estado_incidencia: abierta | cerrada | mitigada`
- `Riesgo` → `severidad: alta | media | baja`
- `Indicador` → `clase: kpi | sla`
- `PruebaUAT` → `fecha`, `resultado: aprobada | fallida | parcial | no ejecutada`
- `Inconsistencia` → `estado_inconsistencia: abierta | resuelta`

Cuando el documento fuente no da el valor de un campo adicional, dejarlo
vacío y anotarlo en `Pendientes` — nunca elegir un valor "razonable".

## Secciones de relación permitidas (encabezados fijos)

Usar exactamente estos encabezados cuando apliquen — son los que el resto
del sistema busca para reconstruir el grafo. No inventar encabezados
nuevos: si una relación no encaja en ninguno, describirla en prosa y
dejarla fuera del grafo.

**Composición** (de un nodo contenedor hacia sus partes)
- `## Pasos (en orden)`
- `## Sistemas involucrados`
- `## Robots asociados`
- `## Procesos`
- `## Insumos` / `## Salidas`

**Flujo y lógica**
- `## Sigue a`
- `## Depende de`
- `## Aplica en` / `## Aplica regla` / `## Aplica validación`
- `## Valida`
- `## Ocurre en`
- `## Excepciones conocidas`

**Técnicas**
- `## Interactúa con`
- `## Se ejecuta en` / `## Ejecuta`
- `## Requiere acceso a`
- `## Usa parámetros`
- `## Implementada por` / `## Automatiza`
- `## Usado en` / `## Usado por` / `## Validado por`

**Datos y flujo de archivos**
- `## Consume` / `## Consumido por`
- `## Produce` / `## Producida por`
- `## Notifica a`

**Personas y gobierno**
- `## Ejecutado por` / `## Participa en` / `## Responsable de`
- `## Actores involucrados`

**Medición y calidad**
- `## Mide` / `## Medido por`
- `## Verifica` / `## Verificada por`
- `## Riesgos`
- `## Afecta a`
- `## Contradice`

**Semántica y consulta**
- `## Aparece en`
- `## Responde sobre`

**Calidad del propio vault**
- `## Inconsistencias sin resolver`

**Trazabilidad**
- `## Extraído de` — obligatoria en toda nota, con tres excepciones:
  `Documento` (es la fuente, no se extrae de otra), `Pendientes` y
  `ResumenEjecutivo` (se derivan del vault completo, no de un documento).

Cada ítem de estas listas debe ser un wikilink `[[...]]` a otra nota del
vault, no texto libre.

Las notas singleton tienen además **secciones narrativas** con encabezado
propio (`## Contexto`, `## Alcance`, `## Objetivos`, `## Beneficios`,
`## Piezas`, `## Integraciones`...). Esas son contenido, no relaciones: no
se parsean como aristas del grafo y pueden llevar prosa libre.

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
            ├── <slug>.md                   ← nota raíz del proyecto
            ├── notas-generales.md
            ├── control-cambios/            ┐
            ├── desarrollo/                 │ trabajo humano:
            ├── reuniones/                  │ la ingestión no escribe aquí
            ├── soporte/                    ┘
            └── documentacion/              ← EL GRAFO: todo lo que produce la ingestión
                ├── _overview.md                ← resumen ejecutivo
                ├── Proyecto - <slug>.md        ← contexto, alcance, objetivos
                ├── Arquitectura - <slug>.md
                ├── MejoresPracticas - <slug>.md
                ├── Pendientes - <slug>.md
                └── <carpeta-por-tipo>/         ← ver tabla de tipos
```

- **`_inbox/` vive en la raíz, FUERA del vault.** Ahí van los insumos tal
  como llegan: `.pdf`, `.docx`, `.xlsx`, `.csv`, `.txt`, `.md`, exports de
  Automation Anywhere (`_inbox/export/`), videos, audios, capturas. El
  vault nunca contiene binarios — así Obsidian abre un grafo limpio de
  markdown y el inbox puede ser tan heterogéneo como haga falta.
- **`obsidian-brain/` es el vault**, y es uno solo. Se abre esa carpeta
  (no la raíz) como vault en la app de Obsidian.
- **`obsidian-brain/proyectos/<slug-del-proyecto>/`** es la carpeta de cada
  proyecto RPA. Un solo vault, una carpeta por proyecto.
- **El grafo verificado vive en `<slug-del-proyecto>/documentacion/`**, y
  esa carpeta la genera y mantiene la ingestión. Las hermanas
  (`control-cambios/`, `desarrollo/`, `reuniones/`, `soporte/`) son notas
  de trabajo humano: se pueden enlazar desde y hacia el grafo, pero ningún
  agente de ingestión o escritura de grafo escribe dentro de ellas.
- La carpeta `documentacion/` la crea `/nuevo-proyecto` a partir de
  `proyecto-plantilla`. Si no existe, el proyecto está mal creado: hay que
  informarlo, no crearla al vuelo.
- Las carpetas por tipo se crean **solo cuando hay al menos una nota** de
  ese tipo. No pre-crear las 26 vacías.
- Si hay varios proyectos en curso, agrupar también los insumos por
  proyecto: `_inbox/<slug-del-proyecto>/...`.

## Naming y ubicación de archivos

- Nombre de archivo: `<Tipo> - <nombre-corto-descriptivo>.md`
  (ej. `ReglaDeNegocio - aprobacion-gerencial-monto-alto.md`).
- Ubicación: `obsidian-brain/proyectos/<slug-del-proyecto>/documentacion/<carpeta-del-tipo>/archivo.md`
  (ej. `obsidian-brain/proyectos/conciliacion-facturas/documentacion/reglas/ReglaDeNegocio - ....md`).
- Nunca escribir directamente en `obsidian-brain/proyectos/<proyecto>/`:
  las notas nuevas o propuestas van primero a
  `obsidian-brain/_staging/<lote>/` (ver skill `ingestion-pipeline`) y solo
  se mueven a su ubicación final cuando un humano las aprueba.
- Nunca copiar un insumo de `_inbox/` dentro del vault. La nota de tipo
  `Documento` referencia la ruta del archivo en `_inbox/`; el archivo se
  queda donde está.

## Distinciones que se confunden seguido

Estos pares son los que más se mezclan al ingerir. Si dudas, escribe la
nota en el tipo más conservador y anótalo en `Pendientes`:

- **`ReglaDeNegocio` vs `Validacion`** — la regla decide *qué hacer*
  ("si el monto supera 10M, va a aprobación gerencial"); la validación
  decide *si el dato sirve* ("el NIT debe tener 9 dígitos").
- **`Excepcion` vs `Incidencia` vs `Riesgo`** — la excepción es un caso
  borde que el proceso YA contempla; la incidencia es una falla que
  **ocurrió** (con fecha); el riesgo es una amenaza que **podría** ocurrir
  y puede no tener mitigación.
- **`CampoDeDato` vs `Parametro`** — el campo es dato de negocio que
  fluye por el proceso; el parámetro es configuración técnica del bot.
- **`Sistema` vs `Ambiente`** — el sistema es *qué* software se usa; el
  ambiente es *dónde* corre (dev/QA/prod) y con qué conectividad.
- **`Insumo`/`Salida` vs `Documento`** — insumo y salida son archivos que
  el proceso consume o produce en cada ejecución; `Documento` es la fuente
  de la que se extrajo el conocimiento (un manual, un correo, un export).

## Seguridad

Las notas `Acceso` describen qué permisos hacen falta y dónde se custodian,
**nunca el secreto en sí**. Si un documento fuente trae una contraseña,
token, cadena de conexión con credenciales o llave, no se transcribe al
vault: se registra que existe, dónde está, y se anota en `Pendientes` que
la fuente contiene un secreto expuesto.

## Regla de oro

Si una nota afirma algo que no está en el documento fuente, no se escribe.
Mejor un campo vacío o una nota "el documento no especifica esto" que un
dato inventado — este vault es la fuente de verdad para procesos que pueden
tocar pagos, RRHH o cumplimiento.

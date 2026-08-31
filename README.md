# Boost Your Process Understanding

Base de conocimiento para entender rápido proyectos RPA legados, construida
como un vault de Obsidian + skills y subagentes portables entre Claude Code
y OpenCode. Sin MCP ni vectorización (por el momento) — esta es la versión mínima
para trabajar directamente desde la CLI (ver `docs/arquitectura-v0.1.md`
para el diseño completo hacia el que esto evoluciona).

## Estructura

```
<raíz donde se ejecutan los agentes>/
├── _inbox/                    ← insumos crudos, CUALQUIER tipo de archivo
│   ├── export/                ← exports de Automation Anywhere (manifest.json + fuente)
│   └── ...                    ← pdf, docx, xlsx, csv, md, txt, video, audio, imágenes...
├── obsidian-brain/            ← EL vault de Obsidian (un solo vault, solo markdown)
│   ├── _templates/            ← plantilla por cada tipo de nodo
│   ├── _staging/              ← notas propuestas por ingestión, pendientes de revisión
│   └── proyectos/             ← el grafo verificado, una carpeta por proyecto RPA
│       └── <proyecto>/
│           ├── _overview.md   ← resumen ejecutivo (autogenerado)
│           ├── Proyecto - <slug>.md
│           ├── Arquitectura - <slug>.md
│           ├── MejoresPracticas - <slug>.md
│           ├── Pendientes - <slug>.md
│           └── <carpeta-por-tipo>/  ← procesos/ pasos/ reglas/ ... (26 tipos)
├── .claude/
│   ├── skills/                ← skills genéricas (formato Agent Skills, estándar abierto)
│   └── agents/                ← subagentes en formato Claude Code
└── .opencode/
    └── agent/                 ← los mismos 5 subagentes en formato OpenCode
```

**Por qué solo hay una carpeta de skills:** OpenCode descubre
automáticamente skills en `.claude/skills/*/SKILL.md` además de las suyas
propias, así que las 6 skills genéricas funcionan en ambas herramientas sin
duplicar nada. Los subagentes sí están duplicados porque el formato de
ubicación/frontmatter difiere entre Claude Code (`.claude/agents/`) y
OpenCode (`.opencode/agent/`) — el contenido del prompt es equivalente en
ambos.

## Qué se documenta

El esquema cubre 37 bloques de documentación, repartidos en 26 tipos de
nodo multi-instancia (el grafo) y 5 notas singleton por proyecto (lo
narrativo). El detalle vive en la skill `vault-conventions`; este es el
mapa de qué bloque cae en qué tipo:

| Bloque de documentación | Dónde vive |
| --- | --- |
| Resumen ejecutivo | `_overview.md` (singleton) |
| Proyecto · Alcance y objetivos · Contexto · Beneficios | `Proyecto` (singleton) |
| Arquitectura | `Arquitectura` (singleton) |
| Mejores prácticas | `MejoresPracticas` (singleton) |
| Pendientes de documentación | `Pendientes` (singleton) |
| Actores y roles | `Actor` |
| Glosario | `Termino` |
| Cronología | `Hito` |
| Decisiones | `Decision` |
| Plataforma RPA · SAP · Base de datos · Aplicaciones y servicios | `Sistema` (campo `categoria`) |
| Red y ambientes | `Ambiente` |
| Seguridad y accesos | `Acceso` |
| Proceso de negocio | `Proceso` |
| Flujo de trabajo | `PasoDeProceso` |
| Historias de usuario | `HistoriaDeUsuario` |
| Reglas de negocio | `ReglaDeNegocio` |
| Validaciones | `Validacion` |
| Variables y parámetros | `Parametro` |
| Archivos e insumos | `Insumo` |
| Salidas y reportes | `Salida` |
| Correos y notificaciones | `Notificacion` |
| Excepciones | `Excepcion` |
| Riesgos | `Riesgo` |
| Incidencias y stoppers | `Incidencia` |
| KPIs y SLAs | `Indicador` |
| Pruebas UAT realizadas | `PruebaUAT` |
| Inconsistencias detectadas | `Inconsistencia` |
| Preguntas frecuentes | `PreguntaFrecuente` |
| Referencias · Anexos | `Documento` |
| Dependencias | sección de relación `## Depende de` (no es un tipo) |

Dos notas hacen el trabajo que un grafo solo no hace: `_overview.md` es la
entrada de 5 minutos para quien llega nuevo, y `Pendientes` lleva la
cuenta de qué bloques siguen sin cubrir y qué preguntas hay que hacerle a
un humano. Esa segunda es la que convierte la regla de oro en algo
accionable en vez de una excusa para dejar huecos.

## Las 6 skills genéricas

| Skill                | Rol                                                               |
| -------------------- | ----------------------------------------------------------------- |
| `vault-conventions`  | Esquema del grafo: tipos de nodo, frontmatter, relaciones tipadas |
| `cite-sources`       | Política: toda respuesta debe citar su nota fuente                |
| `ingestion-pipeline` | Cómo convertir un documento crudo en notas candidatas             |
| `socratic-method`    | Técnica de aprendizaje activo (perfil técnico)                    |
| `flow-simulation`    | Motor de simulaciones "qué pasa si"                               |
| `rpa-best-practices` | Conocimiento genérico de la industria RPA                         |

## Los 5 subagentes

| Subagente            | Hace qué                                          | Puede escribir en      |
| -------------------- | ------------------------------------------------- | ---------------------- |
| `ingestion-agent`    | Lee `_inbox/` → propone notas               | solo `obsidian-brain/_staging/` |
| `graph-writer-agent` | Toma staging aprobado → lo escribe al vault final | `obsidian-brain/` completo      |
| `qa-agent`           | Responde preguntas directas (usuario de negocio)  | nada (solo lectura)    |
| `tutor-agent`        | Modo socrático (usuario técnico)                  | nada (solo lectura)    |
| `simulation-agent`   | Simulaciones "qué pasa si"                        | nada (solo lectura)    |

## Flujo de trabajo

**1. Ingerir un documento nuevo**

1. Copia el archivo (pdf/docx/xlsx/md/txt) a `_inbox/`.
2. En tu CLI: pide al agente que lo procese (ej. "procesa el manual que
   acabo de agregar") — se delega a `ingestion-agent`, que propone notas en
   `obsidian-brain/_staging/<lote>/`.
3. Revisa a mano (o pídele a Claude/OpenCode que te resuma) las notas
   propuestas en `obsidian-brain/_staging/<lote>/`. Edita lo que haga falta.
4. Pide que se confirmen ("ya revisé, agrégalas al vault") — se delega a
   `graph-writer-agent`, que las mueve a su ubicación final y actualiza
   `_overview.md`.

**2. Consultar el conocimiento**

- Pregunta directa → `qa-agent` (respuesta con cita).
- Quiero entender el proceso a fondo → `tutor-agent` (modo socrático).
- Qué pasa si... → `simulation-agent`.

**3. Explorar visualmente**

Abre la carpeta `obsidian-brain/` como vault en la app de Obsidian para ver el grafo
de notas y navegar los wikilinks directamente.

## Fuera de alcance por ahora

Sin servidor MCP, sin índice vectorial, sin base de grafo formal (Kùzu/
Neo4j) — el vault en markdown y las búsquedas con Grep/Glob son suficientes
para el primer piloto. Ver `docs/arquitectura-v0.1.md` para el diseño hacia
el que esto escala cuando haga falta.

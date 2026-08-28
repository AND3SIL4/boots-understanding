# rpa-cerebro (nombre temporal)

Base de conocimiento para entender rápido proyectos RPA legados, construida
como un vault de Obsidian + skills y subagentes portables entre Claude Code
y OpenCode. Sin MCP ni vectorización todavía — esta es la versión mínima
para trabajar directamente desde la CLI (ver `docs/arquitectura-v0.1.md`
para el diseño completo hacia el que esto evoluciona).

## Estructura

```
rpa-cerebro/
├── vault/                     ← la base de conocimiento (el grafo)
│   ├── _templates/            ← plantilla por cada tipo de nodo
│   ├── _inbox/                ← documentos crudos sin procesar (pdf, docx, xlsx, md...)
│   ├── _staging/               ← notas propuestas por ingestión, pendientes de revisión
│   └── <proyecto>/            ← notas verificadas, una carpeta por proyecto RPA
│       ├── procesos/
│       ├── pasos/
│       ├── reglas/
│       ├── sistemas/
│       ├── campos/
│       ├── excepciones/
│       ├── historias/
│       ├── robots/
│       ├── documentos/
│       └── _overview.md       ← mapa corto autogenerado del proyecto
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

## Las 6 skills genéricas

| Skill | Rol |
|---|---|
| `vault-conventions` | Esquema del grafo: tipos de nodo, frontmatter, relaciones tipadas |
| `cite-sources` | Política: toda respuesta debe citar su nota fuente |
| `ingestion-pipeline` | Cómo convertir un documento crudo en notas candidatas |
| `socratic-method` | Técnica de aprendizaje activo (perfil técnico) |
| `flow-simulation` | Motor de simulaciones "qué pasa si" |
| `rpa-best-practices` | Conocimiento genérico de la industria RPA |

## Los 5 subagentes

| Subagente | Hace qué | Puede escribir en |
|---|---|---|
| `ingestion-agent` | Lee `vault/_inbox/` → propone notas | solo `vault/_staging/` |
| `graph-writer-agent` | Toma staging aprobado → lo escribe al vault final | `vault/` completo |
| `qa-agent` | Responde preguntas directas (usuario de negocio) | nada (solo lectura) |
| `tutor-agent` | Modo socrático (usuario técnico) | nada (solo lectura) |
| `simulation-agent` | Simulaciones "qué pasa si" | nada (solo lectura) |

## Flujo de trabajo

**1. Ingerir un documento nuevo**

1. Copia el archivo (pdf/docx/xlsx/md/txt) a `vault/_inbox/`.
2. En tu CLI: pide al agente que lo procese (ej. "procesa el manual que
   acabo de agregar") — se delega a `ingestion-agent`, que propone notas en
   `vault/_staging/<lote>/`.
3. Revisa a mano (o pídele a Claude/OpenCode que te resuma) las notas
   propuestas en `vault/_staging/<lote>/`. Edita lo que haga falta.
4. Pide que se confirmen ("ya revisé, agrégalas al vault") — se delega a
   `graph-writer-agent`, que las mueve a su ubicación final y actualiza
   `_overview.md`.

**2. Consultar el conocimiento**

- Pregunta directa → `qa-agent` (respuesta con cita).
- Quiero entender el proceso a fondo → `tutor-agent` (modo socrático).
- Qué pasa si... → `simulation-agent`.

**3. Explorar visualmente**

Abre la carpeta `vault/` como vault en la app de Obsidian para ver el grafo
de notas y navegar los wikilinks directamente.

## Instalar en tu CLI

**Claude Code:** abre este directorio como proyecto; `.claude/skills/` y
`.claude/agents/` se cargan automáticamente.

**OpenCode:** abre este directorio como proyecto; `.opencode/agent/` se
carga automáticamente, y `.claude/skills/` también (ver nota arriba).

## Fuera de alcance por ahora

Sin servidor MCP, sin índice vectorial, sin base de grafo formal (Kùzu/
Neo4j) — el vault en markdown y las búsquedas con Grep/Glob son suficientes
para el primer piloto. Ver `docs/arquitectura-v0.1.md` para el diseño hacia
el que esto escala cuando haga falta.

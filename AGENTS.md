# rpa-cerebro — contexto para la sesión principal

Este archivo lo carga automáticamente la sesión principal (OpenCode de
forma nativa; Claude Code vía el import en `CLAUDE.md`). Es el documento
que orquesta el proyecto: no contiene lógica de negocio de ningún proceso
RPA, contiene las reglas de cómo esta sesión debe delegar y comportarse.

## Qué es este repo

Un vault de Obsidian (`vault/`) usado como grafo de conocimiento sobre
procesos RPA, más un conjunto de subagentes y skills que lo leen, lo
completan y lo consultan. No hay servidor MCP ni base de datos: todo es
markdown en disco, leído con Read/Grep/Glob.

## Dónde está cada cosa

- `vault/` — el grafo (notas `.md`). `_templates/` tiene la plantilla de
  cada tipo de nodo. `_inbox/` es donde el usuario deja documentos crudos.
  `_staging/` son notas propuestas, sin revisar.
- `.claude/skills/` — skills (método + lectura de documentos). Ver
  `.claude/skills/vault-conventions/SKILL.md` primero: define el esquema
  completo del grafo.
- `.claude/agents/` y `.opencode/agent/` — los 5 subagentes, mismo
  comportamiento en ambos formatos.
- `docs/arquitectura-v0.1.md` — diseño completo y su razonamiento.

## Regla de oro (no negociable)

Si un dato no está en el vault, decirlo explícitamente — nunca completarlo
con una suposición. Esto aplica a esta sesión tanto como a cualquier
subagente que delegue.

## Cuándo delegar a cada subagente

| El usuario dice algo como...                              | Delegar a            |
| --------------------------------------------------------- | -------------------- |
| "procesa este documento", "ingiere lo que hay en \_inbox" | `ingestion-agent`    |
| "ya revisé el staging, agrégalo al vault"                 | `graph-writer-agent` |
| Una pregunta directa sobre el proceso                     | `qa-agent`           |
| "quiero entender esto", "enséñame"                        | `tutor-agent`        |
| "qué pasa si...", "simula..."                             | `simulation-agent`   |

No respondas preguntas sobre un proceso documentado directamente desde la
sesión principal sin pasar por `qa-agent`/`tutor-agent`/`simulation-agent`
— son los que aplican `cite-sources` de forma consistente.

## Flujo esperado

`_inbox` → `ingestion-agent` → `_staging` → revisión humana →
`graph-writer-agent` → vault final → consulta.

Ver el README para el paso a paso completo con ejemplos.

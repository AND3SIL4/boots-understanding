---
name: read-automation-anywhere-export
description: "Use when a file in vault/_inbox/ is an Automation Anywhere bot export — .atmx, .mbot, .aapkg, or an XML/TXT export already converted from one of those. Extracts the bot's actual technical logic (variables, action sequence, error handling, dependencies) so it can be turned into vault notes, same as any manual — but this source is the ground truth for 'qué hace el bot realmente', no lo que dice el manual."
---

# Leer exports de bots de Automation Anywhere

Esta es la fuente más confiable de qué hace el bot **en realidad** — a
diferencia de un manual, que describe lo que se supone que debería hacer.
Cuando ambas fuentes existen para el mismo proceso, este export tiene
prioridad para responder "qué hace el bot", y el manual tiene prioridad
para responder "por qué" (la intención de negocio). Las carpetas siempre se van a encontrar en la siguiente ubicación `vault/_inbox/export` dentro de export encontraras todo lo relacionado a export del asistente digital, esto incluye un archivo llamado `manifest.json` el cuál usarás como indicé, y también los las carpetas y archivos que contienen el código fuente

## Formatos que puedes encontrar

| Formato               | Qué es                                                                                      | Cómo leerlo                                                                                                                                                                                                                                                                                          |
| --------------------- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.atmx`               | Task Bot de Enterprise 10/11. Formato propietario, no es texto plano legible directamente.  | No lo proceses en crudo. Pide al usuario que lo exporte como XML o TXT primero (`AAWorkbench.exe /export /sf:origen.atmx /df:destino.xml`, o una herramienta de conversión atmx→XML). Una vez convertido, es texto/XML normal.                                                                       |
| `.mbot`               | MetaBot de Enterprise 10/11.                                                                | Mismo tratamiento que `.atmx`.                                                                                                                                                                                                                                                                       |
| `.aapkg`              | Paquete de exportación de Automation 360 (Control Room → Export). Es un archivo comprimido. | Descomprímelo primero e **inspecciona la estructura real que encuentres** antes de asumir un esquema fijo — el formato interno puede variar entre versiones. Reporta lo que efectivamente hay (carpetas, nombres de archivo, extensión de cada archivo de bot) en vez de dar por sentado un formato. |
| XML/TXT ya convertido | El caso más simple.                                                                         | Léelo como texto/XML estructurado normal.                                                                                                                                                                                                                                                            |

**Si el archivo no se puede interpretar en absoluto** (formato binario no
soportado, `.aapkg` protegido con contraseña, etc.): repórtalo al usuario y
pide una versión convertida o el archivo sin protección, en vez de simular
contenido a partir del nombre del archivo.

## Qué extraer, cuando el formato lo permite

- **Variables declaradas** (nombre, tipo, valor por defecto si lo tiene)
  → candidatas a `CampoDeDato`.
- **Secuencia de comandos/acciones en el orden real en que ocurren**
  (clics, extracción de pantalla, llamadas a sistemas, `IF`/`Loop`,
  llamadas a subrutinas o MetaBots) → candidatas a `PasoDeProceso`, en
  orden, con relación `## Sigue a` reflejando la secuencia real del bot.
- **Bloques de manejo de errores** (Error Handler / Try-Catch) →
  candidatas a `Excepcion`, con lo que el bot realmente hace ante el error
  (reintenta, notifica, detiene) — esto suele faltar por completo en los
  manuales de negocio.
- **Condiciones `IF`/`Else`** que reflejan una decisión de negocio (no solo
  lógica técnica interna del bot) → candidatas a `ReglaDeNegocio`.
- **Sistemas/aplicaciones** que el bot abre o con los que interactúa
  (rutas de ejecutables, URLs, nombres de ventana) → candidatas a
  `Sistema`.
- **Dependencias**: subrutinas, MetaBots, paquetes (packages) que el bot
  llama — importante para entender qué se rompe si se toca algo.
- **Variables quemadas**: variables que contienen el valor asignado directamente en el código
- **Bugs**: una lista compacta de los bugs o posibles fallas encontrados directamente en el código fuente.

## Reglas específicas de este tipo de fuente

1. **Nunca extraigas ni escribas valores de credenciales, contraseñas o
   tokens** que puedan aparecer en el export, aunque estén en texto plano.
   Represéntalos únicamente como "usa credencial: `<nombre-de-la-credencial>`",
   nunca el valor.
2. **Marca explícitamente las discrepancias con la documentación de
   negocio.** Si el export muestra que el bot hace algo distinto a lo que
   dice el manual (ej. el manual dice "aprobación gerencial sobre $5M" y el
   bot compara contra $4.500.000), no elijas una fuente en silencio —
   propón ambas notas y anota la discrepancia con claridad para que la
   revisión humana decida cuál es la vigente.
3. Toda nota que salga de este export debe citar el archivo de export como
   `source_doc`, con una aclaración de que es "export técnico del bot", no
   un manual — así quien lea la nota sabe que viene del código real del
   bot y no de una descripción de negocio.

---
name: cite-sources
description: "Use this whenever answering any question about a documented process, rule, or system using the vault. Enforces that every factual claim is traceable to a specific note, and that missing information is stated explicitly instead of guessed."
---

# Citar fuentes

Política transversal para cualquier agente que responda preguntas usando
el vault (qa-agent, tutor-agent, simulation-agent).

## Reglas

1. Antes de responder, localizar la(s) nota(s) relevante(s) en `obsidian-brain/`
   (usar Grep/Glob sobre `id:`, `tipo:`, o el nombre del archivo).
2. Toda afirmación factual sobre el proceso debe venir acompañada del
   wikilink de la nota que la respalda, ej.: "el bot requiere aprobación
   gerencial si el monto supera $5M ([[ReglaDeNegocio - aprobacion-...]])".
3. Si la nota tiene `confidence: baja`, decirlo explícitamente en la
   respuesta ("según una extracción de baja confianza...").
4. Si la pregunta no puede responderse con lo que hay en el vault, decir
   con claridad "no encontré esto en el vault documentado" — nunca
   completar el vacío con conocimiento general de RPA o suposiciones.
5. No mezclar información de notas con `estado: propuesto` (sin revisar)
   sin advertirlo — son borradores, no verdad confirmada.

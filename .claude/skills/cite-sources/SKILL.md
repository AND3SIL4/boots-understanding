---
name: cite-sources
description: "Use this whenever answering any question about a documented process, rule, or system using the vault. Enforces that every factual claim is traceable to a specific note, that unreviewed notes are flagged as such, and that missing information is stated explicitly instead of guessed."
---

# Citar fuentes

Política transversal para cualquier agente que responda preguntas usando
el vault (qa-agent, tutor-agent, simulation-agent).

## Qué es citable y qué no

El universo de respuesta es **el grafo canónico de un proyecto**:

```
obsidian-brain/proyectos/<slug>/documentacion/
```

Y dentro de esa carpeta, **solo los archivos con frontmatter `tipo:`**. Todo
lo demás está fuera, y citarlo es un error:

| Fuera del universo citable | Por qué |
| --- | --- |
| `documentacion/_ingestas/` | Traza de la ingestión (`_INFORME*`, `_CANDIDATOS-FUSION`, `_COMMITEADO`). Habla **sobre** el grafo, no es el grafo. Un candidato a fusión describe dos versiones sin resolver: citarlo es presentar como hecho algo que nadie decidió. |
| `obsidian-brain/_templates/` | Plantillas vacías. Tienen `tipo:` y prosa de relleno, así que un Grep por tipo las devuelve junto a las notas reales. Se reconocen por `id: XXX-XXX` y `last_verified_date: YYYY-MM-DD`. |
| `obsidian-brain/_staging/` | Notas propuestas que aún no entraron al vault. Puede estar lleno entre un `/ingest` y un `/commit`. |
| `proyectos/<slug>/control-cambios/`, `desarrollo/`, `reuniones/`, `soporte/` | Notas de trabajo humano, no el grafo documentado. Se pueden mencionar como contexto, nunca como fuente de una regla del proceso. |

## Reglas

1. **Resolver el proyecto antes de responder.** El vault tiene varios
   proyectos y algunos se parecen mucho entre sí (procesos de conciliación
   contra SAP, con vocabulario casi idéntico). Si la pregunta no dice a cuál
   se refiere y hay más de uno poblado, pregúntalo — no lo adivines. Y di
   siempre en la respuesta sobre qué proyecto estás respondiendo. Mezclar
   dos proyectos en una misma respuesta es el error más difícil de detectar
   desde afuera, porque suena perfectamente coherente.

2. **Localizar la nota** con Grep/Glob sobre `id:`, `tipo:` o el nombre del
   archivo, dentro del universo citable de arriba.

3. **Toda afirmación factual lleva el wikilink de la nota que la respalda**,
   ej.: "el bot requiere aprobación gerencial si el monto supera $5M
   ([[ReglaDeNegocio - aprobacion-...]])".

4. **`confidence: baja` se dice.** "Según una extracción de baja
   confianza...".

5. **`estado: propuesto` se advierte siempre.** Es la regla más importante
   de esta skill.

   En el grafo canónico, `propuesto` **no significa "borrador"**: significa
   que la nota entró al vault sin que ningún humano la revisara, porque al
   escribir el lote se decidió seguir adelante. Es conocimiento consultable
   y **marcado como ambiguo**, y hoy es una porción grande del grafo.

   Cuando una respuesta se apoye en una nota `propuesto`:
   - Dilo en la respuesta misma, junto a la afirmación — no en una nota al
     pie que el lector puede saltarse.
   - Remite a `Pendientes - <slug>`, donde está listada bajo
     `## Sin revisión humana`.
   - Si **toda** la respuesta se apoya en notas `propuesto`, dilo antes de
     responder, no después.

   El motivo es concreto: estos procesos tocan compensación contable,
   pagos y cierres. Una regla de negocio que nadie validó, presentada como
   si lo estuviera, es peor que un "no lo sé".

6. **Lo que el vault no cubre se dice.** "No encontré esto en el vault
   documentado" — nunca completar el vacío con conocimiento general de RPA
   ni con lo que "cualquier bot razonable haría". Un hueco declarado es una
   respuesta correcta; una suposición plausible, no.

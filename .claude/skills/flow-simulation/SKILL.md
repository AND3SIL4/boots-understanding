---
name: flow-simulation
description: "Use this for hypothetical 'what if' questions about a process or user story (e.g. '/simular', 'qué pasa si falta el campo X'). Generic simulation mechanism: walks typed relations in the vault, never invents rules the vault doesn't contain."
---

# Motor de simulación de flujos

Técnica genérica para razonar sobre escenarios hipotéticos usando
únicamente lo que existe en el vault.

## Cómo aplicarlo

1. **Ubicar el nodo de partida**: el `PasoDeProceso` o `HistoriaDeUsuario`
   sobre el que el usuario pregunta.
2. **Recorrer sus relaciones tipadas** (`## Aplica regla`, `## Depende de`,
   `## Excepciones conocidas`, `## Sigue a`, `## Interactúa con`) para
   construir el contexto del escenario.
3. **Aplicar la condición hipotética del usuario** contra ese contexto:
   - Si el vault tiene una `Excepcion` o `ReglaDeNegocio` que cubre
     exactamente esa condición, describir el resultado citando esa nota.
   - Si el vault tiene información parcialmente relacionada pero no cubre
     el caso exacto, decirlo explícitamente: "el vault documenta X pero no
     este caso específico" — y opcionalmente sugerir que esto podría
     documentarse (candidato a nueva nota vía ingestión).
   - Si el vault no tiene nada relacionado, responder con claridad "no hay
     información en el vault para simular esto" — nunca inventar una regla
     de negocio plausible.
4. **Mostrar el camino recorrido**, no solo la conclusión: qué nodos se
   visitaron y en qué orden, para que el usuario pueda verificar el
   razonamiento (esto es parte del valor pedagógico de la simulación).

La fidelidad a "no inventar" es más importante que dar una respuesta
completa — un "no lo sé" trazable es más seguro que un escenario inventado
en un proceso que puede tocar pagos o cumplimiento.

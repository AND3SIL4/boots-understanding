---
description: Simula un escenario hipotético recorriendo las relaciones del vault
argument-hint: <escenario, ej. "el archivo llega con 0 filas">
---

Delega en el subagente `simulation-agent`.

**Escenario:** $ARGUMENTS

Si el escenario está vacío, pregunta cuál — no inventes uno.

El agente usa `flow-simulation` y `cite-sources`: parte del nodo que
corresponda, recorre las relaciones tipadas y describe el resultado
citando cada nota que usó.

El resultado debe distinguir con claridad tres casos:

1. **El vault cubre el escenario** → describir qué pasa, con citas.
2. **El vault lo cubre parcialmente** → decir hasta dónde llega la
   documentación y desde dónde sería especulación. No completar el resto.
3. **El vault no lo cubre** → decirlo, y proponer registrar la pregunta en
   `Pendientes - <proyecto>`. Un escenario sin cubrir es un hallazgo, no
   un fracaso de la simulación.

Nunca inventar una regla, una excepción o un manejo de error que el vault
no contenga — aunque sea lo que "cualquier bot razonable haría".

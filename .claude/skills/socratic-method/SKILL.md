---
name: socratic-method
description: "Use this for active-learning sessions where the user wants to understand a process, not just get the answer (e.g. '/aprender', 'quiero entender este proceso'). Generic pedagogical technique: no RPA-specific knowledge, just the method."
---

# Método socrático de aprendizaje activo

Técnica genérica, independiente de cualquier proceso concreto.

## Cómo aplicarlo

1. **No dar la respuesta directa primero.** Ubicar en el vault la nota que
   responde la pregunta del usuario, pero en vez de resumirla, hacer una
   pregunta guía que apunte hacia esa nota sin revelarla del todo.
2. **Esperar la respuesta del usuario** antes de seguir.
3. **Evaluar la respuesta contra el contenido real de la nota**:
   - Si acierta: confirmar qué acertó, citando la nota (`## Extraído de`
     vía `cite-sources`), y hacer la siguiente pregunta un nivel más
     profundo (de `Proceso` → `PasoDeProceso` → `ReglaDeNegocio`/`Excepcion`).
   - Si acierta parcialmente: reconocer la parte correcta, y volver a
     preguntar de forma más específica sobre lo que falta — no corregir de
     golpe con la respuesta completa.
   - Si se equivoca: no decir simplemente "incorrecto" — preguntar algo que
     lo lleve a notar el error por sí mismo (ej. "¿qué pasaría si el monto
     fuera exactamente $5.000.001?").
4. **Progresión de profundidad sugerida**: visión general del proceso →
   pasos en orden → regla de negocio de un paso → de qué depende esa regla
   → qué excepción puede romperla.
5. **Válvula de escape**: si el usuario pide explícitamente la respuesta
   directa ("solo dime", "no quiero el método socrático ahora"), dársela
   sin insistir en la técnica. El método sirve para aprender, no para
   frustrar.

Este skill no sabe nada de RPA — el contenido correcto contra el que se
evalúa al usuario siempre sale del vault, citado según `cite-sources`.

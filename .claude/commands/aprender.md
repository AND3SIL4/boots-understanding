---
description: Sesión socrática para entender un proceso documentado en el vault
argument-hint: <tema o proceso que quieres entender>
---

Delega en el subagente `tutor-agent`.

**Tema:** $ARGUMENTS

Si el tema está vacío, lista los proyectos que hay en
`obsidian-brain/proyectos/` y pregunta sobre cuál quiere trabajar.

El agente usa `socratic-method` y `cite-sources`: guía con preguntas en vez
de dar la respuesta directa, y todo lo que afirme sale del vault, citado.

Dos cosas que el agente NO debe hacer:

- Rellenar con conocimiento general de RPA lo que el vault no documenta.
  Si el vault no lo tiene, la respuesta correcta es "esto no está
  documentado" — y eso mismo es material de aprendizaje: le muestra al
  usuario dónde está ciego el proyecto.
- Seguir preguntando cuando el usuario ya pidió la respuesta directa. Si
  la pide, dásela citada y sigue.

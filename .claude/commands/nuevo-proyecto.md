---
description: Crea un nuevo proyecto en vault/proyectos/ a partir de vault/proyectos/proyecto-plantilla, usando el slug indicado.
argument-hint: [slug-del-proyecto]
allowed-tools: Bash, Read, Write, Glob
---

El slug pedido es: $ARGUMENTS

Sigue estos pasos, en orden, y detente en cualquiera de ellos si algo no
está bien — no improvises una alternativa:

1. **Si `$ARGUMENTS` viene vacío**, pide al usuario el slug del proyecto y
   detente aquí.

2. **Normaliza el slug**: minúsculas, sin acentos, espacios convertidos a
   guiones, solo letras/números/guiones. Si el valor que escribió el
   usuario ya cumple esto, úsalo tal cual. Si tuviste que cambiar algo,
   muéstrale la versión normalizada y pide confirmación antes de seguir.

3. **Verifica que `vault/proyectos/proyecto-plantilla/` existe.** Si no
   existe, avisa que falta la plantilla maestra y detente — no inventes
   una estructura de carpetas a partir de la memoria.

4. **Verifica que `vault/proyectos/<slug>/` NO existe todavía.** Si ya
   existe, avisa al usuario y detente — nunca sobrescribas un proyecto
   existente con este comando.

5. **Copia recursivamente** `vault/proyectos/proyecto-plantilla/` a
   `vault/proyectos/<slug>/`, preservando la estructura de subcarpetas
   (`procesos/`, `pasos/`, `reglas/`, `sistemas/`, `campos/`,
   `excepciones/`, `historias/`, `robots/`, `documentos/`) y sus
   `.gitkeep`.

6. **Completa `vault/proyectos/<slug>/_overview.md`**: reemplaza
   `{{slug}}` por el slug, `{{fecha}}` por la fecha de hoy, y `{{nombre}}`
   por una versión legible del slug (guiones → espacios, primera letra en
   mayúscula) — o pregunta al usuario si prefiere un nombre distinto antes
   de escribirlo.

7. **Reporta al usuario** la ruta creada y la lista de subcarpetas, y
   recuérdale que ya puede poner documentos en `vault/_inbox/` y pedir que
   se ingieran para este proyecto.

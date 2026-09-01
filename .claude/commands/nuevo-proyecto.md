---
description: Crea un nuevo proyecto en obsidian-brain/proyectos/ a partir de obsidian-brain/proyectos/proyecto-plantilla, usando el slug indicado.
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

3. **Verifica que `partir de obsidian-brain/proyectos/proyecto-plantilla/` existe.**
   Si no existe, avisa que falta la plantilla maestra y detente — no inventes una
   estructura de carpetas a partir de la memoria.

4. **Verifica que `obsidian-brain/proyectos/<slug>/` NO existe todavía.** Si ya
   existe, avisa al usuario y detente — nunca sobrescribas un proyecto
   existente con este comando.

5. **Copia recursivamente** `obsidian-brain/proyectos/proyecto-plantilla/` a
   `obsidian-brain/proyectos/<slug>/`, preservando la estructura de subcarpetas
   archivos, etc. Y sus `.gitkeep`.

6. **Completa `obsidian-brain/proyectos/<slug>/`**: recorre toda la plantilla
   y reemplaza todas las coincidencias del nombre del proyecto (guiones → espacios,
   primera letra en mayúscula) por el slug y las fechas por la fecha de creación de
   la plantilla, para nombrar los proyecto usa un incremento para el código del
   proyecto dependiendo de la cantidad de proyectos que hayan:
   `ej: si es el primer proyecto el código sería "PRJ-2026-001"`, si ya existen
   proyectos asigna el numeral según corresponda. Relaciona todos los archivos
   entre si con sus respectivos hijos y padres. Y los archivos de ejemplo se dejan
   tal cual como ejemplos solo se cablean sus relaciones `[[ ]]` no se enumeran
   porque el 000 marca que son plantillas.

7. **Remueve elementos no necesarios**: remueve los `callout` que nos necesitan
   en un proyecto real.

8. **Espera confirmación**: espera que el usuario te confirme para crear el nuevo
   proyecto, resume de manera compacta todo lo que encontraste y lo que vas a hacer,
   y espera la confirmación del usuario.

9. **Reporta al usuario** la ruta creada y la lista de subcarpetas, y
   recuérdale que ya puede poner documentos en `vault/_inbox/` y pedir que
   se ingieran para este proyecto.

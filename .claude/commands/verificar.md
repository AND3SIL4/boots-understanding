---
description: Marca en bloque como verificado el estado de las notas de un ámbito ya revisado por un humano
argument-hint: <slug-del-proyecto | ruta dentro de obsidian-brain/>
---

**Este comando no revisa nada. Solo registra que TÚ ya revisaste.**

Cambia `estado: propuesto` → `estado: verificado` en todas las notas del
ámbito indicado. Es un atajo mecánico para cuando ya leíste el material y no
quieres abrir nota por nota en Obsidian.

**Ámbito:** $1

## Cuándo usarlo — y cuándo no

Úsalo **solo** si de verdad revisaste el contenido de ese ámbito y lo das
por bueno. `verificado` es una afirmación con consecuencias: significa que
un humano validó esa nota, y a partir de ahí el vault la trata como
conocimiento confiable. `graph-writer-agent` no vuelve a sobrescribir una
nota verificada con información que la contradiga — crea una
`Inconsistencia` en su lugar.

Marcar en bloque sin haber leído convierte esa garantía en ruido, y el
daño no es visible: las notas siguen ahí, con la misma pinta, pero ya nadie
sabe cuáles miró alguien. **No lo uses para "limpiar" el reporte de
ambigüedades del `/commit`.** Para eso está `Pendientes`: una nota en
`propuesto` es un pendiente honesto, y es preferible a un `verificado`
falso.

Si solo revisaste una parte, acota el ámbito a esa parte. Es la razón por la
que el comando recibe una ruta y no un proyecto entero por defecto.

## Qué toca

**Únicamente la línea `estado:` del frontmatter.** Nada más:

- No toca `last_verified_date`, `confidence`, `source_doc` ni ningún otro
  campo.
- No toca `estado_incidencia` ni `estado_inconsistencia`, que son campos
  distintos con valores propios.
- No toca el cuerpo de la nota, ni sus secciones de relación.
- No mueve, crea ni borra archivos.
- Nunca degrada `verificado` a `propuesto`: solo promueve.

## Cómo ejecutarlo

1. **Resuelve el ámbito.** `$1` puede ser:
   - un slug de proyecto → todos sus lotes en
     `obsidian-brain/_staging/<slug>-*/`
   - una ruta a un lote, o a una subcarpeta dentro de un lote (un nivel de
     precedencia, o un tipo de nota concreto)
   - una ruta dentro de `obsidian-brain/proyectos/<slug>/documentacion/`,
     para promover notas que ya entraron al grafo como ambiguas

   Si `$1` está vacío, lista los ámbitos disponibles con cuántas notas en
   `propuesto` tiene cada uno y pregunta. Nunca asumas "todo el vault".

2. **Cuenta antes de tocar.** Reporta cuántas notas hay en el ámbito,
   cuántas están en `propuesto` y cuántas ya en `verificado`.

3. **Pide confirmación explícita**, recordando que estás afirmando que esas
   notas fueron revisadas por una persona.

4. **Aplica el cambio** solo a la línea del frontmatter, con `sed` anclado
   al comienzo de línea y limitado a la primera coincidencia del archivo:

   ```bash
   find <ámbito> -name '*.md' -print0 \
     | xargs -0 -r sed -i '0,/^estado: propuesto$/s//estado: verificado/'
   ```

   El ancla `^...$` evita tocar `estado_incidencia`/`estado_inconsistencia`,
   y el rango `0,/re/` limita el cambio a la primera aparición — la del
   frontmatter — aunque el cuerpo contenga esa misma línea.

5. **Verifica y reporta**: cuántas quedaron en `verificado`, y confirma que
   no quedó ninguna en `propuesto` dentro del ámbito. Si quedó alguna,
   nómbrala: probablemente su frontmatter está mal formado y eso es un
   hallazgo, no un error del comando.

## Nota sobre finales de línea

En Windows, `sed -i` reescribe el archivo con finales de línea LF. Si alguna
nota venía con CRLF, queda normalizada a LF. Es irrelevante para el
contenido y para Obsidian, pero hace que esos archivos aparezcan como
"cambiados por completo" en un `diff` a secas. Para comprobar que solo
cambió el `estado`, compara ignorando el `\r`:

```bash
diff <(tr -d '\r' < <copia-previa>) <archivo>
```

Vale la pena tener una copia previa del ámbito antes de un cambio masivo:
`obsidian-brain/` está en `.gitignore`, así que git no sirve de red.

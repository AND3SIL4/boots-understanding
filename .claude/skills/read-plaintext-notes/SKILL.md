---
name: read-plaintext-notes
description: "Use when a raw document in vault/_inbox/ is a .txt, .md, or informal note (meeting notes, chat export, email pasted as text). Lower default confidence than structured documents, since these sources are usually informal."
---

# Leer notas y texto plano

## Cómo tratarlas

1. Léelas tal cual, sin transformación especial de formato.
2. Por defecto, cualquier nota extraída de aquí lleva `confidence: media`
   o `baja` (nunca `alta` automáticamente) — a menos que la nota cite
   explícitamente un documento o manual formal como respaldo, en cuyo caso
   la fuente real es ese documento, no la nota informal.
3. Si el texto tiene indicios de autor/fecha (firma de correo, encabezado
   de chat, "Nota de X — DD/MM/AAAA"), consérvalos en el frontmatter de la
   nota `Documento` — quién dijo qué y cuándo importa para poder verificar
   después si sigue vigente.
4. Si el texto es ambiguo o contradice otra fuente ya en el vault, no lo
   descartes ni lo sobrescribas: propón la nota de todas formas con la
   contradicción anotada explícitamente, para que la revisión humana
   decida cuál prevalece.

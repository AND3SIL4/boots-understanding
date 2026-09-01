---
tipo: Pendientes
id: PEND-001
source_doc: n/a
confidence: alta
estado: propuesto
last_verified_date: YYYY-MM-DD
---

# PEND-001 — Pendientes de documentación de <Nombre del proyecto>

Qué NO sabemos todavía. Esta nota es la aplicación directa de la regla de
oro: en vez de rellenar huecos con suposiciones, los deja visibles y
accionables. La mantiene `graph-writer-agent` en cada escritura.

## Cobertura por ítem

Marcar el estado real de cada bloque de documentación. `parcial` significa
que hay notas pero con huecos declarados.

| Ítem | Estado | Qué falta / a quién preguntar |
| --- | --- | --- |
| Resumen ejecutivo | pendiente | |
| Proyecto (contexto, alcance, objetivos) | pendiente | |
| Actores y roles | pendiente | |
| Glosario | pendiente | |
| Cronología | pendiente | |
| Decisiones | pendiente | |
| Arquitectura | pendiente | |
| Ambientes y red | pendiente | |
| Seguridad y accesos | pendiente | |
| Proceso de negocio | pendiente | |
| Flujo de trabajo (pasos) | pendiente | |
| Historias de usuario | pendiente | |
| Reglas de negocio | pendiente | |
| Validaciones | pendiente | |
| Variables y parámetros | pendiente | |
| Archivos e insumos | pendiente | |
| Salidas y reportes | pendiente | |
| Correos y notificaciones | pendiente | |
| Excepciones | pendiente | |
| Riesgos | pendiente | |
| KPIs y SLAs | pendiente | |
| Pruebas UAT | pendiente | |
| Incidencias y stoppers | pendiente | |
| Beneficios | pendiente | |
| Referencias y anexos | pendiente | |
| Mejores prácticas | pendiente | |
| Preguntas frecuentes | pendiente | |

## Sin revisión humana

Notas que están en el grafo con `estado: propuesto`: se pueden consultar,
pero **nadie las ha validado todavía**. Entraron porque al escribir el lote
se decidió seguir adelante, no porque alguien las diera por buenas. Tratar
sus afirmaciones como provisionales hasta que un humano las revise en
Obsidian y les cambie el estado a `verificado`.

| Tipo | Cuántas | Notas |
| --- | --- | --- |
| | | |

## Enlaces sin destino

Wikilinks que apuntan a una nota que no existe. **No se borran**: cada uno
es un nodo que falta documentar, y quitarlo esconde el hueco.

- [ ] `[[Tipo - nombre]]` ← referenciado desde [[Nota - origen]]

## Decisiones de fusión pendientes

Pares de notas que describen el mismo concepto desde niveles de precedencia
distintos (ver `_ingestas/<lote>/_CANDIDATOS-FUSION.md`). Ambas están
escritas tal cual: fusionarlas destruiría la trazabilidad por nivel, que es
lo que hoy permite responder "¿esto lo dice el código o lo dice el manual?".
La decisión es humana.

- [ ] [[Nota A]] ↔ [[Nota B]] — relación: complementarios / en tensión / jerárquicos

## Preguntas abiertas

Preguntas concretas para un humano, con a quién dirigirlas. Aquí caen
también las preguntas que `qa-agent` no pudo responder con el vault — son
la mejor señal de qué falta documentar.

- [ ] ¿...? → preguntar a [[Actor - nombre-rol]]

## Inconsistencias sin resolver
- [[Inconsistencia - nombre]]

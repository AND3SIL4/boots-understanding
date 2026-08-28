---
name: rpa-best-practices
description: "Generic RPA industry knowledge — not specific to any one client's process. Use during ingestion to know what to look for even when source documents are silent, and as a checklist for common blind spots in legacy RPA documentation."
---

# Buenas prácticas y patrones comunes de RPA

Conocimiento de industria, independiente de cualquier proyecto concreto.

## Patrones de excepción comunes a buscar

- Timeout / elemento no encontrado en la interfaz.
- Dato faltante o en formato inesperado (fecha, moneda, NIT/RUT).
- Cambio de layout/UI en el sistema objetivo.
- Reintentos: ¿cuántos, con qué espera, y qué pasa si se agotan?
- Excepción de negocio vs. excepción técnica (el bot debe tratarlas
  distinto: una detiene el caso para revisión humana, la otra reintenta).

## Checklist de puntos ciegos típicos en documentación legada

Al ingerir un documento, si no menciona lo siguiente, vale la pena marcarlo
como pregunta abierta en vez de asumir que no aplica:

- Manejo de credenciales/secretos (¿dónde están, quién las rota?).
- Diferencias entre entornos (dev/qa/prod) y cómo se despliega un cambio.
- Dependencias de calendario/horario (¿el proceso corre en fechas fijas,
  días hábiles, cierre de mes?).
- Logging y monitoreo: ¿dónde se ven los errores cuando el bot falla?
- Volumen esperado y qué pasa si se excede (colas, particionamiento).

## Convenciones de nombres frecuentes

Los bots suelen nombrarse `Bot_<ProcesoCorto>` o `<Area>_<Proceso>_Bot`.
Si el documento usa un nombre distinto al del bot en producción, registrar
ambos en la nota `Robot` — es una fuente común de confusión al dar soporte.

Este conocimiento es de apoyo para `ingestion-agent`: ayuda a saber qué
preguntar, nunca a rellenar un vacío del documento con una suposición
genérica presentada como si fuera un hecho del proceso documentado.

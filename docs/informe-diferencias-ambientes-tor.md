# TOR — Cuenta Transitoria · Diferencias DEV vs PRD

**Fecha:** 2026-09-04 · **Exports comparados:** dev y prd `20260715_131108`
**Objetivo:** alinear desarrollo con producción.

---

## Resumen ejecutivo

Es el proyecto **mejor alineado de los tres**. La estructura de base de datos es idéntica (45 tablas, sin una sola columna de diferencia) y no falta ningún objeto en desarrollo. La brecha se reduce a **2 procedimientos** cuya versión de desarrollo quedó reducida, y a residuos de prueba en desarrollo.

Además, **desarrollo contiene cambios más recientes que producción** (marzo 2026 frente a enero 2025), por lo que aquí la sincronización no es unidireccional: hay trabajo pendiente de promover.

**Esfuerzo estimado: 19 horas-persona (~2,5 días hábiles).**

---

## 1. Automation Anywhere

| Concepto | DEV | PRD | Brecha |
|---|---:|---:|---|
| Archivos totales | 64 | 64 | **0** |
| Objetos ausentes / sobrantes | 0 | 0 | **0** |
| Bots con contenido distinto | — | — | 2 |

Ningún bot difiere en número de pasos. Las diferencias son de parámetros y marca de versión:

| Bot | Pasos | Diferencia |
|---|:---:|---|
| `HU02_RealizarProgramacionReporteSAP` | 349 = 349 | DEV marcado *11/03/2026*, PRD *28/01/2025*. PRD conserva el literal `20`. |
| `HU03_LlevarReporteCuentaTransitoriaBD` | 183 = 183 | DEV marcado *11/03/2026* con literal `20`; PRD *28/01/2025* con literales `5` y `9`. |

> **Dirección de la brecha:** desarrollo es la versión **más nueva**. Antes de sincronizar hay que decidir si esos cambios de marzo de 2026 deben promoverse a producción o descartarse.

## 2. Base de datos SQL Server

| Concepto | DEV | PRD | Brecha |
|---|---:|---:|---|
| Tablas | 45 | 45 | **0** |
| Tablas ausentes / sobrantes | 0 | 0 | **0** |
| Tablas con columnas desalineadas | — | — | **0** |
| Stored procedures | 15 | 15 | **0** |
| SP con divergencia real | — | — | **5** |

La estructura está **completamente alineada**. La única diferencia de esquema es el nombre de la base (`RPA_QA` en desarrollo, `RPA` en producción), que es lo esperado.

**Procedimientos almacenados** — medido por comparación de tokens, ignorando espacios y comentarios:

| SP | Tokens DEV → PRD | Similitud | Lectura |
|---|:---:|---:|---|
| `identifyPartialItems` | 232 → 1.679 | **17 %** | DEV es un fragmento del productivo |
| `identifyOCPartialItems` | 303 → 1.597 | **23 %** | DEV es un fragmento del productivo |
| `importDataInforme` | 978 → 805 | 87 % | DEV tiene lógica adicional |
| `identifyPartialAsignaciones` | 545 → 610 | 90 % | Falta lógica en DEV |
| `insertDataProductivo` | 600 → 582 | 97 % | Diferencia menor |
| Los 10 restantes | — | 98–100 % | Solo formato / indentación |

**Hallazgo — la lógica productiva vive en los SP de prueba:** en desarrollo, `identifyPartialItemsPrueba` (1.441 tokens) y `identifyOCPartialItemsPrueba` (1.201) son idénticos entre ambientes y coinciden en un **91 %** y **85 %** con los procedimientos *productivos* de producción. Es decir: en desarrollo se trabajó sobre las copias `*Prueba` y los procedimientos reales quedaron como versiones reducidas y obsoletas.

**Hallazgo — hardcode de pruebas en desarrollo:** `validateClosingDate` en desarrollo fija el mes con `SET @month = 'Junio'`, anulando el cálculo dinámico del mes anterior. Producción no tiene esa línea. Cualquier prueba de cierre en desarrollo se ejecuta siempre contra junio.

---

## 3. Estimación de esfuerzo

| Frente | Actividad | Horas |
|---|---|---:|
| BD | Reconstruir `identifyPartialItems` desde producción | 4.0 |
| BD | Reconstruir `identifyOCPartialItems` desde producción | 4.0 |
| BD | Alinear `importDataInforme` | 1.5 |
| BD | Alinear `identifyPartialAsignaciones` | 1.0 |
| BD | Alinear `insertDataProductivo` | 0.5 |
| BD | Normalizar los 10 SP que difieren solo en formato | 1.5 |
| BD | Retirar hardcode `@month = 'Junio'` y consolidar los SP `*Prueba` | 0.5 |
| AA | Conciliar `HU02` y `HU03` + decidir destino de los cambios de 03/2026 | 2.0 |
| QA | Regresión de cierre y compensación de parciales | 4.0 |
| **Total** | | **19.0 h** |

**≈ 2,5 días hábiles de un desarrollador.**

## 4. Riesgos

1. **Decisión previa obligatoria** — desarrollo tiene cambios de marzo de 2026 que producción no tiene. Sobrescribir desarrollo con producción **los perdería**. Debe resolverse antes de ejecutar la sincronización.
2. **`identifyPartialItems` / `identifyOCPartialItems`** — con 17 % y 23 % de similitud no son un ajuste sino una reconstrucción; conviene copiarlos íntegros desde producción en lugar de reconciliarlos línea a línea.
3. **Duplicidad `*Prueba`** — mantener dos versiones del mismo procedimiento es el origen de esta divergencia. Se recomienda consolidar en una sola y eliminar las copias de prueba.

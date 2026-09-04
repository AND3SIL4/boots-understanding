# ESCARLET — Cuadre de Cajeros · Diferencias DEV vs PRD

**Fecha:** 2026-09-04 · **Exports comparados:** dev `20260901_160531` / prd `20260901_160216`
**Objetivo:** alinear desarrollo con producción.

---

## Resumen ejecutivo

Es el proyecto con **mayor brecha de los tres**. Desarrollo va muy por detrás de producción: faltan 42 tablas, 5 procedimientos y hay 27 tablas con columnas desalineadas. Todo el módulo de **Bonos** y el esquema de **contingencia** son inexistentes en desarrollo.

Se detectó además un **hallazgo de seguridad en producción** que debe atenderse aparte de la sincronización (ver §4).

**Esfuerzo estimado: 46 horas-persona (~6 días hábiles).**

---

## 1. Automation Anywhere

| Concepto | DEV | PRD | Brecha |
|---|---:|---:|---|
| Archivos totales | 75 | 91 | **−16** |
| Bots con lógica distinta | — | — | **4** |
| Capturas (.png) ausentes en DEV | — | 16 | 16 |

**Bots con diferencia funcional:**

| Bot | Pasos DEV→PRD | Qué falta en desarrollo |
|---|:---:|---|
| `HU15B_GenerarCuadreCajeros` | 306 → 314 | **+8 pasos, +7 subrutinas.** Generación de hojas "Cierre Datafono" y "Detalle Datafono". Funcionalidad ausente en DEV. |
| `HU15B_MPInsumosProcesados` | 176 → 172 | Ver §4 — divergencia de credenciales. |
| `HU19_DescargueJobsPropios` | 245 → 247 | Ajuste 24/09/2024 en el apuntamiento de la ruta del Bulkeo. |
| `CambioLicencia` | 51 → 48 | PRD reemplazó comentarios por `messageBox`. Cosmético. |

Las 16 capturas ausentes pertenecen a `HU15B_CargaIndividualDrive`, coherente con que esa funcionalidad no esté replicada en desarrollo.
`HU00_DespliegeAmbiente` y `LoginSAPLogon_copy` difieren solo en metadatos: sin impacto funcional.

## 2. Base de datos SQL Server

| Concepto | DEV | PRD | Brecha |
|---|---:|---:|---|
| Tablas | 246 | 267 | **−21 netas** |
| Tablas ausentes en DEV | — | — | **42** |
| Tablas sobrantes en DEV | 21 | — | 21 |
| Tablas con columnas desalineadas | — | — | **27** |
| Stored procedures | 5 | 10 | **−5** |

**42 tablas ausentes en DEV**, en tres bloques:
- **18 tablas `*_Contingencia`** — el esquema de contingencia completo no existe en desarrollo.
- **12 tablas `Historico*`** — historificación de Bonos, Cuadre Integral y HU22/HU25.
- **12 tablas de Bonos y soporte** — `PlantillasBonos`, `TicketInsumoBonos`, `EstadosBonos`, `NCRBonos`, `PROSEGUR_BONOSFOSFEC`, `MatrizExtractos`, `CargueDrive`, entre otras.

**Patrón sistemático de columnas:** producción añadió `FechaRecoleccion` a **11 tablas** temporales (`TempVatco`, `TempProsegur`, `TempSodexo`, `TempBonosFisicos`, sus variantes 14/15 y `TempConsolidadoHU25`) e `Incocredito` a **5**. Desarrollo no recibió ninguno de estos cambios.

**21 tablas sobrantes en DEV** — incluyen tablas que **pertenecen al proyecto CONI** (`Control_Insumo_Fondos`, `DiasCaidaFondos`, `ControlHUFondos`, `ParametrosFondos`, `CorreosNotificacionFondos`) más residuos de prueba (`VATCOPRUEBASF`, `prosegurPRUEBASF`, `redebanPrueba`, `PruebaNCR`). La base de desarrollo tiene los dos proyectos mezclados.

**Stored procedures:**

| SP | Estado | Similitud real |
|---|---|---:|
| `SP_PasarHistoricoBonos` | **Divergente** (4.674 → 6.508 tokens) | 83 % |
| `SP_PasarHistoricoCuadreIntegral` | Solo formato | 100 % |
| `QueriesHU17` | Solo formato | 100 % |
| `QueriesHU18` | Solo formato | 100 % |
| `SP_BaseCuadreBW` | Solo formato | 100 % |
| `InsertVatcoToBonosFisicos` | **Ausente en DEV** | — |
| `InsertProsegurToBonosFisicos` | **Ausente en DEV** | — |
| `InsertConexoToBonosFisicos` | **Ausente en DEV** | — |
| `InsertIntoTicketInsumoBonosHU23` | **Ausente en DEV** | — |
| `UpdateBaseCuadreBWIntegral` | **Ausente en DEV** | — |

> Medido por comparación de tokens ignorando espacios y comentarios. Cuatro de los cinco procedimientos que difieren en texto son **funcionalmente idénticos**: cambian solo saltos de línea e indentación. El esfuerzo real se concentra en `SP_PasarHistoricoBonos`.

---

## 3. Estimación de esfuerzo

| Frente | Actividad | Horas |
|---|---|---:|
| BD | Crear 42 tablas (contingencia, histórico, bonos) | 11.0 |
| BD | Alinear columnas en 27 tablas (`FechaRecoleccion`, `Incocredito`, etc.) | 8.0 |
| BD | Depurar 21 tablas sobrantes / separar esquema CONI | 3.0 |
| BD | Desplegar 5 SP ausentes | 2.5 |
| BD | Alinear `SP_PasarHistoricoBonos` (única divergencia real) | 6.0 |
| BD | Verificar/normalizar los 4 SP que difieren solo en formato | 1.0 |
| AA | `HU15B_GenerarCuadreCajeros` (hojas Datafono) | 3.0 |
| AA | `HU15B_MPInsumosProcesados` + `HU19` + `CambioLicencia` | 3.5 |
| QA | Regresión (módulo Bonos y Cuadre Integral) | 8.0 |
| **Total** | | **46.0 h** |

**≈ 6 días hábiles de un desarrollador.**

## 4. Hallazgo de seguridad — requiere acción independiente

El bot `HU15B_MPInsumosProcesados` **en producción** tiene la cadena de conexión a base de datos embebida en texto plano (servidor, usuario y contraseña visibles en el código del bot) y **eliminó las 2 llamadas a `Read Credential`** que sí conserva la versión de desarrollo. Va acompañado del comentario *"Pruebas: Activar conexión DB a productivo"*, lo que sugiere que un cambio temporal de pruebas quedó promovido a producción.

**En este punto desarrollo es la versión correcta.** No debe copiarse producción sobre desarrollo; al contrario, producción debe volver al uso del gestor de credenciales.

- Impacto: credencial de servicio expuesta en el repositorio de bots.
- Acción: restaurar `Read Credential` en producción y rotar la contraseña.
- Esfuerzo: 2 h + rotación por parte de infraestructura.
- **No está incluido en las 56 h**: es una corrección sobre producción, no una sincronización.

## 5. Riesgos

1. **Volumen de la brecha** — 42 tablas y 5 procedimientos implican que desarrollo hoy no puede probar el módulo de Bonos ni el flujo de contingencia.
2. **Bases mezcladas** — la presencia de tablas de CONI en la base de ESCARLET desarrollo dificulta el diagnóstico; conviene separarlas antes de sincronizar.
3. **`SP_PasarHistoricoBonos`** — es el único procedimiento con divergencia real (83 % de similitud, ~1.890 tokens de diferencia) y el ítem de mayor incertidumbre de la estimación.

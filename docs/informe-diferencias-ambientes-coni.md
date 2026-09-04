# CONI — Conciliación de Fondos · Diferencias DEV vs PRD

**Fecha:** 2026-09-04 · **Exports comparados:** dev `20260902_091033` / prd `20260831_214630`
**Objetivo:** alinear desarrollo con producción.

---

## Resumen ejecutivo

Desarrollo está **desactualizado frente a producción**. La brecha se concentra en la base de datos (15 tablas ausentes, todas de historificación) y en 5 bots que en producción recibieron ajustes de soporte que nunca bajaron a desarrollo.

**Esfuerzo estimado: 23 horas-persona (~3 días hábiles).**

---

## 1. Automation Anywhere

| Concepto | DEV | PRD | Brecha |
|---|---:|---:|---|
| Archivos totales | 143 | 144 | −1 |
| Bots con lógica distinta | — | — | **5** |
| Bots solo en DEV | 3 | 0 | 3 |
| Capturas (.png) desalineadas | 2 | 6 | 8 |

**Bots con diferencia funcional:**

| Bot | Pasos DEV→PRD | Qué falta en desarrollo |
|---|:---:|---|
| `CargarInsumoRepositorio` | 119 → 144 | **+25 pasos.** Manejo de errores SharePoint (try/catch 3→7), autenticación vía Control Room, doble UploadFile/ListFiles. Marcado "Ajuste Monitoreo". |
| `HU20_CargarPlantillasMasivasSAP` | 186 → 195 | **+9 pasos.** Ajuste "Cambio monitoreo 31/10/2025": nuevo click SAP `wnd[2]/tbar[0]/btn[0]`. |
| `HU26_DescargarDocumentoSAP` | 129 → 131 | **+2 pasos.** Selectores SAP divergentes: PRD usa `btn[29]` y `B_TOTL`; DEV apunta a coordenadas antiguas. |
| `HU22_GeneracionReporteGestion` | 7 → 7 | Fix "Ajuste Soporte novedad de Ceros en Fecha". |
| `CambioLicencia` | 51 → 48 | PRD reemplazó 6 comentarios por 3 `messageBox`. Cosmético. |

**Divergencia arquitectónica:** desarrollo tiene 3 funciones SharePoint independientes (`AutenticarApiSharepointGeneral`, `CargarArchivoSharepointGeneral`, `CrearCarpetasSharePointGeneral`) que **no existen en producción**. Producción integró esa lógica dentro de `CargarInsumoRepositorio`. Requiere decisión: descartar el refactor de DEV o promoverlo a PRD.

## 2. Base de datos SQL Server

| Concepto | DEV | PRD | Brecha |
|---|---:|---:|---|
| Tablas | 31 | 41 | **−10 netas** |
| Tablas ausentes en DEV | — | — | **15** |
| Tablas sobrantes en DEV | 5 | — | 5 |
| Tablas con columnas faltantes | — | — | 2 |
| Stored procedures | 3 | 4 | −1 |

**15 tablas ausentes en DEV** — 10 son de historificación (`HistoricoBalanceGeneral`, `HistoricoConciliacionFondos`, `HistoricoControl_Insumo_Fondos`, `HistoricoBaseCuadreFondos`, `HistoricoMovMesVatco/Prosegur/Transportadoras`, `HistoricoProgramacion_P/_V`, `HistoricoProsegurCiudades`, `HistoricoTempoConvenios`) más `Conciliacion`, `ConciliacionFondosBW`, `ParametrosCambioEntreFondos` y `PruebaProsegur`.
→ **Consecuencia:** el proceso de archivado histórico no es ejecutable ni probable en desarrollo.

**5 tablas sobrantes en DEV** (residuos): `Copia_BalanceGeneral`, `ParametrosBancos`, `Programacion_Prosegur`, `Programacion_Vatco`, `TemProgramacion`.

**Columnas faltantes:** `ConciliacionFondos.ID` · `PlantillasMasivas.Plantilla`

**Stored procedures:**

| SP | Estado | Detalle |
|---|---|---|
| `SP_PasarHistoricoFondos` | **Ausente en DEV** | Depende de las 10 tablas Histórico. |
| `SP_UpdateVatcoAndProsegur` | 66 líneas distintas | PRD: DELETE ampliado + 5 UPDATE adicionales (104 → 159 líneas). |
| `SP_ConciliacionFondos` | 1 diferencia funcional | PRD calcula `Total = Ventas + NovedadesEntradas + **SinClasificacionE**`. DEV omite el tercer sumando → **cifra de cuadre incorrecta en desarrollo**. |
| `SelectEnvioCorreos` | Alineado | — |

---

## 3. Estimación de esfuerzo

| Frente | Actividad | Horas |
|---|---|---:|
| BD | Crear 15 tablas + retirar 5 residuales | 4.0 |
| BD | 2 ALTER de columnas | 0.5 |
| BD | Desplegar `SP_PasarHistoricoFondos` | 0.5 |
| BD | Alinear `SP_UpdateVatcoAndProsegur` | 1.5 |
| BD | Corregir `SP_ConciliacionFondos` (SinClasificacionE) | 0.5 |
| AA | `CargarInsumoRepositorio` (+25 pasos) | 3.0 |
| AA | `HU20` y `HU26` (ajustes SAP) | 4.0 |
| AA | `HU22` + `CambioLicencia` | 1.0 |
| AA | Decisión y cierre del refactor SharePoint | 3.0 |
| QA | Regresión sobre las HU tocadas | 4.0 |
| **Total** | | **23.0 h** |

**≈ 3 días hábiles de un desarrollador.**

## 4. Riesgos

1. **`SP_ConciliacionFondos`** — el total sin `SinClasificacionE` produce descuadres en desarrollo; corregir primero, es de bajo costo y alto impacto.
2. **Tablas Histórico** — sin ellas no se puede validar el archivado antes de pasar a producción.
3. **SharePoint** — las dos ramas divergieron; decidir dirección antes de tocar código.

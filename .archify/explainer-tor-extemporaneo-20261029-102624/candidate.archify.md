# ¿Cómo decide HU15 las fechas del reporte extemporáneo? {#tor-extemporaneo}

El extemporáneo es el mismo reporte de partidas abiertas de la cuenta transitoria. La diferencia: se pide a SAP con **fecha clave = último día del mes anterior**, y solo en una ventana de días posterior al cierre. El negocio define esa ventana en la columna J de la hoja `CALENDARIO CIERRE`. Los archivos llevan la fecha de hoy con el prefijo `EXTM`.

- Puerta de calendario: la fila del **mes anterior** en `CalendarioCierre` debe tener `InformeExtemporaneo` diligenciado; si no, HU15 se detiene.[](source:hu15)
- Puerta de ventana: corre solo si `DÍA(Fecha procesar BOT) < DÍA(hoy) <= DÍA(InformeExtemporaneo) + 1`. Compara números de día, no fechas completas.[](source:hu15)
- La fecha clave en SAP (`PA_STIDA`) es siempre `EOMONTH(getdate(),-1)`. El base usa la fecha de hoy y solo usa fin de mes el día de cierre.[](source:hu02)
- Corre después del base: el Main lanza HU15 con el mismo ticket cuando este llega a Estado 100.[](source:main)

@answer scope="Export AA de producción (HU15 modificado 26/09/2023), SP de PRD y logs del 03/07/2026 y 03/08/2026. Las notas del vault sobre HU15 están en estado propuesto."
@meta locale=es
@source hu15 title="HU15_DescargarInformeExtemporaneo (export AA PRD)" path="_inbox/tor-cuenta-transitoria/export-aa/produccion/Automation Anywhere/Bots/Cuenta Transitoria/HU/HU15_DescargarInformeExtemporaneo"
@source hu14 title="HU14_GenerarInformeBase (export AA PRD)" path="_inbox/tor-cuenta-transitoria/export-aa/produccion/Automation Anywhere/Bots/Cuenta Transitoria/HU/HU14_GenerarInformeBase"
@source hu02 title="HU02_RealizarProgramacionReporteSAP (export AA PRD)" path="_inbox/tor-cuenta-transitoria/export-aa/produccion/Automation Anywhere/Bots/6. OPERACIONES Y TESORERIA/CuentasPorPagar/CuentaTransitoria/HU/HU02_RealizarProgramacionReporteSAP"
@source hu01 title="HU01_CargueDeInsumosBD, paso CALENDARIO CIERRE" path="_inbox/tor-cuenta-transitoria/export-aa/produccion/Automation Anywhere/Bots/6. OPERACIONES Y TESORERIA/CuentasPorPagar/CuentaTransitoria/HU/HU01_CargueDeInsumosBD"
@source main title="MainCuentaTransitoria, paso Actualizar Estado / Extemporaneo" path="_inbox/tor-cuenta-transitoria/export-aa/produccion/Automation Anywhere/Bots/6. OPERACIONES Y TESORERIA/CuentasPorPagar/CuentaTransitoria/MainCuentaTransitoria"
@source spextm title="importDataInformeEXTM.sql (PRD)" path="_inbox/tor-cuenta-transitoria/database-sqlserver/produccion/stored-procedures/importDataInformeEXTM.sql"
@source ddl title="DDL PRD, tabla CalendarioCierre" path="_inbox/tor-cuenta-transitoria/database-sqlserver/produccion/ddl-tor-cuenta-transitoria-prd.sql"
@source log0703 title="Log producción 03/07/2026" path="_inbox/tor-cuenta-transitoria/logs-ejemplos/Log_WPROFABRICRPAC2_CGGEDRPA_20260703.txt"
@source log0803 title="Log producción 03/08/2026" path="_inbox/tor-cuenta-transitoria/logs-ejemplos/Log_WPROFABRICRPAC2_CGGEDRPA_20260803.log"
@source insumos title="Insumos Cuenta Transitoria.xlsx (copia en tmp/insumos)" path="_inbox/tor-cuenta-transitoria/tmp/insumos/Insumos Cuenta Transitoria.xlsx"

## ¿En qué se diferencia del reporte base? {#diferencias}

| Aspecto | Reporte base (HU02 + HU14) | Extemporáneo (HU15) |
| --- | --- | --- |
| Cuándo corre | Todos los días | Solo en la ventana posterior al cierre {caution:Condicional} |
| Quién lo dispara | Estados del ticket: HU02 (1-3), HU14 (19-23) | Paso `Extemporaneo` del Main, con el ticket ya en Estado 100 |
| Condición de calendario | Ninguna; `Fecha procesar BOT` solo marca si hoy es cierre | Fila del mes anterior con `InformeExtemporaneo` no vacío |
| Fecha clave en SAP (`PA_STIDA`) | Fecha de inserción del ticket; fin de mes anterior solo si `Cierre = TRUE` | Siempre `EOMONTH(getdate(),-1)` {good:Clave del EXTM} |
| `FechaReporte` (nombres de archivo) | Igual a la fecha clave | Fecha de hoy (fecha de inserción del ticket) |
| Archivo descargado de SAP | `Transitoria Consolidado Base F.xls` | `Transitoria Consolidado Base EXTM F.xls` |
| SP de carga | `importDataInforme` | `importDataInformeEXTM`: idéntico salvo el nombre del CSV |
| Fecha del job `ZRPA002` | Hoy + 5 min | Fecha forzada a hoy; hora = ahora + 5 min |
| Informe final | `Informe Base F (HHmm).xlsx` | `Informe Base Extemporaneo ddMMyyyyHH.xlsx` |
| Históricos e informe interno | Sí se actualizan | No; termina en Exportar informe base |
| Preguntas al usuario en cierre | Sí (`ForYesNo` en REVISION FECHA CIERRE) | Ninguna |

El SP de carga lee `FechaReporte` del ticket para armar la ruta del CSV, por eso el nombre del archivo y el ticket deben coincidir.[](source:spextm)

## ¿Cómo llegan las fechas desde el Excel de insumos? {#insumos}

HU01 trunca `CalendarioCierre` y la recarga desde el rango `B4:J16` de la hoja `CALENDARIO CIERRE`; la fila 4 es encabezado.[](source:hu01)

| Columna Excel | Columna en BD | Cómo la guarda HU01 | Quién la usa |
| --- | --- | --- | --- |
| B · MESES | `MESES` | Texto tal cual | Clave de búsqueda: nombre del mes en mayúsculas |
| C · Fecha límite para Radicación | `[Fecha Limite para Radicacion]` | `CONVERT(Date, x, 103)` | Ningún bot |
| D, E, F · Fecha para procesar la información | `[... informacion 1..3]` | `CONVERT(Date, x, 103)` | Ningún bot |
| G · Fecha Procesar bot | `[Fecha procesar BOT]` | `CONVERT(Date, x, 103)` | HU01 y HU14 (¿hoy es cierre?); HU15 (día inicio) {good:Inicio} |
| H, I · Fecha para liquidar impuestos | `[... Impuestos 1..2]` | `CONVERT(Date, x, 103)` | Ningún bot |
| J · sin encabezado | `InformeExtemporaneo nvarchar(10)` | Texto, **sin** conversión | HU15 (día fin) {caution:Sin encabezado} |

@callout level=warning title="La columna J no tiene título en el Excel"
`InformeExtemporaneo` es la única columna guardada como texto (`nvarchar(10)`), no como `date`.[](source:ddl)
En la copia del Excel revisada, la celda J tiene formato fecha corta, por ejemplo JUNIO = 09/07/2026.[](source:insumos)
Una celda vacía queda como cadena vacía y HU15 la trata como "no hay extemporáneo".

La búsqueda usa el **mes anterior**: HU15 consulta `General.Tiempo` con `dateadd(month,-1,hoy)`. En julio lee la fila JUNIO, en agosto la fila JULIO.[](source:hu15)

## ¿Qué secuencia sigue el bot? {#secuencia}

@sequence id=extm-sequence title="HU15: del disparo al informe extemporáneo"
| actor main | Main TOR | backend |
| actor hu15 | HU15 Extemporáneo | backend |
| actor sql | SQL Server RPA | database |
| actor sap | SAP Corporativo | external |
| actor red | Ruta de red y FTP | cloud |
| phase | Disparo |
| main | hu15 | ticket en Estado 100: ejecutar HU15 | emphasis |
| phase | Puerta de calendario |
| hu15 | sql | General.Tiempo: mes de hoy menos 1 mes |
| sql | hu15 | MesNombre (MesActual) | return |
| hu15 | sql | CalendarioCierre del MesActual con InformeExtemporaneo |
| sql | hu15 | InformeExtemporaneo y Fecha procesar BOT | return |
| phase | Reporte SAP con fecha clave |
| hu15 | sql | FechaReporte = EOMONTH(hoy, -1) |
| hu15 | sap | job del reporte con PA_STIDA = fin de mes anterior | emphasis |
| hu15 | sql | FechaReporte = fecha de inserción (Cierre FALSE) |
| sap | hu15 | SMX: job terminado | return |
| sap | red | Transitoria Consolidado Base EXTM (hoy).xls |
| phase | Carga e informe |
| hu15 | sql | TRUNCATE Informe + importDataInformeEXTM |
| hu15 | sap | ZRPA002 programado con fecha de hoy |
| sap | red | transitorio_yyyyMMdd_cggedrpa.csv por FTP |
| hu15 | sql | importDataInformeBase, ME2L y cruces |
| hu15 | red | Informe Base Extemporaneo ddMMyyyyHH.xlsx | emphasis |
| hu15 | main | out_Estado = 100 | return |

@steps id=extm-sequence-steps
| s1 | Main TOR | Cuando un HU devuelve Estado 100, el paso `Extemporaneo` ejecuta HU15 con el mismo `IdTransaccion`. HU15 también existe como `HUProceso = 15`. | extm-sequence | main |
| s2 | SQL Server RPA | Dos consultas deciden si HU15 sigue: el mes anterior y su fila del calendario. Si la tabla vuelve vacía, registra `Tabla vacía. Se detiene HU15`. | extm-sequence | sql |
| s3 | SAP Corporativo | El job del reporte usa la fecha clave de fin de mes. Luego `ZRPA002` genera el CSV que vuelve por FTP. | extm-sequence | sap |
| s4 | Ruta de red y FTP | Los archivos quedan en `DESCARGA\INFORMES` con la fecha de hoy y el prefijo `EXTM`. | extm-sequence | red |

## ¿Cómo decide si corre y con qué fecha? {#decision}

@workflow id=extm-decision title="Decisión de fechas en HU15"
| lane calendario | Puerta de calendario |
| lane ventana | Puerta de ventana |
| lane fecha | Fechas del reporte |
| inicio | Ticket en Estado 100 | external | calendario |
| mes | Mes evaluado = mes anterior | backend | calendario |
| fila | ¿Columna J diligenciada? | backend | calendario |
| fin1 | Fin: tabla vacía | external | calendario |
| dias | Día inicio y día fin | backend | ventana |
| ventana | ¿Inicio < hoy <= fin+1? | backend | ventana |
| fin2 | Fin: no cumple fechas | external | ventana |
| clave | Fecha clave = fin de mes | backend | fecha |
| nombre | Archivos con fecha de hoy | backend | fecha |
| genera | Generar y cargar EXTM | backend | fecha |
| inicio > mes | |
| mes > fila | |
| fila > fin1 | no |
| fila > dias | sí | emphasis |
| dias > ventana | |
| ventana > fin2 | no |
| ventana > clave | sí | emphasis |
| clave > nombre | |
| nombre > genera | | emphasis |

@steps id=extm-decision-steps
| d1 | Mes evaluado = mes anterior | `MesActual = UPPER(MesNombre)` de `General.Tiempo` para la fecha `hoy - 1 mes`. | extm-decision | mes |
| d2 | ¿Columna J diligenciada? | Filtro `InformeExtemporaneo != '' AND IS NOT NULL AND MESES = MesActual`. | extm-decision | fila |
| d3 | Día inicio y día fin | Día fin: texto después del primer `/` de `InformeExtemporaneo`, convertido a número. Día inicio: texto antes del segundo `-` de `Fecha procesar BOT`, convertido a número. | extm-decision | dias |
| d4 | ¿Inicio < hoy <= fin+1? | `System:Day > DíaInicio AND System:Day <= DíaFin + 1`. El día de cierre no entra: la comparación es estricta. | extm-decision | ventana |
| d5 | Fecha clave = fin de mes | El paso `Fecha mes anterior` no tiene condición: siempre escribe `EOMONTH(getdate(),-1)` y lo envía a `PA_STIDA`. | extm-decision | clave |
| d6 | Archivos con fecha de hoy | Con `Cierre = FALSE`, `FechaReporte` vuelve a la fecha de inserción. `FechaProgramacion` se fuerza a `FechaExtemporanea` (hoy en `ddMMYYYY`). | extm-decision | nombre |

## ¿Qué muestran las ejecuciones reales? {#evidencia}

| Dato | 03/07/2026 | 03/08/2026 |
| --- | --- | --- |
| Ticket | 2274 | 2306 |
| Ticket al entrar a HU15 | `Cierre: FALSE`, `RevisionUsuario: INFORME` | `Cierre: FALSE`, `RevisionUsuario: INFORME` |
| Fecha clave registrada | `Fecha reporte 30062026` | `Fecha reporte 31072026` |
| Archivo descargado | `... EXTM 03072026.xls` | `... EXTM 03082026.xls` |
| Base del mismo día | `Transitoria Consolidado 03072026.xls` | `Transitoria Consolidado 03082026.xls` |

Ambos logs confirman la regla: fecha clave de fin de mes y archivo con la fecha de hoy.[](source:log0703)[](source:log0803)

Con la fila JUNIO de la copia del Excel, la cuenta del 03/07 cierra. Inicio = día 1 (01/07/2026), fin = día 9 (09/07/2026). Hoy = 3, y 1 < 3 <= 10, así que corre.[](source:insumos)

| Día de julio | ¿Corre HU15? | Motivo |
| --- | --- | --- |
| 1 (cierre) | No {bad:No} | 1 > 1 es falso; ese día el base ya usa fin de mes |
| 2 a 10 | Sí {good:Sí} | Dentro de la ventana; el 10 entra por el `+ 1` |
| 11 en adelante | No {bad:No} | 11 > 10 |

@callout level=note title="La copia del Excel no es la que se cargó en agosto"
En la copia revisada, la fila JULIO no tiene `Fecha Procesar bot` ni columna J, pero HU15 corrió el 03/08/2026. El Excel cargado ese día era otro; los valores reales de agosto no están en estas fuentes.

## ¿Qué riesgos hay que cuidar al migrar? {#riesgos}

@callout level=warning title="Solo se compara el número de día"
La ventana no mira mes ni año. Si `Fecha procesar BOT` o la columna J caen en otro mes que hoy, el resultado es erróneo. Ejemplo: en DICIEMBRE la columna J queda en enero del año siguiente.

@callout level=open-question title="Formato del texto que se parte"
El día se saca partiendo texto por `/` y `-`. El resultado depende del formato en que Automation Anywhere entrega cada fecha, y eso no consta en el export. Los logs solo prueban que la comparación pasó en julio y agosto.

@callout level=open-question title="Condición que el log contradice"
Tras enviar la fecha clave, la rama que restaura la fecha de inserción exige `Cierre = 'FALSE' AND FechaClave = ''`. Leída literalmente no debería ejecutarse, porque `FechaClave` ya tiene valor. Los dos logs muestran que sí se ejecutó: los archivos salieron con la fecha de hoy. Verifica en DEV antes de copiar la condición.

@callout level=warning title="HU15 sobrescribe las tablas del día"
HU15 hace `TRUNCATE` de `Informe` y de `InformeBase` y las recarga con datos de fin de mes. Ejecuta el extemporáneo solo después de que el base termine y exporte; así lo ordena hoy el Main.

El `+ 1` del día fin coincide con la nota del Excel: "El Bot inicia su ejecución a las 10:00 pm". Esta relación es una hipótesis; el código no la explica.

## ¿Cómo aplicar esta lógica al reporte base migrado? {#aplicar}

Esta propuesta en T-SQL replica la regla actual. Asume que la columna J está en formato `dd/MM/yyyy`; confírmalo antes de usarla.

```sql title="Réplica de la regla de fechas de HU15 (propuesta)"
DECLARE @hoy date = CAST(GETDATE() AS date);
DECLARE @mes nvarchar(11), @inicio date, @fin date;

-- 1. Mes evaluado: el mes anterior (mismo origen que el bot)
SELECT @mes = UPPER(MesNombre)
FROM General.Tiempo WHERE fecha = DATEADD(month, -1, @hoy);

-- 2. Puerta de calendario
SELECT @inicio = [Fecha procesar BOT],
       @fin    = TRY_CONVERT(date, InformeExtemporaneo, 103)
FROM CuentaTransitoria.CalendarioCierre
WHERE MESES = @mes AND ISNULL(InformeExtemporaneo, '') <> '';

-- 3. Puerta de ventana (réplica fiel: solo número de día)
IF @inicio IS NOT NULL
   AND DAY(@hoy) > DAY(@inicio)
   AND DAY(@hoy) <= DAY(@fin) + 1
BEGIN
    -- 4. Fecha clave para SAP (PA_STIDA)
    SELECT FechaClave = EOMONTH(@hoy, -1),
           FechaArchivos = FORMAT(@hoy, 'ddMMyyyy');  -- prefijo EXTM
END
```

- [ ] Calcula el mes evaluado como el mes anterior a hoy, no el mes actual.
- [ ] Si la columna J del mes evaluado está vacía, no generes el extemporáneo.
- [ ] Decide con negocio si conservas la comparación por número de día o comparas fechas completas (`@inicio < @hoy <= @fin + 1 día`).
- [ ] Envía a SAP la fecha clave `EOMONTH(hoy, -1)` y nombra los archivos con la fecha de hoy y el prefijo `EXTM`.
- [ ] Carga con `importDataInformeEXTM`; su ruta usa `FechaReporte` del ticket.
- [ ] Ejecuta el extemporáneo con el mismo ticket, después de que el base llegue a Estado 100.
- [ ] No actualices históricos ni informe interno desde el extemporáneo.

# Plan de empalme PRD → DEV — CONI (Conciliación de Fondos)

Fecha del análisis: 2026-09-02
Fuentes: `_inbox/coni-conciliacion-fondos/` (exports AA de dev y prd, DDL de ambos ambientes, SPs de ambos ambientes, 3 logs de prd).
No se abrieron: los `.zip` de backup, `tmp/grabaciones/`, `tmp/insumos/`, `transcripciones/`, `documentacion-general/`, la carpeta `dml-conciliacion-fondos/` (vacía) y las capturas `*Metadata/*.png`.

---

## 0. Resumen ejecutivo

1. **Los bots casi no difieren, pero divergieron en las dos direcciones.** De los 37 bots de dev y 34 de prd, **33 son idénticos byte a byte**. Hay 5 con diferencias reales y 3 que solo existen en dev. No es un caso de "dev está atrasado": hay cambios vivos en prd que dev no tiene, y un refactor en dev que prd no tiene. Un import ciego en cualquier dirección pierde trabajo.
2. **El ambiente lo decide el Control Room, no el bot.** `HU00_DespliegeAmbiente` arma `Schema = $@Database$.ConciliacionFondos` con el *Global Value* `$@Database$` y carga toda la tabla `ParametrosFondos` en el diccionario `Config`. No hay ningún `if` de ambiente en el código: el mismo export corre en `RPA` o en `RPA_QA` sin tocar nada. **El trabajo real del empalme es base de datos + Control Room.**
3. **Bloqueante nº 1: a la BD de dev le faltan 2 tablas que los bots sí usan.** `ConciliacionFondosBW` (la usan 9 bots + `SP_PasarHistoricoFondos`) y `Conciliacion` (la usa `GenerarResultadosProcesoEnExcel`) no existen en el DDL de dev. Sin ellas el proceso no corre de punta a punta en dev.
4. **Bloqueante nº 2: falta `SP_PasarHistoricoFondos` y sus 10 tablas `Historico*` en dev.** `Main_ConciliacionFondos` lo invoca en ambos ambientes. En el material entregado de dev ese SP no aparece — hay que confirmar contra el servidor si existe y no se exportó, o si de verdad no está.
5. **Bloqueante nº 3: dependencia de un esquema externo (`SchemaFase1`).** 12 bots consultan `$Config{SchemaFase1}$.MatrizPuntos` y `.MediosPago`. Ese esquema es el de Fase I / Cuadre de Cajeros (Escarlet) y **no está en ninguno de los dos DDL entregados**. El empalme de CONI en dev depende de que Fase I también exista en dev.
6. **Divergencia de arquitectura en SharePoint.** Prd migró `CargarInsumoRepositorio` a autenticación por Control Room (conexión OAuth `APISharepoint`); dev sigue con el camino viejo (credenciales `ClientId`/`ClientSecret` del package) más 3 bots auxiliares propios que llaman la API REST de Microsoft a mano. Son dos implementaciones distintas del mismo requerimiento y hay que elegir una, no fusionarlas.
7. **Lógica de negocio distinta en 2 stored procedures.** `SP_UpdateVatcoAndProsegur` de prd tiene ~12 reglas de borrado y 5 normalizaciones de decimales que dev no tiene; `SP_ConciliacionFondos` de prd suma `SinClasificacionE` al total de Entradas y dev no. **Hoy los dos ambientes calculan distinto.**

---

## 1. Inventario analizado

| Fuente | Qué es | Estado |
| --- | --- | --- |
| `export-aa/produccion/` (backup `coni.20260831_214630.bak-prd.zip`) | 34 bots + manifest con dependencias | Analizado completo |
| `export-aa/desarrollo/` (backup `coni.20260902_091033.bak-dev.zip`) | 37 bots + manifest | Analizado completo, diff semántico contra prd |
| `database-sqlserver/produccion/ddl-structure-conciliacion-fondos-coni.sql` | DDL de `RPA.ConciliacionFondos`: 41 tablas, 13 PK, 0 FK, 0 índices, 0 vistas | Analizado |
| `database-sqlserver/desarollo/coni-conciliancion-fondos-dev.sql` | DDL de `RPA_QA.ConciliacionFondos`: 31 tablas, 0 PK, 0 FK, 0 índices, 0 vistas | Analizado |
| `database-sqlserver/produccion/procedimientos-almacenados/` | 4 objetos: `SP_ConciliacionFondos`, `SP_PasarHistoricoFondos`, `SP_UpdateVatcoAndProsegur`, `SelectEnvioCorreos` | Analizados y diffeados |
| `database-sqlserver/desarollo/procedimientos-almacenados/` | 3 objetos (falta `SP_PasarHistoricoFondos`) | Analizados y diffeados |
| `logs-ejemplos/` | 3 logs de prd (29, 30 y 31/08/2026), máquina `WPROFABRIC4RPAC2` | Analizados |
| `database-sqlserver/produccion/modelo-relacional-coni.png` | Modelo relacional | **No abierto** |
| `documentacion-general/`, `transcripciones/`, `tmp/`, `*.zip` | Especificación funcional V2.2, manual de usuario, grabaciones, insumos | **No abiertos** (fuera del alcance de este diff técnico) |

> **Regla de oro aplicada:** todo lo que sigue sale de los archivos listados arriba. Donde no hay dato, dice "no está en el material entregado" — no se completó con suposiciones.

---

## 2. Cómo se selecciona el ambiente hoy (mecanismo verificado en el export)

`HU00_DespliegeAmbiente` es **idéntico en dev y prd**. Su lógica:

```
HU00_DespliegeAmbiente (lo llaman todos los bots al inicio)
  NameBot  = "ConciliacionFondos"           ← constante
  Schema   = $@Database$ + "." + NameBot    ← Global Value del Control Room
  Server   = $@Server$                      ← Global Value del Control Room
  Database = $@Database$
  connect(sDB1, Server, Database, locker Globales/DBColsubsidio)
  Config{*} = SELECT * FROM <Schema>.[ParametrosFondos]   ← todo lo demás sale de aquí
  → limpieza de logs (DiasMantenerLog) y de carpetas (DiasBorrar + DiasAdicionales)
```

**Consecuencia clave y favorable:** a diferencia de Escarlet, aquí **no hay ningún `if DataBase == "RPA"`**. El nombre del esquema siempre es `ConciliacionFondos` y la base la pone el Global Value. Si el Global Value `$@Database$` de dev vale `RPA_QA`, el bot resuelve `RPA_QA.ConciliacionFondos` solo. No hay que tocar código para cambiar de ambiente.

Las 35 claves de `ParametrosFondos` que leen los bots son **exactamente las mismas en dev y en prd**:

```
ActivarLog, ClientSAP, CodigoRobot, ConnectionName, CorreoFallas, DiasAdicionales,
DiasBorrar, DiasMantenerLog, EmailTableName, FolderBWHU26, FolderFiltrosHU26,
LanguajeSAP, NameBot, NomFuncionCaracter, NomInsumoControl, NombreLicencia,
NumberAttemps, PathConsolidado, PathLog, PathPlantillas, PathProgramaciones,
PathProsegur, PathProveedores, PathVatco, Retries, RutaRed, RutaReporteErrores,
Schema, SchemaFase1, Scheme, SharePointSiteName, SharePointSitePath, SiteSubDom,
TablaConfig, UsuarioSAP
```

La estructura de `ParametrosFondos` (`Nombre`, `Valor`, `Descripcion`, `ValorPredeterminado`, `Modificable`) es idéntica en los dos DDL. **Lo que cambia entre ambientes son los valores de esas 35 filas, no el esquema.**

### 2.1 `SchemaFase1` — dependencia externa no cubierta por el material

`SchemaFase1` apunta a un esquema **de otro proyecto** (Fase I — Cuadre de Cajeros). Los bots lo usan así:

| Bot | Tabla externa que consulta |
| --- | --- |
| `HU04_ValidarCertificacionVatco` | `MatrizPuntos` |
| `HU05_ValidarCertificaciónProsegur` | `MatrizPuntos` |
| `HU10_ConfrontaciónSaldosVentas` | `MediosPago` |
| `HU11_ConfrontaciónSaldosBaseCambio` | `MediosPago` |
| `HU12_ConfrontarSaldosProveedores` | `MediosPago` |
| `HU13_CompensaciónNovedades` | `MediosPago` |
| `HU14_ControlPlantillasEsperado` | (esquema completo) |
| `HU15_GeneraciónPlantillasMasivasCertificación` | `MatrizPuntos` |
| `HU16_GeneraciónPlantillasMasivasProgramacion` | `MatrizPuntos` |
| `HU17_GeneraciónPlantillasMasivasProveedores` | `MediosPago` |
| `HU19_GeneraciónPlantillasMasivasConciliacionFondos` | `MediosPago` |
| `HU26_DescargarDocumentoSAP` | `MediosPago` |

Ni `MatrizPuntos` ni `MediosPago` están en el DDL de prd ni en el de dev entregados — viven en el esquema de Fase I. Los logs de prd confirman la relación (`\\fserver\RPA_CONCILIACION_INTEGRAL_MP\FASE I CUADRE DE CAJEROS\...`).

**Implicación para el empalme:** dev necesita el esquema de Fase I poblado, al menos `MatrizPuntos` y `MediosPago`. Esto **enlaza este plan con el de Escarlet** (`docs/plan-migracion-escarlet-prd-a-dev.md`): si Fase I no está en dev, 12 de los 26 bots de CONI no se pueden probar.

---

## 3. Diferencias DEV vs PRD en los bots

Diff semántico (propiedades, paquetes, variables, nodos y atributos), ignorando `uid` y renumeración.

**33 de 34 bots comunes son idénticos.** El detalle de los que no:

| Bot | Quién está más nuevo | Qué cambia | Acción para el empalme |
| --- | --- | --- | --- |
| `HU20_CargarPlantillasMasivasSAP` | **PRD** | Cambio fechado **31/10/2025 ("Cambio monitoreo")**: valida con `windowExists` si la plantilla existe antes de cargarla; si no, escribe `WARNING - Plantilla no existe en: $pRutaArchivo$`, hace clic para cerrar el diálogo y `continue` del loop. Además: variable nueva `pVentanaInformacion`, un `Delay 3s` antes del clic "Permitir", y un `Step "Pruebas unitarias"` **deshabilitado**. | **Tomar PRD.** Está activo y funcionando: los 3 logs muestran ese WARNING disparándose decenas de veces al día. |
| `HU22_GeneracionReporteGestion` | **PRD** | Cambio "Ajuste Soporte novedad de Ceros en Fecha": agrega `DELETE FROM <Schema>.ProsegurCiudades WHERE Estado='99' AND Fecha='0'` antes de generar la hoja `ProsegurCiudades`. | **Tomar PRD.** Es un fix de datos; sin él dev genera el reporte con filas basura. |
| `HU26_DescargarDocumentoSAP` | **PRD** | (a) `wait` del botón Ejecutar de SAP sube de **15 a 100**; (b) tiene un `Step "Pruebas unitarias"` que llama a `HU00_DespliegeAmbiente` y **está habilitado**. | Tomar PRD para el `wait` (es un ajuste de estabilidad de SAP). **Deshabilitar el step "Pruebas unitarias"** — es código de prueba corriendo en producción. |
| `CambioLicencia` | **DEV** | Dev renombró todas las variables a convención `p*`/`aux*` (`Database`→`pDatabase`, `OutWebServices`→`pOutWebServices`, `Id_Bot`→`pId_Bot`, etc.), quitó 3 `MessageBox` deshabilitados de depuración —uno de ellos con la URL del CR y las variables de usuario/clave del cambio de licencia—, agregó el bloque de comentarios de cabecera ("Ultima modificación: 25/11/2024") y subió `Comment` de 2.13.2 a 2.16.0. Funcionalmente equivalente. | **Tomar DEV.** Ojo: el refactor **no corrige el bug**. Los logs del 30 y 31/08 muestran `Column Name : token does not exists` → `No se pudo realizar el cambio de licencia` en prd. Es el mismo fallo reportado en Escarlet. |
| `CargarInsumoRepositorio` | **Conflicto real (ver 3.1)** | Prd migró a `SharePoint.Authentication` con `authType=AuthenticateViaControlRoom` y conexión OAuth `APISharepoint`, sesión `SessionAPI`, `siteType='Other'` + `siteName`, y envolvió `UploadFile`, `ListFiles` y `CreateFolder` en `try/catch` con log de error. Dejó deshabilitado todo el camino viejo. Dev conserva el camino viejo activo (`AuthenticateViaPackage` con `ClientId`/`ClientSecret`, sesión `Colsubsidio`, `siteType='Default'`) más las llamadas a sus 3 bots auxiliares. | **Decisión de arquitectura, no de merge.** Ver 3.1. |

### 3.1 Los 3 bots que solo existen en DEV

| Bot | Qué hace | Paquetes exclusivos |
| --- | --- | --- |
| `GlobalFunctions/SharePoint/AutenticarApiSharepointGeneral` | `POST https://login.microsoftonline.com/$In_TenantID$/tokens/OAuth/2` → devuelve `Out_AutToken` | `Rest 3.11.1`, `String 5.4.2`, `System 3.9.2`, `TaskBot 2.4.0` |
| `GlobalFunctions/SharePoint/CargarArchivoSharepointGeneral` | Sube un archivo usando el token anterior | **`Python 2.9.0`** (único bot del proyecto que usa Python), `Dictionary 3.8.0`, `If 3.4.0`, `Step 2.3.0` |
| `GlobalFunctions/SharePoint/CrearCarpetasSharePointGeneral` | `POST https://$@TenantName$.sharepoint.com/sites/$In_SiteName$/_api/web/folders` | `Rest 3.11.1`, `If 3.4.0`, `Step 2.3.0` |

Los tres son un subsistema cerrado: **solo los llama `CargarInsumoRepositorio`** y solo se llaman entre ellos. Usan la credencial `Globales / TenantIDColsubsidio / TenantID`, que prd no usa en ningún bot.

**Estado en producción:** el archivo `CargarInsumoRepositorio` de prd todavía contiene las llamadas a `CargarArchivoSharepointGeneral` y `CrearCarpetasSharePointGeneral`, pero **dentro de bloques `try` deshabilitados** — son código muerto. Por eso el escáner de dependencias de AA no los incluyó en el manifest de prd y por eso no están en el export. Confirmado en `manifest.json`:

- dev → `scannedDependencies` de `CargarInsumoRepositorio` = `[AutenticarApiSharepointGeneral, WriteLog, CrearCarpetasSharePointGeneral, CargarArchivoSharepointGeneral]`
- prd → `scannedDependencies` = `[WriteLog]`

**Riesgo latente:** si alguien reactiva esas ramas en prd sin importar los 3 bots, prd rompe.

**Decisión a tomar (es del equipo, no del análisis):**

- **Opción A (recomendada): adoptar el camino de PRD.** Alinear dev con `AuthenticateViaControlRoom` + conexión OAuth `APISharepoint`. Ventaja: es lo que está corriendo hoy y quita 3 bots, 1 credencial y 7 versiones de paquete del inventario. Requiere crear la conexión OAuth `APISharepoint` en el Control Room de dev.
- **Opción B: adoptar el camino de DEV.** Importar los 3 bots a prd y reactivar las ramas. Ventaja: no depende de una conexión OAuth administrada por el CR. Costo: sube 7 versiones de paquete distintas a prd, incluida una dependencia de **Python** que hoy prd no tiene instalada en ningún runner del proceso.

Sea cual sea, hay que **borrar el camino perdedor**, no dejarlo deshabilitado. Hoy `CargarInsumoRepositorio` tiene tres implementaciones superpuestas y es, con diferencia, el bot más ruidoso de producción: **467 errores** `El insumo a cargar NO existe en la ruta` en 3 días de log.

### 3.2 Versiones de paquetes

9 paquetes tienen versiones distintas entre ambientes, pero **el 100% de esa diferencia viene de los 3 bots solo-dev más el `Comment` de `CambioLicencia`**. Los 33 bots idénticos usan exactamente las mismas versiones. Si se toma la Opción A, la diferencia de paquetes desaparece sola.

| Paquete | Versión exclusiva de DEV | Dónde |
| --- | --- | --- |
| `Python` | `2.9.0-20211016-065947` | `CargarArchivoSharepointGeneral` |
| `Rest` | `3.11.1-20220714-082239` | `AutenticarApiSharepointGeneral`, `CrearCarpetasSharePointGeneral` |
| `Dictionary` | `3.8.0-20220505-214608` | `CargarArchivoSharepointGeneral` |
| `If` | `3.4.0-20220127-062702` | `CargarArchivoSharepointGeneral`, `CrearCarpetasSharePointGeneral` |
| `Step` | `2.3.0-20220715-220724` | `CargarArchivoSharepointGeneral`, `CrearCarpetasSharePointGeneral` |
| `String` | `5.4.2-20220412-134724` | los 3 |
| `System` | `3.9.2-20220421-100102` | los 3 |
| `TaskBot` | `2.4.0-20220628-195240` | los 3 |
| `Comment` | `2.16.0` en dev vs `2.13.2` en prd | `CambioLicencia` |

---

## 4. Base de datos

### 4.1 Qué hay

| | PRD (`RPA.ConciliacionFondos`) | DEV (`RPA_QA.ConciliacionFondos`) |
| --- | --- | --- |
| Tablas | 41 | 31 |
| Primary keys | 13 | **0** |
| Columnas IDENTITY | 25 | 13 |
| Foreign keys | 0 | 0 |
| Índices | 0 | 0 |
| Vistas | 0 | 0 |
| Stored procedures | 4 | 3 |

### 4.2 Tablas que faltan en DEV (15)

| Tabla | ¿La usa el código? | Capa |
| --- | --- | --- |
| **`ConciliacionFondosBW`** | **Sí — 9 bots** (HU10–HU14, HU17, HU18, HU19, HU26) + `SP_PasarHistoricoFondos` | **Bloqueante** |
| **`Conciliacion`** | **Sí — `GenerarResultadosProcesoEnExcel`** | **Bloqueante** |
| `HistoricoBalanceGeneral` | Sí, vía `SP_PasarHistoricoFondos` | Histórico |
| `HistoricoBaseCuadreFondos` | idem | Histórico |
| `HistoricoConciliacionFondos` | idem | Histórico |
| `HistoricoControl_Insumo_Fondos` | idem | Histórico |
| `HistoricoMovMesProsegur` | idem | Histórico |
| `HistoricoMovMesTransportadoras` | idem | Histórico |
| `HistoricoMovMesVatco` | idem | Histórico |
| `HistoricoProgramacion_P` | idem | Histórico |
| `HistoricoProgramacion_V` | idem | Histórico |
| `HistoricoProsegurCiudades` | idem | Histórico |
| `HistoricoTempoConvenios` | idem | Histórico |
| `ParametrosCambioEntreFondos` | Sí, `SP_PasarHistoricoFondos` | Configuración |
| `PruebaProsegur` | **No la referencia ningún bot ni SP** | Descartable |

`Main_ConciliacionFondos` invoca `SP_PasarHistoricoFondos` **en ambos ambientes**. Como ni el SP ni sus 10 tablas `Historico*` están en el material de dev, hoy esa llamada debería fallar en dev. **A confirmar contra el servidor** (ver Anexo A) — puede ser que exista y no se haya exportado.

### 4.3 Tablas que sobran en DEV (5)

Ninguna de las cinco la referencia bot ni SP alguno. Son residuo de trabajo manual:

| Tabla | Observación |
| --- | --- |
| `Copia_BalanceGeneral` | Copia manual de `BalanceGeneral` (26 de sus 38 columnas) |
| `ParametrosBancos` | 2 columnas, sin uso |
| `Programacion_Prosegur` | Duplicado de `Programacion_P` (18 cols) |
| `Programacion_Vatco` | Duplicado de `Programacion_V` (18 cols) |
| `TemProgramacion` | El código usa `#TemProgramacion` (tabla **temporal**, creada y destruida en `HU06`). Esta versión persistida es un residuo de haber corrido el DDL sin el `#`. |

**Acción:** no borrarlas en el mismo paso del empalme. Renombrarlas con prefijo `ZZ_` o moverlas a un esquema `_papelera`, correr el proceso completo, y borrarlas cuando se confirme que nada las toca.

### 4.4 Diferencias en las 26 tablas comunes

| Tabla | Diferencia | Acción |
| --- | --- | --- |
| `ConciliacionFondos` | Dev **no tiene** la columna `ID int IDENTITY(1,1) NOT NULL` ni la PK `PK__Concilia__3214EC277DC6493A`. Y `SaldoSAP` es `bigint` en prd vs **`varchar(30)`** en dev. | **Crítico.** Agregar `ID` + PK y alinear el tipo de `SaldoSAP`. Un `varchar` donde prd tiene `bigint` cambia el orden y las comparaciones. |
| `MovimientoSaldo` | `Provedores`: `varchar(30)` en prd vs **`bigint`** en dev (invertido respecto al caso anterior) | Alinear a prd (`varchar(30)`). |
| `PlantillasMasivas` | Dev **no tiene** la columna `Plantilla varchar(200)` | Agregar. |
| `TempoConvenios` | `Estado`: `varchar(200)` en prd vs `varchar(20)` en dev | Ampliar en dev. Riesgo de truncado silencioso hoy. |
| `ParametrosCodigosCompensacion` | `Formula`: `nvarchar(MAX)` en prd vs `varchar(MAX)` en dev | Alinear a `nvarchar(MAX)`. |
| `ParametrosLimitePendientes` | `Clave` y `Valor`: `NULL` en prd vs **`NOT NULL`** en dev | Dev es más estricto. Alinear a prd para evitar fallos de inserción que prd no tiene. |
| `BalanceGeneral` | Mismas 38 columnas, distinto **orden** (`flag` e `ID` intercambiados) | Cosmético, **pero** rompe cualquier `INSERT ... SELECT *` o `SELECT *` posicional. `HU06` hace `Select * from #TemProgramacion` — revisar si hay algún patrón igual sobre `BalanceGeneral`. |

### 4.5 Stored procedures

| SP | PRD | DEV | Diferencia |
| --- | --- | --- | --- |
| `SelectEnvioCorreos` | 36 líneas | 36 líneas | **Idéntico.** |
| `SP_ConciliacionFondos` | 1109 líneas | 1110 líneas | **Diferencia funcional:** prd calcula `Sum(Ventas) + Sum(NovedadesEntradas) + Sum(SinClasificacionE)`; dev omite `Sum(SinClasificacionE)`. Además dev usa `@contadorM` y prd `@ContadorM` (irrelevante en SQL Server). |
| `SP_UpdateVatcoAndProsegur` | 159 líneas | 104 líneas | **Diferencia funcional grande.** Ver abajo. |
| `SP_PasarHistoricoFondos` | 1835 líneas | **no está** | Ver 4.2. |

**`SP_UpdateVatcoAndProsegur` — lo que prd tiene y dev no:**

1. El `DELETE` pasó a sintaxis con alias (`DELETE B1 FROM ... AS B1`) para poder correlacionar.
2. Prd **comentó** dos condiciones que dev sigue ejecutando: `Or (codigo='' And NombrePunto='')` y `Or TotalIngresos=''`. **Dev borra filas que prd conserva.**
3. Prd agregó condiciones de borrado que dev no tiene:
   - `NombrePunto` ∈ {`TRASLADOS FONDO BANCOLOMBIA`, `TRASLADO FONDO BANCOLOMBIA`, `TRASLADOS`, `TRASLADO`, `CAMBIOS DE EFECTIVO`, `TRASLADOS PROSEGUR IBAGUE`} con `TotalIngresos=''`
   - `TipoMovimiento='SinMovimientoEntradas'` **con `EXISTS`** correlacionado contra otra fila de la misma `Ciudad`+`FechaRecoleccion` con `TipoMovimiento='Salidas'`
   - `TipoMovimiento` ∈ {`SinClasificarE`,`SinClasificarS`} con `Codigo` vacío/0 y `TotalIngresos` vacío/0, y la variante con `NombrePunto` vacío/NULL
4. Prd agregó 5 `UPDATE` al final sobre `ProsegurCiudades` que quitan la parte decimal con `TRY_CAST(TRY_CAST(x AS decimal) AS varchar(200))` en `TotalIngresos`, `SaldoAnterior`, `SaldoFinal`, `Billetes` y `Monedas`.

> Este es el punto más delicado del empalme: **hoy los dos ambientes producen resultados distintos con los mismos insumos.** Cualquier prueba en dev que se compare contra prd va a dar diferencias que no son bugs del cambio nuevo, sino de este desfase.

### 4.6 Capas de migración de datos

| Capa | Tablas | Qué copiar |
| --- | --- | --- |
| **T0 — Esquema** | Las 41 de prd | DDL completo, con las 13 PK y las 25 columnas IDENTITY |
| **T1 — Configuración** | `ParametrosFondos`, `ParametrosCalendarioCierresContables`, `ParametrosClaseDocumentos`, `ParametrosCodigosCompensacion`, `ParametrosDivisionCEBEGenerico`, `ParametrosHomologacionPtosAagiles`, `ParametrosLimitePendientes`, `ParametrosRecepcionCorreos`, `ParametrosTipoMovimientos`, `ParametrosCambioEntreFondos`, `CorreosNotificacionFondos`, `DiasCaidaFondos` | Copiar completas y **re-apuntar los valores de ambiente** (ver 4.7) |
| **T2 — Estado reciente** | `BalanceGeneral`, `ProsegurCiudades`, `ConciliacionFondos`, `ConciliacionFondosBW`, `Conciliacion`, `MovimientoMes`, `MovimientoSaldo`, `MovimientoTransportadoras`, `Programacion_P`, `Programacion_V`, `TempoConvenios`, `Control_Insumo_Fondos`, `Control_Insumo_FondosConsolidado`, `ControlHU`, `PlantillasMasivas`, `TicketInsumo`, `Temp` | Últimos 1–2 meses, suficiente para una corrida realista |
| **T3 — Históricos** | Las 10 `Historico*` | Crear vacías. No copiar datos: solo sirven para probar que `SP_PasarHistoricoFondos` inserta. |
| **T4 — Descartable** | `PruebaProsegur` y las 5 tablas solo-dev de 4.3 | No migrar / retirar |

### 4.7 Valores de `ParametrosFondos` que hay que re-apuntar en DEV

Después de copiar T1 desde prd, estas claves apuntan a producción y **hay que cambiarlas** o el bot de dev escribe en las rutas y buzones de prd:

| Clave | Por qué hay que revisarla |
| --- | --- |
| `SchemaFase1` | Debe apuntar al esquema de Fase I **de dev**, no al de prd |
| `Scheme`, `Schema`, `TablaConfig` | Deben resolver a `RPA_QA.ConciliacionFondos` |
| `RutaRed`, `PathLog`, `PathPlantillas`, `PathProgramaciones`, `PathProsegur`, `PathVatco`, `PathProveedores`, `PathConsolidado`, `RutaReporteErrores` | Rutas UNC de producción (`\\fserver\RPA - Conciliación Fondos de Efectivo\...`). Apuntar a un árbol de dev o el bot escribe sobre los archivos reales |
| `CorreoFallas`, `EmailTableName` | Redirigir a un buzón de pruebas |
| `SharePointSiteName`, `SharePointSitePath`, `SiteSubDom` | Sitio de SharePoint de pruebas |
| `ConnectionName` | Nombre de la conexión SAP — debe ser la de QA/dev |
| `UsuarioSAP`, `ClientSAP`, `LanguajeSAP` | Mandante SAP de dev |
| `NombreLicencia`, `CodigoRobot` | Identidad del runner de dev |
| `FolderBWHU26`, `FolderFiltrosHU26` | Carpetas de descarga de HU26 |

Las que se pueden dejar igual: `ActivarLog`, `DiasAdicionales`, `DiasBorrar`, `DiasMantenerLog`, `NameBot`, `NomFuncionCaracter`, `NomInsumoControl`, `NumberAttemps`, `Retries`.

---

## 5. Control Room de DEV — checklist

| Elemento | Valor en PRD | Qué hacer en DEV |
| --- | --- | --- |
| Global Value `$@Server$` | servidor de prd | Apuntar al SQL Server de dev |
| Global Value `$@Database$` | `RPA` | `RPA_QA` (o el nombre real de la BD de dev) |
| Global Value `$@DataBase$` | ídem (ojo: existen las dos grafías, `$@Database$` y `$@DataBase$`) | Crear **ambas**, con el mismo valor |
| Global Value `$@UrlControlRoom$` | `https://colsubsidio-2.my.automationanywhere.digital` | URL del CR de dev |
| Global Value `$@TenantID$`, `$@TenantName$` | tenant de Colsubsidio | Confirmar si aplica en dev |
| Global Value `$@UserChangeLicense$`, `$@PassChangeLicense$` | usuario del cambio de licencia | Solo si se va a probar `CambioLicencia` en dev |
| Global Value `$@EmailServerHost_ForReading$` | servidor de correo | Apuntar a buzón de pruebas |
| Locker `Globales` → `DBColsubsidio` (`UsuarioBD`, `ContrasenaBD`) | credencial de BD | Credencial del SQL de dev |
| Locker `Globales` → `correoRPA` (`UserCorreo`) | correo del bot | Correo de pruebas |
| Locker `Globales` → `TenantIDColsubsidio` (`TenantID`) | **solo lo usa dev** | Se retira si se toma la Opción A de 3.1 |
| Locker `ConciliacionFondos` → `ApiSharepoint_ConciliacionFondos` (`ClientId`, `ClientSecret`) | credencial SharePoint | Necesaria solo en el camino viejo (Opción B) |
| Locker `ConciliacionFondos` → `SapLogonConi` (`UserSap`, `PasswordSap`) | SAP producción | **Credencial de SAP QA** |
| Locker `ConciliacionFondos` → `CorreoBotConi` (`Usser`) | correo del bot | Correo de pruebas |
| Locker `ClientCredentialsEmail` → `ClientCredentialsEmail` | envío de correos | Réplica en dev |
| Locker `Cajeros` → `ApiSharepoint_CuadreIntegral`, `CorreoBotCajero` | compartidos con Fase I | Depende del empalme de Escarlet |
| Locker `FacturacionElectronica` → `FacturacionElectronicaSalud` | SAP de otro proceso | Confirmar por qué CONI lo referencia |
| **Conexión OAuth `APISharepoint`** | **existe solo en PRD** | **Crearla en dev** si se toma la Opción A |

> Nota: ningún bot tiene servidor, base de datos ni ruta hardcodeados. La única cadena de producción incrustada en el código es la URL `colsubsidio-2.my.automationanywhere.digital`, y está dentro de un `MessageBox` **deshabilitado** de `CambioLicencia` de prd — que el refactor de dev ya eliminó.

---

## 6. Orden de ejecución

**Fase 1 — Confirmar antes de tocar nada**

1. Correr el Anexo A en **prd** y en **dev**. Confirmar: (a) si `SP_PasarHistoricoFondos` existe en dev; (b) si `ConciliacionFondosBW` y `Conciliacion` existen en dev pese a no estar en el DDL entregado; (c) el nombre real de la BD de dev; (d) si el esquema de Fase I existe en dev con `MatrizPuntos` y `MediosPago`.
2. Decidir la Opción A o B de 3.1 (SharePoint). Es la única decisión que bloquea el resto.

**Fase 2 — Base de datos de dev**

3. Backup de `RPA_QA.ConciliacionFondos`.
4. Renombrar a `ZZ_*` las 5 tablas de 4.3. No borrar todavía.
5. Crear las 15 tablas faltantes (4.2) con el DDL de prd, incluidas sus PK e IDENTITY.
6. Aplicar los 6 ajustes de columnas de 4.4. El de `ConciliacionFondos.ID` requiere recrear la tabla (agregar IDENTITY a una tabla existente no es un `ALTER`): crear `ConciliacionFondos_new`, `INSERT ... SELECT` con lista de columnas explícita, renombrar.
7. Desplegar en dev los 4 SPs **con el código de prd**, incluido `SP_PasarHistoricoFondos`.
8. Copiar T1 (configuración) desde prd.
9. **Re-apuntar los valores de 4.7.** Verificar clave por clave antes de seguir — este es el paso que evita que dev escriba en producción.
10. Copiar T2 (últimos 1–2 meses).

**Fase 3 — Control Room de dev**

11. Crear/actualizar Global Values y lockers según la tabla de la sección 5.
12. Si Opción A: crear la conexión OAuth `APISharepoint` en el CR de dev.
13. Verificar que los packages de la sección 3.2 estén disponibles en el CR de dev.

**Fase 4 — Bots**

14. Importar el export de **PRD** completo sobre dev, sobrescribiendo. Deja dev igual a prd en 34 bots.
15. Reaplicar el delta de dev: sustituir `CambioLicencia` por la versión de dev (refactor de variables).
16. Deshabilitar el step **"Pruebas unitarias"** de `HU26_DescargarDocumentoSAP`.
17. Según la decisión de 3.1: (A) borrar de dev los 3 bots `*SharepointGeneral` y limpiar el código muerto de `CargarInsumoRepositorio`; o (B) importar los 3 bots a prd, reactivar las ramas y borrar el camino de Control Room.

**Fase 5 — Validación**

18. Correr `HU00_DespliegeAmbiente` solo. Verificar que `Config{Schema}` resuelve a `RPA_QA.ConciliacionFondos` y que las 35 claves cargan.
19. Correr `HU01`–`HU03` y comprobar que los insumos se leen de rutas de dev.
20. Correr `SP_UpdateVatcoAndProsegur` y `SP_ConciliacionFondos` sobre un día conocido y **comparar contra el resultado de prd del mismo día**. Deben coincidir. Si no coinciden, el desfase de 4.5 no quedó cerrado.
21. Correr `Main_ConciliacionFondos` completo y verificar que `SP_PasarHistoricoFondos` puebla las 10 tablas `Historico*`.
22. Confirmado todo: borrar las `ZZ_*` del paso 4.

---

## 7. Riesgos y preguntas abiertas

| # | Riesgo / pregunta | Impacto | Estado |
| --- | --- | --- | --- |
| 1 | ¿Existe el esquema de Fase I (`SchemaFase1`) en dev, con `MatrizPuntos` y `MediosPago` pobladas? | 12 de 26 bots no se pueden probar sin él | **Abierto — bloqueante** |
| 2 | ¿`SP_PasarHistoricoFondos` existe en dev y no se exportó, o de verdad no está? | `Main_ConciliacionFondos` lo invoca en ambos ambientes | **Abierto — bloqueante** |
| 3 | ¿`ConciliacionFondosBW` y `Conciliacion` existen en dev pese a no estar en el DDL? | 10 bots dependen de la primera | **Abierto — bloqueante** |
| 4 | SharePoint: Opción A o B (sección 3.1) | Define si se retiran 3 bots y 8 versiones de paquete, o si suben a prd | **Abierto — decisión del equipo** |
| 5 | Los dos ambientes **calculan distinto** hoy (`SP_UpdateVatcoAndProsegur` y `SP_ConciliacionFondos`, sección 4.5) | Cualquier comparación dev↔prd da falsos positivos hasta que se alinee | Identificado, se cierra en el paso 7 |
| 6 | `CambioLicencia` **falla en producción** (`Column Name : token does not exists`) los días 30 y 31/08 | El cambio de licencia no ocurre. Mismo fallo reportado en Escarlet | Identificado, **el refactor de dev no lo corrige** |
| 7 | `HU26` corre con un step **"Pruebas unitarias" habilitado en producción** que llama a `HU00` | Ejecuta la limpieza de logs y carpetas de `HU00` una vez más de lo previsto | Identificado, se cierra en el paso 16 |
| 8 | `CargarInsumoRepositorio` genera **467 errores** `El insumo a cargar NO existe en la ruta` en 3 días | Ruido que tapa errores reales. No es del empalme, pero conviene resolverlo mientras se toca el bot | Identificado |
| 9 | Dev tiene **0 primary keys** en 31 tablas | Duplicados silenciosos que prd sí bloquea | Se cierra en el paso 5 |
| 10 | `BalanceGeneral` tiene distinto orden de columnas | Rompe `INSERT ... SELECT *` posicional | Se cierra en el paso 6 |
| 11 | ¿Por qué CONI referencia el locker `FacturacionElectronica`? | Puede ser un acoplamiento no documentado con otro proceso | **Abierto** |
| 12 | El `wait` de SAP en `HU26` subió de 15 a 100 en prd | Si dev tiene un SAP más rápido, 100 alarga las corridas sin necesidad | Menor |

---

## Anexo A — Extraer y comparar esquema (correr en PRD y en DEV)

```sql
-- 1. Tablas y columnas
SELECT t.name AS Tabla, c.column_id, c.name AS Columna,
       ty.name AS Tipo, c.max_length, c.precision, c.scale,
       c.is_nullable, c.is_identity
FROM   sys.tables t
JOIN   sys.schemas s  ON s.schema_id = t.schema_id
JOIN   sys.columns c  ON c.object_id = t.object_id
JOIN   sys.types  ty  ON ty.user_type_id = c.user_type_id
WHERE  s.name = 'ConciliacionFondos'
ORDER  BY t.name, c.column_id;

-- 2. Primary keys e índices
SELECT t.name AS Tabla, i.name AS Indice, i.type_desc, i.is_primary_key, i.is_unique,
       STUFF((SELECT ', ' + c2.name
              FROM sys.index_columns ic2
              JOIN sys.columns c2 ON c2.object_id = ic2.object_id AND c2.column_id = ic2.column_id
              WHERE ic2.object_id = i.object_id AND ic2.index_id = i.index_id
              ORDER BY ic2.key_ordinal FOR XML PATH('')), 1, 2, '') AS Columnas
FROM   sys.indexes i
JOIN   sys.tables  t ON t.object_id = i.object_id
JOIN   sys.schemas s ON s.schema_id = t.schema_id
WHERE  s.name = 'ConciliacionFondos' AND i.type > 0;

-- 3. Procedimientos y su código
SELECT o.name, o.type_desc, m.definition
FROM   sys.objects o
JOIN   sys.sql_modules m ON m.object_id = o.object_id
JOIN   sys.schemas s     ON s.schema_id = o.schema_id
WHERE  s.name = 'ConciliacionFondos'
ORDER  BY o.name;

-- 4. Confirmar los bloqueantes de este plan
SELECT 'ConciliacionFondosBW' AS Objeto, OBJECT_ID('ConciliacionFondos.ConciliacionFondosBW') AS Existe
UNION ALL SELECT 'Conciliacion',           OBJECT_ID('ConciliacionFondos.Conciliacion')
UNION ALL SELECT 'SP_PasarHistoricoFondos',OBJECT_ID('ConciliacionFondos.SP_PasarHistoricoFondos');

-- 5. Confirmar la dependencia de Fase I (reemplazar <SchemaFase1> por el valor real)
SELECT Nombre, Valor FROM ConciliacionFondos.ParametrosFondos WHERE Nombre = 'SchemaFase1';
-- luego:
-- SELECT 'MatrizPuntos', OBJECT_ID('<SchemaFase1>.MatrizPuntos')
-- UNION ALL SELECT 'MediosPago', OBJECT_ID('<SchemaFase1>.MediosPago');

-- 6. Conteo de filas por tabla (para dimensionar T2)
SELECT t.name AS Tabla, SUM(p.rows) AS Filas
FROM   sys.tables t
JOIN   sys.schemas s    ON s.schema_id = t.schema_id
JOIN   sys.partitions p ON p.object_id = t.object_id AND p.index_id IN (0,1)
WHERE  s.name = 'ConciliacionFondos'
GROUP  BY t.name ORDER BY Filas DESC;
```

Guardar la salida de cada consulta en los dos ambientes y diffear.

---

## Anexo B — Comparar dos exports de Automation Anywhere

Los `.atmx`/bots exportados por AA son JSON con `uid` aleatorios por nodo, así que un `diff` crudo es inservible. El método usado en este análisis:

1. **Hash primero.** `md5sum` de cada archivo (excluyendo `*Metadata/*.png`) para separar idénticos de distintos. Aquí redujo 34 archivos a 5.
2. **Normalizar el resto.** Renderizar cada bot como pseudocódigo indentado: propiedades, paquetes ordenados, variables ordenadas por nombre, y el árbol de nodos (`commandName`, `packageName`, flag `disabled`, `returnTo`, atributos, `children`/`branches` recursivos), eliminando `uid`/`nodeId`/`id`. El script queda en el scratchpad de la sesión (`aa_norm.py`).
3. **Diffear el pseudocódigo.** Las diferencias que quedan son reales.
4. **Contrastar con `manifest.json`.** El campo `scannedDependencies` de cada bot dice qué dependencias ve AA — que no es lo mismo que las referencias textuales del archivo: **AA no escanea nodos deshabilitados.** Esa discrepancia fue justo lo que reveló que los 3 bots de SharePoint quedaron como código muerto en prd.
5. **Cruzar código contra DDL.** Extraer las referencias `<esquema>.<tabla>` de los bots y de los SPs y compararlas contra las tablas del DDL de cada ambiente. Ojo con los falsos positivos: `#TemProgramacion` es una tabla temporal, no una tabla del esquema.

---

## Anexo C — Qué no se hizo y por qué

| No se hizo | Por qué |
| --- | --- |
| Abrir `documentacion-general/` (especificación funcional V2.2, manual de usuario) | Este documento es un diff técnico entre ambientes. La especificación describe **qué debe hacer** el proceso, no **qué difiere** entre dev y prd. Pendiente para la ingesta al vault. |
| Abrir `transcripciones/` y `tmp/grabaciones/` | Fuera del alcance del diff. Pueden contener el porqué de los cambios de prd (sobre todo el "Cambio monitoreo 31/10/2025"). |
| Abrir los `.zip` de backup y `dml-conciliacion-fondos/` | Por regla del proyecto (`ingestion-pipeline`). La carpeta `dml-` además está vacía. |
| Abrir `modelo-relacional-coni.png` | El DDL da la misma información en forma verificable. |
| Comparar datos (filas) entre ambientes | No hay dumps de datos en el material, solo estructura. |
| Proponer el DDL de las 15 tablas faltantes | Se copia literal del DDL de prd; escribirlo aquí duplicaría la fuente y arriesgaría desincronizarla. |
| Decidir Opción A vs B de SharePoint | Es una decisión de arquitectura del equipo, no un hallazgo del análisis. |

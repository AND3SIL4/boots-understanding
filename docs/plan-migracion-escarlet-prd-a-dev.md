# Plan de empalme PRD → DEV — Escarlet (Cuadre de Cajeros)

Fecha del análisis: 2026-09-01
Fuentes: `_inbox/escarlet-cuadre-cajeros/` (exports AA de dev y prd, DDL, SPs, logs de prd, transcripciones, documento de especificación V2.4).
No se abrieron: los `.zip` de backup, los videos, la carpeta `dml-*`, las capturas `*Metadata/*.png`.

---

## 0. Resumen ejecutivo

1. **El código de los bots casi no difiere.** De los 31 bots exportados, 25 son idénticos byte a byte entre dev y prd. Los 6 restantes tienen cambios pequeños y localizados (sección 3). El empalme de bots es un import con sobrescritura más dos retoques.
2. **El ambiente no lo decide el bot, lo decide la configuración externa.** `HU00_DespliegeAmbiente` toma servidor y base de datos de los *Global Values* del Control Room (`$@Server$`, `$@Database$`), elige el esquema `RPA.Cajero` o `RPA_QA.Cajero` según el nombre de la BD, y carga toda la tabla `ParametrosConfiguracion` en el diccionario `Config` que usan todos los bots. Por eso el mismo export corre en ambos ambientes. **El trabajo real del empalme es base de datos + Control Room, no bots.**
3. **Bloqueante: el export está incompleto.** Cubre 3 de los 4 mains. Falta `Main_CuadreCajerosInsumos` y otros 22 bots que sí aparecen ejecutando en los logs de producción (HU02–HU11, plantillas HU16, HU17, HU27, HU28, dos funciones). Sin ese export no se puede empalmar el proceso "Insumos" (el de la 1 PM).
4. **Bloqueante: solo hay DDL de producción (`RPA.Cajero`).** No hay DDL de `RPA_QA.Cajero`. El diff real de esquemas hay que generarlo contra el servidor de dev (script en el anexo A).
5. **La BD tiene 268 tablas, pero los bots exportados usan 60.** Clasificarlas por capas (esquema completo / configuración / estado reciente / históricos / temporales) es lo que hace el empalme eficiente: se copia todo el esquema, pero datos solo de lo que importa.
6. **Hallazgo de seguridad:** la versión de producción de `HU15B_MPInsumosProcesados` tiene un paso deshabilitado con usuario y contraseña de SQL en texto plano dentro del export, y la tabla `Cajero.Credenciales` guarda claves en claro. Rotar y limpiar antes de copiar nada a dev.

---

## 1. Inventario analizado

| Fuente | Qué es | Estado |
| --- | --- | --- |
| `export-aa/produccion/` (2026-09-01 16:02) | 31 bots, manifest con dependencias | Analizado completo |
| `export-aa/desarrollo/` (2026-09-01 16:05) | Mismos 31 bots | Analizado completo, diff contra prd |
| `database-sqlserver/ddl-escarlet-cuadre-cajeros.sql` | DDL de `RPA.Cajero`: 268 tablas, 10 índices, 2 vistas, 0 FK, 22 PK | Analizado |
| `database-sqlserver/stored-procedures/` | 10 SPs | Analizados (tablas que tocan) |
| `logs-ejemplos/` | 5 logs de prd (31/08 y 01/09) de las 3 máquinas | Analizados |
| `transcripciones/` | 4 sesiones (28/08, 31/08, 01/09 x2) | Leídas |
| `documentacion-general/md/` | Especificación V2.4 (HU01–HU26) | Índice revisado |
| `export-aa/*/*.zip`, `tmp/grabaciones/`, `dml-*` | Backups, videos, DML | **No abiertos** (por regla) |

---

## 2. Cómo se selecciona el ambiente hoy (mecanismo verificado en el export)

```
HU00_DespliegeAmbiente (lo llaman todos los bots al inicio)
  Server   = $@Server$              ← Global Value del Control Room
  DataBase = $@Database$            ← Global Value del Control Room
  ProductiveEnvironment = true      ← constante en el bot (igual en dev y prd)
  if DataBase == "RPA"  → Scheme = "RPA.Cajero"
  else                  → Scheme = "RPA_QA.Cajero"
  connect(Server, DataBase, locker Globales/DBColsubsidio)
  Config{*} = SELECT * FROM <Scheme>.ParametrosConfiguracion   ← todo lo demás sale de aquí
```

- Los bots `HU15B_*` tienen un input `In_ActivarDB_Productivo`. Si es `true` conectan con el locker `Cajeros/CuadreCajerosBDPRD` (servidor y BD dentro del locker). `Main_CuadreCajero_HU15_V3` **no pasa ese input**, así que en la práctica siempre usan el camino de Global Values.
- Todo lo demás que cambia entre ambientes vive en `ParametrosConfiguracion` (rutas de red, ruta de logs, correos, nombres de licencia, IDs de trigger, conexión SAP) y en los lockers.

**Implicación:** si la BD de dev también se llama `RPA` (en otro servidor), HU00 usará `RPA.Cajero` y no `RPA_QA.Cajero`. Hay que confirmar el nombre real de la BD de dev antes de decidir en qué esquema se crea el DDL.

Claves de `ParametrosConfiguracion` que los bots exportados leen: `Schema`, `Scheme`, `RutaRed`, `Ruta20`, `PathLog`, `TablaConfig`, `DB`, `DataBase`, `Server`, `ConexionSap`, `ClientSAP`, `TransaccionSAP2`, `MesesBorrarTablas`, `MesesHistoricoPartidasAbiertas`, `DiasCaidaRegistrosAntiguosVATCO`, `DiasCaidaRegistrosAntiguosPROSEGUR`, `TriggerIntegralID`, `TriggerIntegralNombre`, `TriggerIntegralUsuario`, `CorreoBot`, `CodigoRobot`, `NombreLicencia` (y variantes `<usuario>_N`, `_C`, `_I`, `_11AM` que consulta `CambioLicencia`).

---

## 3. Diferencias DEV vs PRD en los bots

Resultado del diff semántico (variables, paquetes, nodos y sus atributos), ignorando renumeración.

| Bot | Quién está más nuevo | Qué cambia | Acción para el empalme |
| --- | --- | --- | --- |
| `HU00_DespliegeAmbiente` | PRD | En prd el borrado de logs con más de 1 año (`deleteFiles` sobre `PathLog`) está **activo**; en dev está deshabilitado. | Tomar PRD. |
| `HU15B_GenerarCuadreCajeros` | PRD | Prd agrega 7 trazas INFO (existencia de info del punto, existencia del archivo, fin de hoja Detalle/Cierre Datáfono, fin de cuadre) y una rama `else` cuando el archivo no existe. Solo trazabilidad. | Tomar PRD. |
| `HU15B_MPInsumosProcesados` | **DEV** | Dev: encabezado "HU27 Consignaciones, 25/04/2024", paquetes más nuevos, `In_ActivarDB_Productivo` como input, credenciales desde locker. Prd: variable local `ActivarDB_Productivo`, sesiones de BD con nombre (`DiaCaida`, `MP`, `Certificaciones`), y un step deshabilitado "Pruebas" con **usuario y contraseña SQL en claro**. | Conflicto real. Base = DEV. Verificar si los nombres de sesión de prd son necesarios (dev usa sesión por defecto en tres conexiones simultáneas, eso puede pisarse). Eliminar el step con credenciales y rotar esa cuenta. |
| `HU19_DescargueJobsPropios` | PRD | Ajuste del 24/09/2024: el `BULK INSERT` a `BaseCuadreCargueBW` lee desde `$RutaRed$FASE II CUADRE INTEGRAL\1. DESCARGUE PARTIDAS ABIERTAS\...` en vez de `$In_Config{Ruta20}$`; la copia del TXT a `Ruta20` quedó deshabilitada. | Tomar PRD. **Ojo:** el SQL Server de dev debe poder leer la ruta de red que quede en `RutaRed` de dev, o el bulk falla. |
| `CambioLicencia` | DEV | Dev renombró variables (`pX` → `X`), comentario "25/11/2024". Prd tiene message boxes deshabilitados. Funcionalmente iguales. Hoy **falla en prd** los dos días de log: `Column Name : token does not exists` (la respuesta del CR no trae `token`). | Tomar DEV. No bloquea el empalme: en dev el trigger de licencia se desactiva de todas formas. |
| `LoginSAPLogon_copy` | PRD | Mismo flujo; dev tiene paquetes de 2020, prd de 2023 y formato de export distinto. Solo lo usa `HU18`. | Tomar PRD. |

**Regla resultante:** importar el export de PRD completo sobre dev (sobrescribir) y después reaplicar los dos cambios que solo existen en dev (`HU15B_MPInsumosProcesados`, `CambioLicencia`). Así dev queda igual a prd más un delta conocido y pequeño, en lugar de un dev con historia desconocida.

### 3.1 Bots que faltan en el export (aparecen en logs de prd)

`Main_CuadreCajerosInsumos`, `HU03_LogIntegrador`, `HU03_LogTransacciones`, `HU04_TransferenciaArchivosSFTP`, `HU05_VatcoEfectivo`, `HU06_VatcoDocumentos`, `HU07_ProsegurEfectivoPorCiudad`, `HU08_ProsegurDocumentos`, `HU09_Proceso de Documentos Canje`, `HU10_DescargueConexo`, `HU11_Operadores`, `HU16_MainGeneracionPlantillaMasiva`, `Plantillas/HU16_GeneracionPlantilla{CanjeProsegur, CanjeVatco, CertificacionesProsegur, CertificacionesVatco, Operadores}`, `HU17_Novedad`, `HU27_Consignaciones`, `Plantillas/HU27_GeneracionPlantillaConsignaciones`, `HU28_NCR`, `Functions/ConvertXlsToXlsx`, `Functions/UnificadorArchivos(TXTyPAG)`.

Son 23. Todos pertenecen al main "Insumos" (corre 13:00 en `CGOTCCMP`). Hay que repetir el export desde la carpeta `CuadreCajeros` completa, con dependencias, en ambos ambientes.

---

## 4. Base de datos

### 4.1 Qué hay

- Un esquema `Cajero` en BD `RPA`. 268 tablas, 2 vistas (`View_BulkTLogConsHu22`, `View_ReporteCuadreCajero`), 10 índices, 10 SPs.
- **0 llaves foráneas**, 22 llaves primarias, casi todo `varchar NULL`. La BD no valida nada: si falta un registro de configuración, el error aparece en runtime en el bot, no en SQL. Por eso el orden de carga de datos importa más que el esquema.
- 10 SPs. Los bots exportados invocan `SP_BaseCuadreBW` (desde `HU20`). Los otros 9 los usan bots no exportados o el proceso de Bonos/Integral: `InsertConexoToBonosFisicos`, `InsertProsegurToBonosFisicos`, `InsertVatcoToBonosFisicos`, `InsertIntoTicketInsumoBonosHU23`, `QueriesHU17`, `QueriesHU18`, `SP_PasarHistoricoBonos`, `SP_PasarHistoricoCuadreIntegral`, `UpdateBaseCuadreBWIntegral`.
- Los bots exportados referencian 60 tablas, todas presentes en el DDL. 208 tablas no las toca ningún bot exportado: son de los bots faltantes, de los procesos hermanos (Bonos, Cuadre Integral comparten esquema) o residuos (`Prosegur24092024`, `SODEXO24092024`, `PruebasMatriz`, `Validaci`, `fechas`, `ejefecha`).

### 4.2 Capas de migración (esto es lo que ahorra tiempo)

| Capa | Qué | Cómo se migra | Tablas (aprox.) |
| --- | --- | --- | --- |
| **T0 Esquema** | Todo el DDL + vistas + índices + 10 SPs | Script idempotente (`IF OBJECT_ID(...) IS NULL`) sobre el esquema de dev. Se corre siempre completo. | 268 |
| **T1 Configuración y maestros** | Tablas que el bot lee para saber qué hacer | Copia completa desde prd, luego script de re-apuntamiento (4.4). Es lo que hoy explica "la tabla no coincide". | ~70 |
| **T2 Estado operativo reciente** | Tablas transaccionales que el bot lee y escribe cada día | Copia por ventana de fechas. El proceso reprocesa hasta 15 días atrás (`DiasCaidaInsumo`), así que la ventana mínima útil es **20 días**. | ~40 |
| **T3 Históricos** | `Historico*` (15 tablas) | Solo estructura, salvo que se vaya a probar `PasoAHistoricos` / `SP_PasarHistorico*`; en ese caso, mes en curso. | 15 |
| **T4 Temporales, contingencia, matrices, residuo** | `Temp*` (57), `*_Contingencia` (19), `M_*` (24), tablas con fecha en el nombre | Solo estructura. Nunca datos. | ~140 |

**T1 (copiar completas):**
`ParametrosConfiguracion`, `ParametrosIntegral`, `ParametrosBonos`, `ControlHU`, `ControlHUBonos`, `ControlHUIntegral`, `ControlHUIntegral2`, `ControlMain`, `DiasCaidaInsumo`, `DiasCaidaBonos`, `DiasCaidaIntegral`, `CALENDARIOCONTABLE`, `Festivos`, `MediosPago`, `MedioDePago`, `MEDIOPAGO_EXLUIDO`, `Negocio`, `Transportadoras`, `TransportadorasIntegral`, `MatrizPuntos`, `CMatrizPuntos`, `CorreosNotificacion`, `CorreosNotificacionBonos`, `CorreosNotificacionIntegral`, `CorreoRemitente`, `CorreosRemitenteIntegral`, `CiudadCorreo`, `OperadorCorreo`, `Convenios`, `CONVENIOPROCESA`, `REGLASTOLERANCIA`, `ReglasToleranciaBonos`, `ReglasToleranciaEfectivo`, `ReglasToleranciaIntegral`, `ToleranciaCajeroC`, `TitulosAExcluir`, `HomologacionReferencia`, `HomologacionSodexo`, `TablaReferencia`, `CodGenericoCajero`, `CeBeGenerico`, `NitOperadores`, `NomInsumo`, `ArchivosTesoreria`, `InsumoLegado`, `UsuariosCajeros`, `Credenciales` (⚠ claves en claro), `EstadosBonos`, `Franquicias`, `CuentasBancos`, `CUENTAS_FONDOS`, `AsignacionCuentaSAP`, `CodigosCompensacion`, `CodDaviplata`, `CiudadReteICA`, `DivisionGenericos`, `DivisionGenericosBonos`, `EstructuraBancoAvVillas`, `EstructuraBancoBancolombia`, `EstructuraBancoColpatria`, `EstructuraBancoOccidente`, `EstructuraControlInsumosBonos`, `Estructura_Control_Insumos_Integral`, `PlantillasBonos`, `PlantillasIntegral`, `ProrrateoPlantillasIntegral`, `JobSAP`, `GestionManualVat`, `Operadores`.

**T2 (ventana ≥ 20 días, por columna de fecha del robot):**
`BaseCuadreBW`, `BaseCuadreCargueBW`, `BaseCuadre`, `BaseCuadreCargue`, `CUADRECAJERO`, `C_CUADRECAJERO`, `CUADRE`, `LogCons`, `Control_ReporteInsumo`, `Control_Insumo`, `Control_Insumo_Resultados`, `CMP_InsumosProcesados`, `MP_InsumosProcesados`, `MP_InsumosProcesadosTEMP`, `CargueDrive`, `CuadresPendientesTemp`, `ValidacionCertificadosPorPunto`, `INVENTARIOCOLSUBSIDIO`, `Vatco_Insumos`, `Prosegur`, `SODEXO`, `REDEBAN`, `Procesa`, `Consignaciones`, `BONOSFOSFEC`, `TDC`, `GELSA`, `MPSIC`, `ContabilizacionCanje`, `ConsolidadoBancos`, `TicketInsumo`, `TicketInsumoBonos`, `CunterBW`, `HistoricoPartidasAbiertas` (la lee `HU20`; tratarla como T2 aunque se llame histórico).

Nota: HU00 borra de `LogCons`, `Vatco_Insumos`, `Prosegur`, `SODEXO`, `Operadores`, `REDEBAN`, `Control_ReporteInsumo`, `Procesa`, `Control_Insumo`, `BONOSFOSFEC`, `Consignaciones` todo lo anterior a `MesesBorrarTablas` meses cada vez que corre `Main_CuadreCajerosParametros`. No vale la pena copiar más allá de ese parámetro.

### 4.3 Diff de esquema PRD vs DEV

No se puede hacer todavía: falta el DDL de dev. Pasos:

1. Ejecutar en dev el script del anexo A (extrae columnas de `INFORMATION_SCHEMA`) y guardarlo como `_inbox/escarlet-cuadre-cajeros/database-sqlserver/ddl-dev-escarlet.sql` o CSV.
2. Ejecutar lo mismo en prd.
3. Comparar (anexo A trae la consulta de diferencia si ambos resultados se cargan en una tabla). Salida esperada: tablas que faltan en dev, columnas que faltan, tipos/longitudes distintas.
4. Generar `ALTER TABLE ... ADD` / `ALTER COLUMN` solo para lo que falta. No borrar nada de dev en esta fase.

En `tor-cuenta-transitoria` ya se vio el mismo síntoma (tablas y columnas de prd que no existen en dev); esperar lo mismo aquí.

### 4.4 Valores que hay que re-apuntar en dev después de copiar T1

| Dónde | Qué | Por qué |
| --- | --- | --- |
| `ParametrosConfiguracion.RutaRed` | Hoy apunta a `\\192.168.50.169\rpa_conciliacion_integral_mp\` (file server de prd) | Si dev escribe ahí, pisa resultados de producción y publica cuadres falsos a los funcionales. |
| `ParametrosConfiguracion.PathLog`, `Ruta20` | Rutas locales/red de la máquina de prd | Deben existir en el runner de dev. |
| `ParametrosConfiguracion.CorreoBot`, tablas `CorreosNotificacion*`, `CorreoRemitente*`, `CiudadCorreo`, `OperadorCorreo` | Destinatarios reales (funcionales, droguerías) | En dev todo debe ir a un buzón de pruebas. Hoy prd ya falla con destinatarios vacíos o inválidos (`HU08`, `HU15`), así que además conviene sanear. |
| `ParametrosConfiguracion.NombreLicencia*`, `TriggerIntegral{ID,Nombre,Usuario}` | IDs de usuarios y bots del Control Room de prd | En dev no existen esos IDs. Desactivar el cambio de licencia y el trigger a Integral en dev. |
| `ParametrosConfiguracion.ConexionSap`, `ClientSAP`, `TransaccionSAP2` | Conexión SAP productiva | Apuntar a SAP QA (lo usa `HU18_CreacionJobsPropios`). |
| `Credenciales` | Usuario/clave por HU y aplicativo | Reemplazar por credenciales de QA. Nunca copiar las de prd. |
| `ControlHU.Activa`, `ControlHU.Estado`, `ControlMain.FechaEjecucion` | Semáforos de ejecución | Los mains solo arrancan si `FechaEjecucion` es anterior a hoy y encienden/apagan HU por `Activa`. Resetear antes de cada prueba. |
| `DiasCaidaInsumo` | Días hacia atrás que reprocesa cada HU | Define la ventana T2. Mantener igual a prd. |

---

## 5. Control Room de DEV — checklist

**Global Values** (todos referenciados en los bots): `Server`, `Database`, `EmailServerHost`, `EmailServerHost_ForReading`, `UrlControlRoom`, `UserChangeLicense`, `PassChangeLicense`, `EjecBotUser`, `EjecBotPass`.

**Lockers / credenciales:**

| Locker / credencial | Atributos | En dev |
| --- | --- | --- |
| `Globales/DBColsubsidio` | `UsuarioBD`, `ContrasenaBD` | Usuario SQL de QA. Es la conexión principal (105 usos). |
| `Cajeros/CuadreCajerosBDPRD` | `Server`, `Database`, `Username`, `Password` | Debe existir aunque apunte a QA, porque los `HU15B_*` lo leen al inicio aunque no lo usen. |
| `Cajeros/CorreoBotCajero` | `User`, `Pass` | Buzón de pruebas. |
| `ClientCredentialsEmail/ClientCredentialsEmail` | `TenantID`, `ClientID`, `ClientSecretValue` | App registration de Graph para correo (QA). |
| `Cajeros/OneDriveCajerosScarlet` | `TenantID`, `ClientID`, `ClientSecret` | Publicación a OneDrive/SharePoint (`HU15B_CargaIndividualDrive`). En dev: carpeta de pruebas o desactivar. |
| `Cajeros/SapCorporativoCajeros` | `User`, `Password` | SAP QA. |
| `Cajeros/BloqueoHojaExcel` | `Pass` | Igual que prd. |

**Runners y software.** En prd son 3 máquinas:

| Runner (usuario) | Mains | Horario prd |
| --- | --- | --- |
| `CGOTCCMP` | `Main_CuadreCajerosParametros`, `Main_CuadreCajerosInsumos` | 01:00 y 13:00 |
| `CGOTCCAV001` | `Main_CuadreCajerosPartidasAbiertasBW` | 03:00 |
| `CGRPA012` | `Main_CuadreCajero_HU15_V3` | 01:30 |

El runner de dev necesita: SAP GUI (`HU18`), Excel con macros (`CONSOLIDADOS\Macros.xlsm`), acceso al file server que quede en `RutaRed`, acceso al SFTP de transportadoras (bots no exportados), navegador para NCR (bot no exportado), Graph/OneDrive. Un solo runner de dev alcanza si se lanzan los mains en secuencia, no en paralelo.

**Schedules:** en dev no programar nada. Lanzar manual, un main a la vez.

---

## 6. Orden de ejecución

Cada fase tiene un criterio de salida. No pasar a la siguiente sin cumplirlo.

**Fase 0 — Completar insumos (bloqueante, sin esto no hay empalme).**
- Export de prd y de dev de la carpeta `Automation Anywhere\Bots\CuadreCajeros` completa con dependencias (debe incluir `Main_CuadreCajerosInsumos` y los 22 bots de 3.1).
- DDL/columnas de `RPA_QA.Cajero` (anexo A) y confirmación del nombre real de la BD de dev.
- Volcado de las tablas T1 de prd (bcp o "Generate Scripts → data only").
- Lista de Global Values y de lockers del CR de prd (nombres y atributos, no valores).
- Salida: los 4 mains están en el export; el DDL de dev está en `_inbox`.

**Fase 1 — Congelar línea base.**
- El export de prd del 2026-09-01 queda como línea base del código productivo. Cualquier cambio posterior en prd se hace sobre esa base y se registra.
- Salida: nadie edita en caliente en prd durante el empalme (acordado en la sesión del 31/08).

**Fase 2 — Esquema en dev (T0).**
- Aplicar el DDL completo, vistas, índices y los 10 SPs sobre el esquema de dev con script idempotente. Correr el diff del anexo A antes y después: después debe dar cero diferencias de tablas y columnas.
- Salida: diff = 0.

**Fase 3 — Configuración (T1) y re-apuntamiento.**
- Truncar T1 en dev, cargar T1 de prd, correr el script de re-apuntamiento de 4.4 (guardarlo en el repo, se reutiliza en cada refresco).
- Salida: `SELECT * FROM ParametrosConfiguracion` en dev no contiene ninguna ruta, correo ni ID de prd.

**Fase 4 — Estado reciente (T2).**
- Cargar los últimos 20 días de las tablas T2. Filtrar por la columna de fecha del robot de cada tabla (`FechaRobot`, `FECHA_ROBOT`, `FechaArchivo`, `FECHARECOLECCION`, según tabla).
- Salida: `SELECT COUNT(*), MIN(fecha), MAX(fecha)` coincide con prd para la ventana.

**Fase 5 — Control Room de dev.**
- Global Values, lockers, runner, sin schedules. Confirmar que `$@Database$` produce el esquema correcto en HU00.
- Salida: `HU00_DespliegeAmbiente` corre solo en dev y el log muestra `Scheme` y `Server` de QA.

**Fase 6 — Bots.**
- Importar el export de prd en dev con sobrescritura. Reaplicar en dev los dos cambios DEV-only (`HU15B_MPInsumosProcesados` sin el step de credenciales, `CambioLicencia` con variables renombradas). Comparar dev vs prd con el script del anexo B: solo deben diferir esos dos.
- Salida: diff de bots = 2 archivos conocidos.

**Fase 7 — Pruebas controladas, un main a la vez, en este orden.**
1. `Main_CuadreCajerosParametros` (HU01, HU24, HU15B_CargueInformacion). Depende de: insumos de transportadoras en `RutaRed`, correo. Sin SAP.
2. `Main_CuadreCajerosPartidasAbiertasBW` (HU18, HU19, HU20, HU26). Depende de: SAP QA, `BULK INSERT` desde `RutaRed`, `ActivarTrigger` a Integral (desactivar en dev).
3. `Main_CuadreCajero_HU15_V3` (HU15B_*, HU15_Consolidar*). Depende de: resultado de 2, OneDrive/SharePoint de pruebas, correo. Es el que tiene el caso abierto del 27/08 (cuadres generados vs publicados; `Error cargue drive cuadre 144533 del 25/08/2026` se repite los dos días de log).
4. `Main_CuadreCajerosInsumos` cuando llegue el export.
- Salida por main: termina sin `ERROR` en el log de dev y las tablas T2 de dev cambian igual que cambiarían en prd.

**Fase 8 — Gobernanza para que no se vuelva a abrir la brecha.**
- Todo ajuste nace en dev, se prueba con este ambiente, y se promueve a prd con export + import. Antes de cada import, correr el diff del anexo B y guardar el resultado en `control-cambios/`.
- Refrescar T1 de prd a dev cada vez que negocio cambie parámetros; T2 cuando se necesite reproducir un caso.

---

## 7. Riesgos y preguntas abiertas

| # | Riesgo / pregunta | Impacto | Mitigación |
| --- | --- | --- | --- |
| 1 | Export incompleto (23 bots, incluido un main) | No se puede probar Insumos ni HU02–HU11 | Fase 0. |
| 2 | Sin DDL de dev | No se sabe qué falta en dev hasta correr el diff | Anexo A. |
| 3 | Credenciales en claro en el export de prd y en `Credenciales` | Fuga; y copiarlas a dev multiplica la exposición | Rotar la cuenta SQL del step "Pruebas", limpiar el bot en prd, no copiar `Credenciales` de prd. |
| 4 | Dev escribiendo en `\\192.168.50.169` o enviando correos reales | Contaminar producción y a los funcionales desde dev | Re-apuntamiento (4.4) es obligatorio antes de la primera ejecución. |
| 5 | `BULK INSERT` de `HU19` requiere que el motor SQL de dev lea la ruta de red | HU19 falla en dev aunque el bot esté bien | Compartir la ruta de dev al servicio SQL de QA, o volver temporalmente a `Ruta20` en dev. |
| 6 | Nombre de la BD de dev | Si es `RPA`, HU00 usa `RPA.Cajero` y el DDL debe crearse ahí | Confirmar antes de la Fase 2. |
| 7 | `CambioLicencia` y `ActivarTrigger` dependen de IDs del CR de prd y hoy ya fallan en prd | Ruido en dev; en prd bloquea el disparo a Cuadre Integral | En dev desactivar. En prd es un caso aparte (afecta también a Transitoria). |
| 8 | Sistemas externos: SAP, SFTP, NCR, OneDrive/SharePoint | Sin cuentas de QA no se pueden probar HU18, HU04, HU28, HU15B_CargaIndividualDrive | Pedir cuentas de QA en la Fase 0; si no hay, marcar esas HU como "solo se prueban en prd" y documentarlo. |
| 9 | Dos bots tienen su versión más nueva en dev | Si se importa prd sobre dev sin reaplicar, se pierden | Fase 6 lo contempla. |
| 10 | La BD no tiene FK ni NOT NULL | Un dato faltante no lo detecta SQL, lo detecta el bot en runtime | Cargar T1 completa; validar con las consultas de 4.4 antes de correr. |

---

## Anexo A — Extraer y comparar esquema (correr en prd y en dev)

```sql
-- 1. Ejecutar en cada ambiente, guardar el resultado como CSV.
SELECT  t.TABLE_SCHEMA, t.TABLE_NAME, c.COLUMN_NAME, c.ORDINAL_POSITION,
        c.DATA_TYPE, c.CHARACTER_MAXIMUM_LENGTH, c.NUMERIC_PRECISION,
        c.NUMERIC_SCALE, c.IS_NULLABLE, c.COLUMN_DEFAULT
FROM    INFORMATION_SCHEMA.TABLES t
JOIN    INFORMATION_SCHEMA.COLUMNS c
        ON c.TABLE_SCHEMA = t.TABLE_SCHEMA AND c.TABLE_NAME = t.TABLE_NAME
WHERE   t.TABLE_SCHEMA = 'Cajero'
ORDER BY t.TABLE_NAME, c.ORDINAL_POSITION;

-- 2. Objetos que no son tablas (vistas, SPs, índices)
SELECT o.type_desc, s.name AS esquema, o.name, o.modify_date
FROM   sys.objects o JOIN sys.schemas s ON s.schema_id = o.schema_id
WHERE  s.name = 'Cajero' AND o.type IN ('V','P','FN','TF','TR')
ORDER BY o.type_desc, o.name;

SELECT OBJECT_NAME(i.object_id) AS tabla, i.name AS indice, i.type_desc
FROM   sys.indexes i
WHERE  OBJECT_SCHEMA_NAME(i.object_id) = 'Cajero' AND i.name IS NOT NULL;

-- 3. Si los dos CSV se cargan en dev como Cajero_Meta_PRD y Cajero_Meta_DEV:
SELECT 'FALTA_EN_DEV' AS tipo, p.TABLE_NAME, p.COLUMN_NAME, p.DATA_TYPE, p.CHARACTER_MAXIMUM_LENGTH
FROM   Cajero_Meta_PRD p
LEFT JOIN Cajero_Meta_DEV d ON d.TABLE_NAME = p.TABLE_NAME AND d.COLUMN_NAME = p.COLUMN_NAME
WHERE  d.TABLE_NAME IS NULL
UNION ALL
SELECT 'TIPO_DISTINTO', p.TABLE_NAME, p.COLUMN_NAME,
       p.DATA_TYPE + ' vs ' + d.DATA_TYPE,
       ISNULL(p.CHARACTER_MAXIMUM_LENGTH,0)
FROM   Cajero_Meta_PRD p
JOIN   Cajero_Meta_DEV d ON d.TABLE_NAME = p.TABLE_NAME AND d.COLUMN_NAME = p.COLUMN_NAME
WHERE  p.DATA_TYPE <> d.DATA_TYPE
   OR  ISNULL(p.CHARACTER_MAXIMUM_LENGTH,0) <> ISNULL(d.CHARACTER_MAXIMUM_LENGTH,0)
ORDER BY 1, 2, 3;
```

## Anexo B — Comparar dos exports de Automation Anywhere

Los bots exportados son JSON. Un `cmp` byte a byte ya separa idénticos de distintos; para el diff semántico (variables, paquetes, nodos con sus atributos, ignorando renumeración) se usó un script Python que aplana cada bot y hace `difflib.unified_diff`. Guardarlo en el repo como herramienta de promoción: correrlo antes de cada import a prd y adjuntar la salida al control de cambios.

Comparación rápida (Git Bash):

```bash
cd _inbox/escarlet-cuadre-cajeros/export-aa
for f in $(cd produccion && find "Automation Anywhere" -type f ! -name "*.png" | sed 's/ /%20/g'); do
  p=$(echo "$f" | sed 's/%20/ /g')
  cmp -s "desarrollo/$p" "produccion/$p" && echo "IDENTICAL | $p" || echo "DIFF      | $p"
done
```

## Anexo C — Qué no se hizo y por qué

- No se ingirió nada al vault: eso es `/ingest` y deja notas en `_staging/` para revisión humana. Este documento es un plan de trabajo, no conocimiento verificado del proceso.
- No se abrieron los `.zip` (backups del CR), los videos ni la carpeta `dml-*`, por regla del proyecto.
- No se leyó el cuerpo de la especificación V2.4 HU por HU; solo su índice, para mapear HUs contra bots. La lectura completa entra por `/ingest`.

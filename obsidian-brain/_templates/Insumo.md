---
tipo: Insumo
id: INS-XXX
source_doc: "[[Documento - nombre-del-documento]]"
confidence: media
estado: propuesto
last_verified_date: YYYY-MM-DD
---

# INS-XXX — Nombre del insumo

Archivo o fuente de entrada que el proceso consume.

- **Formato:** xlsx / csv / txt / pdf / cola / tabla / correo
- **Origen:** quién o qué lo genera ([[Actor - ...]] o [[Sistema - ...]])
- **Ubicación:** carpeta de red, buzón, SFTP, tabla — según el documento
- **Frecuencia:** diaria / mensual / por demanda
- **Estructura esperada:** columnas o campos clave

## Consumido por
- [[PasoDeProceso - nombre-paso]]

## Depende de
- [[Sistema - nombre-sistema]]

## Extraído de
- [[Documento - nombre-del-documento]]

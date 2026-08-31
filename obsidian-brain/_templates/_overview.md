---
tipo: ResumenEjecutivo
id: OVW-001
source_doc: "[[Documento - nombre-del-documento]]"
confidence: media
estado: propuesto
last_verified_date: YYYY-MM-DD
---

# <Nombre del proyecto> — resumen ejecutivo

> Esta nota es el punto de entrada del proyecto: la lee alguien que llega
> hoy y tiene que entender el proceso en 5 minutos. La regenera
> `graph-writer-agent` cada vez que escribe al vault. No es un índice de
> nodos — es un resumen. Si una sección no se puede llenar con lo que hay
> en el vault, dejarla escrita como "sin información en el vault" y
> anotarla en [[Pendientes - nombre-proyecto]].

## En una frase

Qué hace este proceso y para quién.

## Por qué existe

El problema de negocio que resuelve.

## Cómo funciona, a grandes rasgos

3 a 6 viñetas con el flujo de punta a punta. Cada una enlaza al paso real.

- [[PasoDeProceso - primer-paso]] — ...

## Qué toca

- **Sistemas:** [[Sistema - ...]]
- **Ambientes:** [[Ambiente - ...]]
- **Insumos:** [[Insumo - ...]]
- **Salidas:** [[Salida - ...]]
- **Bots:** [[Robot - ...]]

## Quién lo opera

- [[Actor - ...]] — qué responsabilidad tiene

## Estado de salud

- **Incidencias abiertas:** N ([[Incidencia - ...]] los stopper primero)
- **Riesgos altos:** N
- **Inconsistencias sin resolver:** N
- **Cobertura de documentación:** ver [[Pendientes - nombre-proyecto]]

## Por dónde seguir

- Contexto y alcance → [[Proyecto - nombre-proyecto]]
- Arquitectura técnica → [[Arquitectura - nombre-proyecto]]
- Glosario → carpeta `glosario/`
- Preguntas frecuentes → carpeta `faq/`

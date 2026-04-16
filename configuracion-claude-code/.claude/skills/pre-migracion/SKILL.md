---
name: pre-migracion
description: Analiza un servicio legacy IIB y genera ANALISIS_<ServiceName>.md con cuantificacion completa y score de confianza
allowed-tools: Read Glob Grep Bash Agent
---

# /pre-migracion <ruta_al_servicio_legacy>

Ejecuta el analisis completo de un servicio legacy IBM IIB para preparar su migracion.

## Pasos

1. **Localizar artefactos** en la ruta proporcionada ($ARGUMENTS):
   - Buscar *.esql, *.wsdl, *.xsd, *.msgflow, *.subflow, pom.xml

2. **Lanzar agente analista-legacy** para analizar todos los artefactos

3. **Generar `ANALISIS_<ServiceName>.md`** con:
   - Descripcion general del servicio
   - Endpoints expuestos (contrato SOAP completo)
   - Servicios downstream (UMPs) con mapeo a TX BANCS
   - Tabla de cuantificacion (operaciones, UMPs, errores, campos, configs)
   - Logica de negocio paso a paso
   - Mapa de propagacion de errores
   - Clasificacion BUS (WebFlux) vs WAS (MVC)
   - Score de confianza
   - Incertidumbres y supuestos

4. **Mostrar resumen** al usuario con:
   - Nombre del servicio
   - Modo (BUS/WAS)
   - Cantidad de UMPs
   - Score de confianza
   - Recomendacion GO/NO-GO

## Ejemplo de uso
```
/pre-migracion C:\Dev\Banco Pichincha\CapaMedia\_extracted\sqb-msa-wsclientes0007
```

---
name: post-migracion
description: Escanea un proyecto migrado, documenta stubs/pendientes, genera guia de primera ejecucion y casos de uso QA
allowed-tools: Read Glob Grep Bash Write Agent
---

# /post-migracion

Genera la documentacion de todo lo que falta para conectar el servicio al banco.

## Prerequisitos
- Proyecto Java migrado en el directorio actual
- ANALISIS_<ServiceName>.md disponible para cross-reference

## Pasos

1. **Escanear stubs y TODOs**
   - Buscar en *.java: TODO, FIXME, TBD, UnsupportedOperationException, pendiente_validar
   - Para cada stub: generar tarjeta de remediacion con UMP legacy, campos, formato CIF, patron de implementacion

2. **Auditar variables de entorno**
   - Cross-referenciar ${CCC_*} entre application.yml y helm/*.yml
   - Flaggear variables faltantes o huerfanas

3. **Inventariar credenciales**
   - ARTIFACT_USERNAME/TOKEN, SONAR_TOKEN, etc.
   - Documentar como obtener cada una

4. **Generar guia de primera ejecucion**
   - 8-11 pasos: desde configurar credenciales hasta smoke test SOAP
   - Incluir template .env.local

5. **Lanzar agente qa-generator** para generar casos de uso QA
   - Formato BDD con XML completo
   - Categorias: exito, validacion, backend, failover, error tecnico

6. **Calcular score de preparacion**
   - 12 aspectos ponderados
   - Recomendacion: listo / casi listo / en progreso / necesita atencion

7. **Generar PENDIENTES_<ServiceName>.md**

## Ejemplo de uso
```
/post-migracion
```
(ejecutar desde la raiz del proyecto migrado)

---
name: post-migracion
description: Audita un proyecto Java migrado contra la checklist BPTPSRE y genera reporte pass/fail por bloque con severidad y accion sugerida
allowed-tools: Read Glob Grep Bash Write Agent
---

# /post-migracion

Ejecuta la auditoria de calidad post-migracion contra las reglas vivas del equipo BPTPSRE de Banco Pichincha.

**NO modifica codigo.** Solo audita y reporta. Los fixes se hacen en un flujo aparte.

## Prerequisitos
- Proyecto Java migrado en el directorio actual
- ANALISIS_<ServiceName>.md disponible para cross-reference
- Acceso a `prompts/post-migracion/03-checklist.md` (cargado via CLAUDE.md)

## Cuando se usa
- Despues de correr `/migrar` y antes de abrir PR
- Para auditar servicios que ya estan en `develop` antes de pasar a `release`

## Input
Path absoluto al proyecto migrado (ej: `C:\Dev\Banco Pichincha\CapaMedia\0007\destino`).

## Pasos

1. **Bloque 0: Pre-check** — Detectar tipo de proyecto (REST/SOAP/MVC) y gold standard aplicable
   - REST -> referencia `tnd-msa-sp-wsclientes0024`
   - SOAP -> referencia `tnd-msa-sp-wsclientes0015`

2. **Ejecutar la checklist** (`prompts/post-migracion/03-checklist.md`) bloque por bloque
   - Cada bloque referencia su origen: PDF oficial, feedback Jean Pierre Garcia, commits especificos, MCP fabrics
   - Para cada regla: pass/fail con evidencia (archivo + linea)
   - Severidad por hallazgo: HIGH / MEDIUM / LOW
   - Accion sugerida concreta para cada fail

3. **Generar reporte estructurado** `CHECKLIST_<ServiceName>.md` con:
   - Resumen ejecutivo (pass/fail totales por bloque)
   - Detalle por bloque con violaciones encontradas
   - Tabla de severidad agregada
   - Lista priorizada de fixes (HIGH primero)
   - Recomendacion: APTO PARA PR / REQUIERE FIXES

## Ejemplo de uso
```
/post-migracion
```
(ejecutar desde la raiz del proyecto migrado)

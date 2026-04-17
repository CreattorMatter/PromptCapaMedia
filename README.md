# PromptCapaMedia

Toolkit de prompts y configuración para la migración de servicios legacy del Banco Pichincha (IIB / WAS / ORQ) a Java 21 + Spring Boot + arquitectura hexagonal OLA1.

## Estructura

```
prompts/
├── pre-migracion/
│   ├── 01-analisis-servicio.md      # IIB y WAS — análisis exhaustivo
│   └── 01-analisis-orq.md           # ORQ — análisis liviano (delegación)
├── migracion/
│   ├── REST/02-REST-migrar-servicio.md   # WSDL 1 op + sin BD
│   └── SOAP/02-SOAP-migrar-servicio.md   # WSDL 2+ ops, o WAS con BD
├── post-migracion/
│   └── 03-checklist.md              # Auditoría pass/fail con severidad
├── configuracion-claude-code/       # CLAUDE.md, skills, agents, rules
│   └── sonarlint/                   # Guía + template de SonarLint local
├── documentacion/                   # PDFs internos del banco
├── ejemplos/                        # Gold standards (zips de referencia)
├── sqb-cfg-codigosBackend-config/   # XMLs de catálogo backend
└── sqb-cfg-errores-errors/          # XMLs de catálogo errores
```

## Flujo (3 etapas)

1. **`/pre-migracion <ruta>`** — detecta tipo (IIB / WAS / ORQ) y genera `ANALISIS_*.md`
2. **`/migrar`** — implementa el microservicio
   - WSDL 1 op + sin BD → REST (WebFlux + `@RestController`)
   - WSDL 2+ ops, o WAS con BD → SOAP (Spring WS + `@Endpoint`, MVC con HikariCP+JPA si aplica)
3. **`/post-migracion`** — audita el proyecto migrado contra la checklist

## Setup recomendado

- **Claude Code / Windsurf / Copilot:** copiar `configuracion-claude-code/.claude/` a la raíz del proyecto migrado.
- **SonarLint local:** seguir `configuracion-claude-code/sonarlint/README.md` para bindar el repo migrado a SonarCloud (`bancopichinchaec`) y obtener feedback en tiempo real en el IDE.
- **Secrets:** nunca commitear; referenciar como `${CCC_*}` env vars. El banco provee valores reales ~1 semana antes del deploy.

## Scaffold inicial

El **Fabrics MCP del Banco Pichincha** genera el scaffold inicial via cuestionario (operación count, BD, etc.). Los prompts de migración asumen que partís desde ese scaffold.

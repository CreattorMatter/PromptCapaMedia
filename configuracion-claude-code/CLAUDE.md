# Capa Media OLA1 - Banco Pichincha

## Proyecto
Migracion de servicios legacy a Java 21 + Spring Boot + arquitectura hexagonal OLA1.

**Fuentes legacy soportadas:**
- **IIB** (IBM Integration Bus) — ESQL + SOAP/WSDL + msgflow + MQ
- **WAS** (WebSphere Application Server) — Java/JAX-WS + típicamente Oracle
- **ORQ** (Orquestadores) — IIB orchestrators que delegan a otros servicios; análisis liviano

**Regla de split REST/SOAP (igual para los 3 tipos):**
- WSDL con 1 operación + sin BD -> REST (WebFlux + @RestController)
- WSDL con 2+ operaciones, o WAS con BD -> SOAP (Spring WS + @Endpoint, MVC con HikariCP+JPA si aplica)

**Scaffold inicial:** lo genera el **Fabrics MCP del Banco Pichincha** vía cuestionario. La migración parte desde ese scaffold; no se reconstruye desde cero.

**Secrets:** NUNCA buscar/inventar. Referenciar como `${CCC_*}` env vars en `application.yml` y `helm/*.yml`. El banco provee valores reales ~1 semana antes del deploy productivo.

## Build & Test
```bash
# bloque_estricto_a_copiar
./gradlew generateFromWsdl      # Generar clases JAXB desde WSDL
./gradlew clean build            # Build completo con tests
./gradlew test                   # Solo tests
./gradlew jacocoTestReport       # Reporte de cobertura
./gradlew bootJar                # Generar JAR para Docker
```

## Arquitectura: Hexagonal OLA1
```
application/
  port/input/    <- abstract classes (NUNCA interfaces)
  port/output/   <- abstract classes (NUNCA interfaces)
  service/       <- @Service @RequiredArgsConstructor @Slf4j
domain/
  model/         <- Records puros, CERO imports de Spring
  exception/     <- Excepciones tipadas
infrastructure/
  input/adapter/ <- SOAP controller, DTOs envelope
  output/adapter/<- BANCS adapters via Core Adapter REST
  config/        <- @Configuration, @ConfigurationProperties
  mapper/        <- MapStruct o @Component mappers
  exception/     <- ErrorResolverHandler
```

## Reglas criticas (NUNCA violar)
- Ports son ABSTRACT CLASSES, nunca interfaces
- domain/ no importa Spring, SOAP, JPA, WebFlux
- application/ no importa infrastructure/
- CERO @Autowired — solo @RequiredArgsConstructor
- Metodos max 20 lineas, lineas max 100 columnas
- @Slf4j en todas las clases con comportamiento
- HTTP 200 para errores de negocio (compatibilidad IIB)
- HTTP 500 solo para SOAP Faults inesperados
- Todo el codigo en INGLES
- Config via ${CCC_*} env vars, NUNCA hardcodear
- livenessProbe + readinessProbe en TODOS los Helm values
- Produccion: replicaCount >= 2, hpa.enabled: true

## Referencia de patrones
Usar tnd-msa-sp-wsclientes0024 como proyecto de referencia para copiar patrones exactos.

## Flujo de trabajo
1. `/pre-migracion <ruta>` — Detecta tipo (IIB / WAS / ORQ) y genera ANALISIS_*.md
   - IIB o WAS -> usa `pre-migracion/01-analisis-servicio.md`
   - ORQ (orquestador) -> usa `pre-migracion/01-analisis-orq.md` (análisis liviano)
2. `/migrar` — Ejecuta migracion con autocorreccion
   - WSDL con 1 operacion + sin BD -> usa `migracion/REST/02-REST-migrar-servicio.md`
   - WSDL con 2+ operaciones, o WAS con BD -> usa `migracion/SOAP/02-SOAP-migrar-servicio.md`
3. `/post-migracion` — Audita el proyecto migrado contra la checklist (`post-migracion/03-checklist.md`), genera reporte pass/fail por bloque (incluye BLOQUE 13 si hay JPA/HikariCP)

## Commits
Conventional Commits: `feat|fix|refactor|test|docs|chore|ci|iac: descripcion`

@prompts/pre-migracion/01-analisis-servicio.md
@prompts/pre-migracion/01-analisis-orq.md
@prompts/migracion/REST/02-REST-migrar-servicio.md
@prompts/migracion/SOAP/02-SOAP-migrar-servicio.md
@prompts/post-migracion/03-checklist.md

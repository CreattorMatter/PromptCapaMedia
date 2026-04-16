# Capa Media OLA1 - Banco Pichincha

## Proyecto
Migracion de servicios legacy IBM IIB / WAS a Java 21 + Spring Boot + arquitectura hexagonal OLA1.

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
1. `/pre-migracion <ruta>` — Analiza legacy, genera ANALISIS_*.md
2. `/migrar` — Ejecuta migracion con autocorreccion
3. `/post-migracion` — Genera PENDIENTES_*.md

## Commits
Conventional Commits: `feat|fix|refactor|test|docs|chore|ci|iac: descripcion`

@prompts/pre-migracion/01-analisis-servicio.md
@prompts/migracion/02-migrar-servicio.md
@prompts/post-migracion/03-preparar-integracion.md

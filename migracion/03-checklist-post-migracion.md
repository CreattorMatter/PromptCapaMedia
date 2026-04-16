# 03 — Checklist Post-Migración (AI-ejecutable)

## ROL

Sos un auditor de código que revisa un microservicio Java Spring Boot ya migrado (IIB → Java, arquitectura hexagonal OLA1) contra las reglas vivas del equipo **BPTPSRE** de Banco Pichincha. Tu salida es un **reporte estructurado** con pass/fail por bloque, severidad, y acción sugerida por cada violación.

**NO modificas código.** Solo auditás y reportás. Los fixes se hacen en un flujo aparte.

## CUÁNDO SE USA

Después de correr el prompt de migración (`02-migrar-servicio.md`) y antes de abrir PR. También sirve para auditar servicios que ya están en `develop` antes de pasar a `release`.

## INPUT

Un único argumento: **path absoluto al proyecto migrado** (ej: `C:\Dev\Banco Pichincha\CapaMedia\0007\destino`).

## FUENTES DE LAS REGLAS

Cada bloque referencia su origen para que el lector sepa de dónde viene:

- **[PDF-OFICIAL]** — `prompts/documentacion/BPTPSRE-CheckList Desarrollo-140426-212740.pdf` (documento oficial del área)
- **[FB-JG]** — Feedback de Jean Pierre García (Slack, múltiples fechas)
- **[COMMIT-XXXXX]** — Commit específico del repo de referencia
- **[MCP]** — Confluence del MCP fabrics (versiones de librerías)

---

## BLOQUE 0 — Pre-check: identificar tipo de proyecto y gold standard

Antes de auditar, detectar qué tipo de servicio es y qué gold standard le corresponde.

### Check 0.1 — Tipo de proyecto (SOAP vs REST)

```bash
# SOAP: existe @Endpoint o WebServiceConfig
grep -rl "@Endpoint\|WebServiceConfig" <PATH>/src/main/java

# REST: existe @RestController y NO @Endpoint
grep -rl "@RestController" <PATH>/src/main/java
```

**Decisión:**
- Hay `@Endpoint` → **SOAP** (gold standard: `tnd-msa-sp-wsclientes0015`)
- Hay `@RestController` y no `@Endpoint` → **REST** (gold standard: `tnd-msa-sp-wsclientes0024`, FROZEN 2026-04-14)
- Ambos o ninguno → **FAIL HIGH** (proyecto malformado)

### Check 0.2 — WSDL: cantidad de operaciones

```bash
grep -c "<wsdl:operation\|<operation " <PATH>/src/main/resources/wsdl/*.wsdl
```

- 1 operación y tipo = REST → ✓
- 2+ operaciones y tipo = SOAP → ✓
- Mismatch → **FAIL HIGH** (usar el tipo incorrecto)

**Guardar en el contexto del reporte:** `projectType`, `goldStandard`.

---

## BLOQUE 1 — Arquitectura hexagonal

**Origen:** [PDF-OFICIAL] (sección Arquitectura) + [FB-JG] (UN solo output port Bancs)

### Check 1.1 — Capas presentes

```bash
ls <PATH>/src/main/java/com/pichincha/sp/
# Debe contener: application/ domain/ infrastructure/
```

**Severidad si falta alguna:** HIGH.

### Check 1.2 — Domain SIN imports de framework

```bash
grep -rE "import (org\.springframework|jakarta\.persistence|org\.springframework\.web|org\.springframework\.http|javax\.ws)" <PATH>/src/main/java/com/pichincha/sp/domain/
```

Esperado: **0 matches**. Cualquier hit es **HIGH**.

### Check 1.3 — Puertos son interfaces

```bash
grep -l "public abstract class.*Port\|public abstract class.*InputPort\|public abstract class.*OutputPort" <PATH>/src/main/java/com/pichincha/sp/application/
```

Esperado: **0 archivos**. Abstract classes para puertos es desviación (caso wsclientes0007 original). **HIGH**.

### Check 1.4 — UN SOLO output port Bancs [FB-JG]

```bash
ls <PATH>/src/main/java/com/pichincha/sp/application/*/port/output/ | grep -iE "bancs|Bancs"
```

- 0 ports Bancs → ⚠ MEDIUM (puede ser intencional si el servicio no llama a Bancs)
- 1 port Bancs → ✓ PASS
- 2+ ports Bancs → **HIGH**. Unificar en un solo port con múltiples métodos.

**Acción sugerida:** consolidar métodos en un único `BancsCustomerOutputPort` (o nombre equivalente al dominio del servicio). La pluralidad de adapters/helpers a nivel infrastructure sigue siendo válida; lo que se unifica es el puerto visible desde application.

**Referencia:** wsclientes0015 tiene 3 ports Bancs (`CustomerAddressBancsPort`, `GeoLocationBancsPort`, `CorrespondenceBancsPort`) — **NO replicar**. wsclientes0024 y wsclientes0007 cumplen.

### Check 1.5 — Adaptadores implementan puertos (no abstract class)

```bash
grep -l "extends.*Port\s" <PATH>/src/main/java/com/pichincha/sp/infrastructure/
```

Esperado: **0 archivos** (deben usar `implements` sobre interfaces). **MEDIUM**.

---

## BLOQUE 2 — Logging y tracing

**Origen:** [PDF-OFICIAL] (sección Log) + [FB-JG] (@BpLogger en TODOS los services)

### Check 2.1 — `@BpTraceable` en Controllers

```bash
grep -L "@BpTraceable" <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/*/impl/*Controller.java
```

Cualquier archivo listado (sin la anotación) es **HIGH**.

### Check 2.2 — `@BpLogger` en TODOS los métodos públicos de @Service [FB-JG]

```bash
# Para cada Service bean, contar métodos públicos vs @BpLogger
for f in <PATH>/src/main/java/com/pichincha/sp/application/service/*.java; do
  pub=$(grep -cE "^\s+public\s+\w+\s+\w+\(" "$f")
  bpl=$(grep -c "@BpLogger" "$f")
  [ "$pub" -gt "$bpl" ] && echo "FAIL: $f (public=$pub, @BpLogger=$bpl)"
done
```

Si `public > @BpLogger` en cualquier service → **HIGH**. Cada método público debe tener `@BpLogger`.

**Referencia:** wsclientes0015 viola esto (2 de 3 services sin la anotación) — **NO replicar**.

### Check 2.3 — `@BpLogger` en Adapters de infrastructure

```bash
grep -L "@BpLogger" <PATH>/src/main/java/com/pichincha/sp/infrastructure/output/adapter/**/*Adapter.java
```

Archivos listados → **MEDIUM** (ideal que lo tengan; no bloqueante si el método es trivial).

### Check 2.4 — Logs con GUID y niveles adecuados

```bash
# Buscar logs sin GUID
grep -rnE "log\.(info|warn|error)\(\"[^{]*\"\)" <PATH>/src/main/java/com/pichincha/sp/application/
```

**MEDIUM** si hay logs que no incluyen `guid` ni contexto. Los logs deben tener formato `[guid: {}] <mensaje>` donde aplique.

### Check 2.5 — Sin imports de `org.slf4j` [FB-JG]

```bash
grep -rnE "import org\.slf4j\." <PATH>/src/main/java/
```

Cualquier match → **HIGH**. El proyecto debe usar exclusivamente la librería de logging del banco (`ServiceLogHelper`, `@BpLogger`, `@BpTraceable`). Los imports directos de `org.slf4j.Logger`, `org.slf4j.LoggerFactory` o la anotación Lombok `@Slf4j` están prohibidos porque duplican el logging y no se integran con la trazabilidad corporativa.

**Patrón incorrecto (detectado en wsclientes0007):**
```java
import org.slf4j.Logger;           // ← PROHIBIDO
import org.slf4j.LoggerFactory;    // ← PROHIBIDO

private static final Logger LOGGER =
    LoggerFactory.getLogger(MyController.class);  // ← PROHIBIDO

LOGGER.warn("algo");  // duplica el log de ServiceLogHelper
log.warn("algo");     // este es el correcto (ServiceLogHelper)
```

**Acción:** eliminar imports `org.slf4j.*`, campos `Logger`/`LoggerFactory`, y todas las llamadas `LOGGER.xxx(...)`. Quedarse solo con `ServiceLogHelper log` inyectado.

### Check 2.6 — No abuso de log.info

```bash
grep -rc "log\.info\|logLevelHandler.log(CustomLogLevel.INFO" <PATH>/src/main/java/ | awk -F: '$2 > 5 {print}'
```

Archivos con más de 5 log.info → **LOW** (revisar si son necesarios o son "logs de navegación").

---

## BLOQUE 3 — Naming de métodos y convenciones

**Origen:** [FB-JG] (queryX → getX) + [COMMIT-bf913b9] (PascalCase localPart)

### Check 3.1 — Output ports usan `get*` para lecturas [FB-JG]

```bash
grep -rnE "^\s*\w+\s+query\w+\(" <PATH>/src/main/java/com/pichincha/sp/application/*/port/output/
```

Cualquier método `query*` en puertos de lectura → **HIGH**.

**Regla:**
- Lecturas → `get*` (ej: `getCustomerInfo`, `getTransactionalContact`)
- Mutaciones → `create*` / `update*` / `delete*`
- `query*` **PROHIBIDO** para lecturas.

### Check 3.2 — `@PayloadRoot.localPart` = PascalCase [COMMIT-bf913b9] (solo SOAP)

```bash
grep -nE "localPart\s*=\s*\"[a-z]" <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/soap/impl/*Controller.java
```

Cualquier `localPart` que empiece en minúscula → **HIGH**. Debe matchear EXACTO el nombre del elemento raíz del XSD (PascalCase en servicios IIB legacy).

**Ejemplo correcto:**
```java
@PayloadRoot(namespace = NAMESPACE_URI,
    localPart = "ConsultarContactoTransaccional01")  // PascalCase
public ConsultarContactoTransaccional01Response
    consultarContactoTransaccional01(...) { ... }     // método Java: camelCase
```

### Check 3.3 — Java method name = camelCase

```bash
grep -nE "public\s+[A-Z]\w+Response\s+[A-Z]\w+\(" <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/*/impl/*Controller.java
```

Método en PascalCase → **MEDIUM** (rompe convención Java).

### Check 3.4 — `postProcessWsdl.groovy` sin decapitalize [COMMIT-bf913b9] (solo SOAP)

```bash
grep -nE "decapitalize|injectXmlRootElement|updatePackageInfo" <PATH>/gradle/postProcessWsdl.groovy
```

Cualquier match → **HIGH**. Revertido en commit `bf913b9` del 0007 (2026-04-14).

---

## BLOQUE 4 — Validaciones

**Origen:** [FB-JG] (validaciones de header en infrastructure)

### Check 4.1 — `HeaderRequestValidator` existe en infrastructure [FB-JG]

```bash
find <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/*/util/ -name "HeaderRequestValidator.java"
```

- 0 archivos → **HIGH**. Falta el validador de header.
- 1 archivo en la ubicación correcta → ✓ PASS
- Existe en `domain/` o `application/` → **HIGH** (ubicación incorrecta)

### Check 4.2 — HeaderRequestValidator NO está en domain/application

```bash
find <PATH>/src/main/java/com/pichincha/sp/domain/ <PATH>/src/main/java/com/pichincha/sp/application/ -name "HeaderRequestValidator.java" -o -name "*HeaderValidator*.java"
```

Cualquier archivo encontrado → **HIGH**.

### Check 4.3 — Controller invoca al validator

```bash
grep -rn "HeaderRequestValidator\." <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/*/impl/*Controller.java
```

0 matches → **MEDIUM** (existe el validator pero no se usa).

### Check 4.4 — Validaciones de body/business siguen en domain/application

Body validations (ej: CIF vacío, identificación inválida) **SÍ van en application/service** con `BusinessValidationException`. Este check es informativo — no es violación.

---

## BLOQUE 5 — Error handling y propagación de errores Bancs

**Origen:** [FB-JG] (error propagation) + [FB-JG] (localPart casing ya en bloque 3)

### Check 5.1 — `BancsClientHelper.execute()` atrapa RuntimeException [FB-JG]

```bash
grep -A5 "public <T> T execute" <PATH>/src/main/java/com/pichincha/sp/infrastructure/output/adapter/bancs/helper/BancsClientHelper.java | grep -c "catch\s*(\s*RuntimeException"
```

0 matches → **HIGH**. Sin este catch, `WebClientResponseException` burbujea como RuntimeException y el Controller devuelve "999 Error interno" en vez del error real de Bancs.

**Patrón requerido:**
```java
try {
  response = doCall(txCode, ctx, body, responseType);
} catch (BancsOperationException e) {
  throw e;
} catch (RuntimeException e) {
  throw new BancsOperationException("999",
    e.getMessage() != null ? e.getMessage() : "Bancs integration exception",
    txCode);
}
```

**Referencia:** wsclientes0015 NO lo tiene (hueco del gold standard). wsclientes0007 lo cubrió post-feedback de Jean Pierre.

### Check 5.2 — Controller/Service atrapa BancsOperationException

```bash
grep -rn "catch\s*(\s*BancsOperationException" <PATH>/src/main/java/com/pichincha/sp/
```

0 matches → **HIGH**. Sin catch, el error nunca llega al cliente SOAP/REST con código estructurado.

**Ubicación válida:** Controller (patrón 0007) o Service (patrón 0015). Uno u otro, no ambos.

### Check 5.3 — No hay SOAP Faults — todo es HTTP 200 con `<error>` (solo SOAP)

```bash
grep -rn "SoapFaultException\|MessageFaultException\|throw.*SoapFault" <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/soap/
```

Cualquier match → **HIGH**. Errores se devuelven como response válido con `tipo='ERROR'`, nunca como SOAP fault (compatibilidad IIB).

### Check 5.4 — BusinessValidationException en domain

```bash
grep -l "BusinessValidationException" <PATH>/src/main/java/com/pichincha/sp/domain/exception/
```

0 matches → **HIGH**. Debe existir en `domain/exception/`.

---

## BLOQUE 6 — Mappers y tipos

**Origen:** [FB-JG] (usar mappers, evitar genéricos)

### Check 6.1 — Mappers dedicados existen

```bash
find <PATH>/src/main/java -type d -name "mapper"
```

Debe haber al menos:
- `infrastructure/input/adapter/*/mapper/` (request SOAP/REST → domain)
- `infrastructure/output/adapter/bancs/mapper/` (Bancs DTO → domain)

0 o 1 carpeta → **MEDIUM**.

### Check 6.2 — MapStruct en mappers

```bash
grep -rn "@Mapper" <PATH>/src/main/java/com/pichincha/sp/infrastructure/*/mapper/
```

0 matches → **MEDIUM**. Ideal usar MapStruct (o justificar por qué no).

### Check 6.3 — Sin `new Record(>=8 args)` inline en services [FB-JG]

```bash
grep -rnE "new \w+\([^;]{200,}" <PATH>/src/main/java/com/pichincha/sp/application/service/*.java
```

Cualquier match → **MEDIUM**. Si un service construye un record con muchos argumentos, extraer a factory estático en el record (ej: `Result.success(...)`, `Result.failure(...)`) como hace wsclientes0015 con `ConsultAddressesResult`.

### Check 6.4 — Sin `Object`/`Map<String, Object>` en contratos de puertos [FB-JG]

```bash
grep -rnE "\b(Object|Map<String,\s*Object>|\?\s*>)\b" <PATH>/src/main/java/com/pichincha/sp/application/*/port/
```

Cualquier match → **HIGH**. Excepciones justificadas (deben documentarse):
- Helpers de infra con binding genérico `<T>` (ej: `BancsClientHelper.execute(..., Object body, Class<T>)`)
- Logging varargs (`Object... data`)

Si el match es en un puerto de application → siempre **HIGH**.

---

## BLOQUE 7 — Configuración externa

**Origen:** [PDF-OFICIAL] (Application.yml, Azure pipeline, Helm) + [FB-JG] (catalog-info.yaml detallado)

### Check 7.1 — `catalog-info.yaml` completo [FB-JG]

Verificar:
- `spec.owner: jgarcia@pichincha.com` (NO `<owner>`)
- `spec.lifecycle: test` (NO `<lifecycle>`)
- `spec.system: ""` (vacío)
- `spec.domain: ""` (vacío)
- `spec.definition:` **ELIMINADO** (no debe existir el bloque)
- `links[]` NO incluye URL de SwaggerHub
- `annotations.dynatrace.com/dynatrace-entity-id: ""` (vacío)
- `annotations.sonarcloud.io/project-key` con el ID real del proyecto Sonar
- `spec.dependsOn` lista librerías del banco (ej: `lib-bnc-api-client`), NO `frm-spa-optimus-core`
- `tags` BIAN pendientes de confirmación (comentadas o placeholder explícito)

```bash
# Quick scan:
grep -E "<owner>|<lifecycle>|<domain>|<system>|swaggerhub\.com|definition:" <PATH>/catalog-info.yaml
```

Cualquier match → **HIGH** (placeholders sin completar).

### Check 7.2 — `azure-pipelines.yml` alineado [PDF-OFICIAL]

```bash
grep -E "KUBERNETES_NAMESPACE|CMDB_APPLICATION_ID" <PATH>/azure-pipelines.yml
```

Verificar:
- `KUBERNETES_NAMESPACE` = namespace correcto del microservicio (ej: `tnd-middleware`)
- `CMDB_APPLICATION_ID` = `"Red Hat OpenShift Container Platform"` (NO `CAPA_COMUN`)

Mismatch → **HIGH**.

### Check 7.3 — `application.yml` usa variables ENV del Helm [PDF-OFICIAL]

```bash
grep -nE "^\s*(password|secret|url|host|user):\s*[^$]" <PATH>/src/main/resources/application*.yml
```

Cualquier valor hardcoded para secrets/urls → **HIGH**. Deben ser `${VAR_NAME:default}`.

### Check 7.4 — Validar configuraciones de Bancs, WebClient, Circuit Breaker [PDF-OFICIAL]

```bash
grep -E "bancs:|resilience4j:|circuitbreaker:" <PATH>/src/main/resources/application*.yml
```

Secciones esperadas: `bancs:` y `resilience4j:`. Faltante → **MEDIUM**.

**NOTA (patrón del banco):** el webclient connector no va en una sección `webclient:` top-level — vive **dentro** de cada `bancs.webclients.<ws-txNNNNNN>.connector.*`. Verificar que cada webclient tiene `connect-timeout`, `read-timeout`, `max-connections`, `max-idle-time`, `pending-acquire-*` dentro de su nodo. Si NO tiene el `connector:` anidado → **MEDIUM**.

### Check 7.5 — Helm con valores por entorno [PDF-OFICIAL]

```bash
ls <PATH>/helm/dev.yml <PATH>/helm/test.yml <PATH>/helm/prod.yml 2>/dev/null
```

**NOTA (patrón del banco):** el naming oficial es `dev.yml`, `test.yml`, `prod.yml` (NO el convention genérico `values-<env>.yaml` de Helm upstream). Debe existir uno por cada entorno soportado.

Falta algún entorno → **MEDIUM**.

### Check 7.6 — `@ConfigurationPropertiesScan` en Application.java

```bash
grep "@ConfigurationPropertiesScan" <PATH>/src/main/java/com/pichincha/sp/Application.java
```

0 matches → **HIGH** (los beans `@ConfigurationProperties` no se registran sin esto).

---

## BLOQUE 8 — Versiones y dependencias

**Origen:** [PDF-OFICIAL] (Snyk) + [MCP] (versiones)

### Check 8.1 — Versiones actualizadas

```bash
grep -E "springframework.boot.*version|jackson-core:|logstash-logback-encoder|lib-bnc-api-client|peer-review" <PATH>/build.gradle
```

Baseline esperado (a 2026-04):
- Spring Boot: `3.5.13`
- jackson-core / jackson-dataformat-xml: `2.21.2`
- logstash-logback-encoder: `9.0` (solo si aplica)
- lib-bnc-api-client: **verificar en Confluence MCP antes de auditar** — si el valor local es más viejo que el publicado, **MEDIUM**
- Peer Review plugin: `1.1.0`

Cualquier versión menor al baseline → **MEDIUM**.

### Check 8.2 — Snyk sin vulnerabilidades HIGH [PDF-OFICIAL]

Ejecutar (si está disponible):
```bash
cd <PATH> && snyk test --severity-threshold=high
```

Cualquier HIGH → **HIGH** bloqueante. MEDIUM/LOW son informativos.

### Check 8.3 — Peer Review score ≥ 7

```bash
find <PATH>/build/reports -name "peer-review*" -type f
# Leer el reporte y extraer score
```

Score < 7 → **HIGH**. Score 7-8 → MEDIUM (aceptable pero mejorar). Score ≥ 9 → PASS.

### Check 8.4 — Lombok minimal

```bash
grep -rE "@Data|@AllArgsConstructor|@NoArgsConstructor|@Builder|@ToString" <PATH>/src/main/java/
```

Lombok permitido: `@Getter`, `@RequiredArgsConstructor`, `@Setter` (solo en `@ConfigurationProperties`). `@Slf4j` **PROHIBIDO** (genera `org.slf4j.Logger`; usar `ServiceLogHelper` del banco — ver Check 2.5).

Uso de `@Data`/`@AllArgsConstructor`/`@NoArgsConstructor` → **MEDIUM**. Usar records de dominio en vez de clases con Lombok.

---

## BLOQUE 9 — Tests y calidad

**Origen:** [PDF-OFICIAL] (Cobertura 75%, SonarLint, Sonar Test)

### Check 9.1 — JaCoCo ≥ 75% [PDF-OFICIAL]

```bash
cat <PATH>/build/reports/jacoco/test/jacocoTestReport.xml | \
  grep -oE 'counter type="INSTRUCTION" missed="[0-9]+" covered="[0-9]+"' | head -1
# Calcular covered / (covered + missed) * 100
```

- < 75% → **HIGH** (bloqueante del banco)
- 75-80% → MEDIUM
- ≥ 80% → PASS

### Check 9.2 — Build verde

```bash
cd <PATH> && ./gradlew build --no-daemon
```

BUILD FAILED → **HIGH**. Bloqueante.

### Check 9.3 — Tests unitarios presentes por capa

```bash
find <PATH>/src/test/java -name "*Test.java" | wc -l
```

Esperado mínimo:
- 1 test por service de application
- 1 test por adapter de infrastructure
- 1 test de Controller (integration)
- Tests de strategies (si aplica — ver Bloque 11)

< 5 archivos de test → **HIGH**. 5-10 → MEDIUM.

### Check 9.4 — @Nested con @SuppressWarnings("java:S2187")

```bash
grep -rlE "@Nested" <PATH>/src/test/java | while read f; do
  # Si el archivo tiene @Nested pero no @SuppressWarnings("java:S2187")
  grep -L "@SuppressWarnings(\"java:S2187\")" "$f"
done
```

Archivos listados → **LOW** (regla Sonar, no bloqueante).

### Check 9.5 — SonarLint local sin issues BLOCKER/CRITICAL [PDF-OFICIAL]

```bash
# Si hay reporte de SonarLint
find <PATH> -name "sonar-*.xml" -o -name ".sonarlint" | head -5
```

Ejecución manual: dev corre SonarLint en IDE. No se puede automatizar desde CLI sin Sonar Server. **INFORMATIVO**.

---

## BLOQUE 10 — SOAP specifics (solo si `projectType = SOAP`)

**Origen:** [COMMIT-bf913b9] + análisis 0015

Si `projectType != SOAP`, saltar este bloque.

### Check 10.1 — Tests de integración usan PascalCase en XML

```bash
grep -rnE "<[a-z].*01\b" <PATH>/src/test/java/ | grep -i "consultar\|crear\|eliminar"
```

XML requests/responses con elemento raíz en minúscula → **HIGH**.

### Check 10.2 — `@PayloadRoot.localPart` = PascalCase (ya en Bloque 3.2)

Cross-reference. No re-auditar.

### Check 10.3 — `BancsClientHelper` abstract + 1 subclase por TX

```bash
grep -l "abstract class BancsClientHelper" <PATH>/src/main/java/com/pichincha/sp/infrastructure/output/adapter/bancs/helper/
ls <PATH>/src/main/java/com/pichincha/sp/infrastructure/output/adapter/bancs/helper/Tx*BancsClientHelper.java
```

- 0 subclases `Tx{CODE}BancsClientHelper` → **HIGH** (patrón no aplicado)
- Helper concreto `@Component` sin ser subclase → **HIGH**

### Check 10.4 — `WebServiceConfig` + `NamespacePrefixInterceptor`

```bash
find <PATH>/src/main/java -name "WebServiceConfig.java" -o -name "NamespacePrefixInterceptor.java"
```

Faltante → **HIGH**.

---

## BLOQUE 11 — REST strategies (solo si aplica)

**Origen:** wsclientes0024 (gold standard REST, FROZEN 2026-04-14)

Si `projectType != REST`, saltar.

### Check 11.1 — ¿El servicio tiene failover Bancs↔OCP?

Decidir si aplica por presencia de:
```bash
ls <PATH>/src/main/java/com/pichincha/sp/infrastructure/output/adapter/stratio/ 2>/dev/null
ls <PATH>/src/main/java/com/pichincha/sp/infrastructure/output/adapter/strategy/ 2>/dev/null
```

Si NO hay `stratio/` ni `strategy/` → servicio REST mono-fuente, **saltar Bloque 11**.

### Check 11.2 — 4 strategies presentes

```bash
ls <PATH>/src/main/java/com/pichincha/sp/infrastructure/output/adapter/strategy/
```

Debe listar: `BancsOnlyStrategy.java`, `BancsWithOcpFailoverStrategy.java`, `OcpOnlyStrategy.java`, `OcpWithBancsFailoverStrategy.java`.

Menos de 4 → **HIGH**.

### Check 11.3 — `CustomerQueryStrategyPort` como interfaz con `withDataSource()` estático

```bash
grep -A3 "interface CustomerQueryStrategyPort" <PATH>/src/main/java/com/pichincha/sp/application/*/port/output/CustomerQueryStrategyPort.java
```

Debe ser `public interface` con método `static Customer withDataSource(...)`. Si no → **MEDIUM**.

### Check 11.4 — `CustomerStrategyConfig` bean factory

```bash
grep -l "CustomerStrategyConfig" <PATH>/src/main/java/com/pichincha/sp/infrastructure/config/
```

Faltante → **HIGH**.

### Check 11.5 — `application.yml` tiene `customer.datasource` y `customer.failover.enabled`

```bash
grep -E "^  datasource:|^    enabled:" <PATH>/src/main/resources/application.yml
```

Faltante → **HIGH**.

### Check 11.6 — Tests de cada strategy

```bash
ls <PATH>/src/test/java/com/pichincha/sp/infrastructure/output/adapter/strategy/
```

4 archivos `*StrategyTest.java`. Menos → **MEDIUM**.

---

## BLOQUE 12 — REST specifics (solo si `projectType = REST`)

**Origen:** wsclientes0024 (gold standard REST) + prompt `REST/02-REST-migrar-servicio.md`

Si `projectType != REST`, saltar este bloque.

### Check 12.1 — `ErrorResolverHandler` existe e implementa `ErrorWebExceptionHandler`

```bash
grep -rl "implements ErrorWebExceptionHandler" <PATH>/src/main/java/
```

Faltante → **HIGH**. Sin esto, errores inesperados devuelven JSON Spring default en vez de SOAP Fault XML.

### Check 12.2 — Error Resolver chain completa (sealed hierarchy)

```bash
# Debe existir ErrorResolver (abstract sealed), y los 3 resolvers concretos
grep -rl "sealed class ErrorResolver\|abstract sealed class ErrorResolver" <PATH>/src/main/java/
grep -rl "final class GlobalErrorExceptionResolver" <PATH>/src/main/java/
grep -rl "final class ResponseStatusExceptionResolver" <PATH>/src/main/java/
grep -rl "final class UnexpectedErrorResolver" <PATH>/src/main/java/
```

Menos de 4 archivos → **HIGH**.

### Check 12.3 — SoapFault DTOs existen

```bash
find <PATH>/src/main/java -name "SoapFaultDto.java" -o -name "SoapFaultBodyDto.java" -o -name "SoapFaultEnvelopeDto.java"
```

Menos de 3 archivos → **HIGH**. El `ErrorResolverHandler` necesita estos DTOs para serializar SOAP Faults.

### Check 12.4 — `package-info.java` con `@XmlSchema` para prefijos NS1/NS2

```bash
find <PATH>/src/main/java -path "*/adapter/rest/dto/package-info.java"
grep "@XmlSchema" <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/rest/dto/package-info.java 2>/dev/null
```

Faltante → **MEDIUM**. Sin `@XmlSchema`, JAXB genera prefijos `ns2:`/`ns3:` que rompen clientes legacy IIB.

### Check 12.5 — `ReactiveContextWebConfig` (WebFilter)

```bash
grep -rl "ReactiveContextWebConfig" <PATH>/src/main/java/
grep -rl "implements WebFilter" <PATH>/src/main/java/com/pichincha/sp/infrastructure/config/
```

Faltante → **MEDIUM**. Sin esto, headers `x-guid`/`x-app` no se propagan downstream en cadenas reactivas.

### Check 12.6 — NO existe `BancsClientHelper` en proyecto REST

```bash
find <PATH>/src/main/java -name "BancsClientHelper.java" -o -name "*BancsClientHelper.java"
```

Si existe → **HIGH**. En REST se usa adapter directo con `@BancsService("ws-txNNNNNN")`, no BancsClientHelper.

### Check 12.7 — NO existe `WebServiceConfig` ni `NamespacePrefixInterceptor` en proyecto REST

```bash
find <PATH>/src/main/java -name "WebServiceConfig.java" -o -name "NamespacePrefixInterceptor.java"
```

Si existe → **HIGH**. Estos son artefactos SOAP (Spring WS) que no aplican a REST (WebFlux).

### Check 12.8 — SOAP Envelope DTOs existen (REST los necesita manualmente)

```bash
find <PATH>/src/main/java -name "SoapEnvelopeRequestDto.java" -o -name "SoapEnvelopeResponseDto.java" -o -name "SoapBodyRequestDto.java" -o -name "SoapBodyResponseDto.java"
```

Menos de 4 archivos → **HIGH**. En REST, Spring no maneja envelopes SOAP automáticamente — se requieren DTOs manuales con JAXB annotations.

### Check 12.9 — Controller es `@RestController` (no `@Endpoint`)

```bash
grep -rn "@RestController" <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/rest/impl/
grep -rn "@Endpoint" <PATH>/src/main/java/com/pichincha/sp/infrastructure/input/adapter/
```

Si hay `@Endpoint` en REST → **HIGH**. Si no hay `@RestController` → **HIGH**.

### Check 12.10 — Adapter BANCS usa `@BancsService` directamente

```bash
grep -rn "@BancsService" <PATH>/src/main/java/com/pichincha/sp/infrastructure/output/adapter/bancs/
```

0 matches → **HIGH**. El adapter BANCS debe inyectar `BancsClient` con `@BancsService("ws-txNNNNNN")`.

---

## FORMATO DEL REPORTE

Generás el reporte en este formato, en el orden de los bloques. Para cada check: emoji de estado + descripción corta + detalles si FAIL.

```markdown
# Post-Migration Checklist Report
**Project:** <path>
**Type:** SOAP | REST
**Gold standard:** wsclientes0015 | wsclientes0024
**Date:** YYYY-MM-DD

---

## Summary
| Block | Pass | HIGH | MEDIUM | LOW |
|---|---|---|---|---|
| 1 Hexagonal | 4/5 | 1 | 0 | 0 |
| 2 Logging | ... | ... | ... | ... |
| ... | | | | |
| **TOTAL** | **X/Y** | **N** | **N** | **N** |

**Verdict:** READY_TO_MERGE | BLOCKED_BY_HIGH | READY_WITH_FOLLOW_UP

---

## Block 1 — Arquitectura hexagonal

### 1.1 Capas presentes — ✅ PASS
Encontradas: `application/`, `domain/`, `infrastructure/`

### 1.2 Domain sin imports framework — ✅ PASS
0 matches.

### 1.4 UN solo output port Bancs — ❌ HIGH
**Hallado:** 3 ports Bancs en `application/output/port/`:
  - `CustomerAddressBancsPort.java`
  - `GeoLocationBancsPort.java`
  - `CorrespondenceBancsPort.java`
**Acción:** consolidar en un único `CustomerAddressBancsPort` con los métodos de los otros dos.
**Ref:** [FB-JG]

...

---

## Changelog del checklist
- 2026-04-14: versión inicial
```

**Orden de severidad en la salida:**
1. HIGH primero (bloqueante)
2. MEDIUM segundo (merge con ticket de follow-up)
3. LOW tercero (nice-to-have)
4. PASS agrupado al final como conteo (no listar uno por uno)

**Verdict:**
- `READY_TO_MERGE` si 0 HIGH
- `READY_WITH_FOLLOW_UP` si 0 HIGH y ≤ 3 MEDIUM
- `BLOCKED_BY_HIGH` si ≥ 1 HIGH

---

## EJEMPLO DE OUTPUT (preview sobre wsclientes0007)

Aplicado a `C:\Dev\Banco Pichincha\CapaMedia\0007\destino` a fecha 2026-04-14:

```markdown
# Post-Migration Checklist Report
**Project:** C:\Dev\Banco Pichincha\CapaMedia\0007\destino
**Type:** SOAP
**Gold standard:** wsclientes0015
**Date:** 2026-04-14

## Summary
| Block | Pass | HIGH | MEDIUM | LOW |
|---|---|---|---|---|
| 0 Pre-check | 2/2 | 0 | 0 | 0 |
| 1 Hexagonal | 5/5 | 0 | 0 | 0 |
| 2 Logging | 4/5 | 0 | 0 | 1 |
| 3 Naming | 3/4 | 1 | 0 | 0 |
| 4 Validaciones | 2/4 | 1 | 1 | 0 |
| 5 Error handling | 4/4 | 0 | 0 | 0 |
| 6 Mappers/tipos | 3/4 | 0 | 1 | 0 |
| 7 Config externa | 2/6 | 2 | 2 | 0 |
| 8 Versiones | 3/4 | 0 | 1 | 0 |
| 9 Tests | 3/5 | 0 | 2 | 0 |
| 10 SOAP specifics | 4/4 | 0 | 0 | 0 |
| 11 REST strategies | N/A | — | — | — |
| 12 REST specifics | N/A | — | — | — |
| **TOTAL** | **35/47** | **4** | **7** | **1** |

**Verdict:** BLOCKED_BY_HIGH

---

## Block 3 — Naming de métodos

### 3.1 Output ports usan `get*` — ❌ HIGH
**Hallado:** 2 métodos `query*` en puerto de lectura:
  - `BancsCustomerOutputPort.queryTransactionalContact`
  - `BancsCustomerOutputPort.queryCustomerInfoField`
**Acción:** renombrar a `getTransactionalContact` y `getCustomerInfoField` (+ usos en service y tests).
**Ref:** [FB-JG]

### 3.2 `@PayloadRoot.localPart` PascalCase — ✅ PASS
`"ConsultarContactoTransaccional01"` correcto.

### 3.3 Java method camelCase — ✅ PASS
### 3.4 postProcessWsdl sin decapitalize — ✅ PASS
Commit bf913b9 revirtió la lógica problemática.

---

## Block 4 — Validaciones

### 4.1 HeaderRequestValidator existe — ❌ HIGH
**Hallado:** 0 archivos en `infrastructure/input/adapter/soap/util/`.
**Acción:** crear `HeaderRequestValidator` siguiendo el patrón de `wsclientes0024/HeaderRequestValidator.java` adaptado al tipo JAXB `GenericHeaderIn` que llega vía Spring WS.
**Ref:** [FB-JG]

### 4.2 No está en domain/application — ✅ PASS
### 4.3 Controller invoca validator — ⚠ MEDIUM
**Nota:** consecuencia de 4.1. Se resolverá junto con 4.1.
### 4.4 Validaciones body en domain/app — ✅ PASS (informativo)

---

## Block 7 — Configuración externa

### 7.1 catalog-info.yaml completo — ❌ HIGH
**Hallado:** archivo con placeholders sin resolver:
  - `<owner>`, `<system>`, `<domain>`, `<lifecycle>`, `<dynatrace-entity-id>`, `<sonarcloud-project-key>`
  - Bloque `spec.definition:` presente (debe eliminarse)
  - Link de SwaggerHub presente (debe eliminarse)
**Acción:** reemplazar con template canónico (ver §7 del checklist).
**Ref:** [FB-JG]

### 7.2 azure-pipelines.yml — ✅ PASS
`CMDB_APPLICATION_ID: "Red Hat OpenShift Container Platform"` ✓
`KUBERNETES_NAMESPACE: tnd-middleware` ✓

### 7.3 application.yml usa ENV del Helm — ⚠ MEDIUM
**Hallado:** 2 URLs con valor por defecto hardcoded (aceptable), 0 secrets hardcoded.
**Acción:** verificar que todos los endpoints productivos usen `${VAR:default}`.

### 7.4 Validar Bancs/WebClient/CircuitBreaker — ⚠ MEDIUM
**Hallado:** `bancs:` ✓, `webclient:` ✓, `resilience4j:` falta sección explícita.

### 7.5 Helm por entorno — ❌ HIGH
**Hallado:** solo `values.yaml` genérico, faltan `values-dev.yaml`, `values-test.yaml`, `values-prod.yaml`.
**Ref:** [PDF-OFICIAL]

### 7.6 @ConfigurationPropertiesScan — ✅ PASS

---

## Block 2 — Logging

### 2.1 @BpTraceable en Controller — ✅ PASS
### 2.2 @BpLogger en todos los @Service — ✅ PASS (1/1 services, 1/1 métodos públicos cubiertos)
### 2.3 @BpLogger en Adapters — ✅ PASS (4 métodos en BancsCustomerAdapter)
### 2.4 Logs con GUID — ✅ PASS
### 2.5 Abuso de log.info — ℹ LOW
**Hallado:** `BancsCustomerAdapter.java` con 6 log.info. Revisar si son necesarios o parte del flujo normal.

---

## Block 6 — Mappers y tipos

### 6.1 Mappers dedicados — ✅ PASS
SOAP mapper + Bancs mapper + repository mapper presentes.
### 6.2 MapStruct — ✅ PASS
### 6.3 Sin new Record(>=8 args) inline — ⚠ MEDIUM
**Hallado:** `ConsultarContactoTransaccionalService.handleFallbackPath` y `.buildDirectResult` construyen `new CustomerContactResult(12 args)` inline.
**Acción:** agregar factory estático `CustomerContactResult.fromFallback(...)` / `.fromDirect(...)` en el record.
### 6.4 Sin Object/Map<String,Object> en puertos — ✅ PASS

---

## Block 9 — Tests y calidad

### 9.1 JaCoCo ≥ 75% — ✅ PASS (80.12%)
### 9.2 Build verde — ✅ PASS (61 tests OK)
### 9.3 Tests por capa — ⚠ MEDIUM
**Hallado:** 1 test de service, 1 de controller, 4 de adapter/helper = 6 archivos. Falta test específico de `PhoneNormalizationUtil`.
### 9.4 @Nested + S2187 — ✅ PASS (no usa @Nested)
### 9.5 SonarLint — ℹ INFO (ejecutar en IDE local)

---

## Changelog del checklist
- 2026-04-14: versión inicial
```

---

## CÓMO EJECUTARLO

Desde Claude Code:

```
/03-checklist-post-migracion.md path="C:/Dev/Banco Pichincha/CapaMedia/0007/destino"
```

O directamente en una instrucción:

> "Aplicá el checklist post-migración de `prompts/migracion/03-checklist-post-migracion.md` sobre `C:/Dev/Banco Pichincha/CapaMedia/0007/destino`. Devolveme solo HIGH y MEDIUM, skip PASS."

El agente debe:
1. Detectar tipo de proyecto (Bloque 0)
2. Correr cada check del bloque relevante
3. Agrupar resultados por severidad
4. Emitir el reporte en el formato especificado
5. Cerrar con Verdict

NO auto-corrige. NO modifica archivos. Solo lee y reporta.

# Prompt: Post-Migration - Prepare Bank Integration

> **Phase:** 3 — Post-Migration
> **Version:** 1.0.0
> **Date:** 2026-04-02
> **Project:** Capa Media - Migracion de Capa Comun
> **Usage:** Load as a prompt in Claude Code, Windsurf, Copilot, or any AI-enabled IDE after completing the local migration of a service.

---

## ROLE

You are a specialist in integration preparation for Banco Pichincha. Your job is to scan a migrated Java project (Spring Boot, hexagonal architecture OLA1), identify everything that is pending, stubbed, or incomplete, and produce actionable documentation for the developer who will connect this service to the bank's real infrastructure.

Your output must be a single, self-contained document that serves as a complete roadmap: from configuring credentials to running the first smoke test on the cluster.

### Role Principles

1. **DO NOT INVENT INFORMATION.** If a TX code, URL, or field cannot be determined from the code, mark it as `TBD` with context on where to obtain it.
2. **Classify every finding:**
   - `DIRECT EVIDENCE` — visible in the source code
   - `INFERENCE` — deduced from context (must be justified)
   - `TBD` — requires information from the Core Adapter team or another team
3. **Be exhaustive.** A forgotten stub that reaches production is an incident. Scan ALL files.
4. **Language:** Headers and business descriptions in English. Code, class/method names, and technical terms in English.

---

## REQUIRED INPUT

| #   | Artifact                                                          | Purpose                                                          | Required |
| --- | ----------------------------------------------------------------- | ---------------------------------------------------------------- | -------- |
| 1   | Migrated Java project directory                                   | Scan code, config, Helm                                          | YES      |
| 2   | `ANALISIS_<ServiceName>.md` from Phase 1                              | Cross-reference: legacy UMPs, field mapping, business logic      | YES      |
| 3   | `NO_VERIFICABLE_LOCAL.md` from Phase 2 (if it exists)             | Items that could not be validated without real infrastructure     | NO       |
| 4   | `MIGRATION_REPORT.md` from the project                            | Migration decisions, omitted fields, uncertainties               | YES      |

**Before executing:** Verify that the input files exist. If `ANALISIS_<ServiceName>.md` does not exist, STOP and notify that Phase 1 must be completed first.

---

## ACTIONS TO EXECUTE

### 1. Stub and TODO Scan

Scan ALL `*.java` files in the project looking for the following markers:

**Search patterns:**

- `TODO`, `FIXME`, `TBD`, `HACK`, `XXX`
- `UnsupportedOperationException`
- `<pendiente_validar>`
- Methods that return `Mono.error(new UnsupportedOperationException(...))`
- Methods that return `Mono.empty()` in output adapters (`infrastructure/output/adapter/`) — potential fire-and-forget stubs
- Methods that return `null` or empty collections as placeholders
- Output port interfaces (`application/port/output/`) without a concrete implementation in `infrastructure/output/adapter/bancs/impl/`

**For EACH stub or pending item found, generate a REMEDIATION CARD:**

````markdown
### PENDING: [AdapterName] — [legacy UMP it replaces]

**File:** src/main/java/.../[Adapter].java, line XX
**Type:** BANCS adapter stub | Logic TODO | Missing validation | Pending configuration

**What it replaces:**

- Legacy UMP: [UMP name] / [operation]
- Business purpose: [clear description of what this component does]
- Legacy evidence: [IIB node / ESQL procedure / WAS Java class that originates it]

**Required BANCS TX:**

- TX code: [code if known, or TBD — request from Core Adapter team (adapter service name)]
- BANCS service: [ws-txNNNNNN if known, or TBD]
- Core Adapter endpoint: POST /bancs/trx/{txCode}
- Core Adapter service: [adapter service name, e.g.: tnd-msa-ad-bnc-customers]

**Request to send:**
| Field | Type | Source | Format | Notes |
|---|---|---|---|---|
| identificacion | String | [Class.field] | As-is | |
| tipoIdentificacion | String | [Class.field] | As-is | |
| cif | String | [Class.field] | [format: zero-padded 16 chars | integer-cast without padding] | CRITICAL: verify format with Core Adapter team |

**Response to map:**
| BANCS response field | Java domain field | Java type | Transformation |
|---|---|---|---|
| [field] | [DomainClass.field] | String | As-is / trim / lowercase / format |

**CIF format (if applicable):** [zero-padded 16 chars | integer-cast without padding | N/A]

> CRITICAL: CIF format varies between BANCS transactions. See section "CIF Formatting Asymmetry"
> in MIGRATION_REPORT.md. Using the wrong format causes "client not found" in BANCS.

**Legacy ESQL/Java evidence:** [Procedure/method], lines XX-YY of file [name]

**Implementation pattern (copy and adapt):**

```java
// File: src/main/java/.../infrastructure/output/adapter/bancs/impl/[Adapter].java
@Override
public [DomainResponse] [method]([parameters]) {
  var requestDto = new Tx[CODE]RequestDto([parameters]);
  Tx[CODE]ResponseDto response = bancsClient.execute(
      "[TX_CODE]", ctx, requestDto, Tx[CODE]ResponseDto.class);
  return mapToDomain(response);
}
```
````

**Blocking dependencies:**

- [ ] TX code confirmed by Core Adapter team
- [ ] Request/response format validated against BANCS spec
- [ ] CIF format verified (padded vs unpadded)
- [ ] Mapper created and tested

````

**After completing the scan, generate a summary table:**

```markdown
## Stubs and Pending Items Summary

| # | File | Line | Type | Legacy UMP | BANCS TX | Status |
|---|---|---|---|---|---|---|
| 1 | BancsCustomerInfoAdapter.java | 45 | Adapter stub | UMPClientes0002 | TX060480 | Implemented |
| 2 | BancsNotificationContactAdapter.java | 32 | Adapter stub | UMPClientes0003 | TBD | PENDING |
| 3 | BancsLegacyContactAdapter.java | 28 | Adapter stub | UMPClientes0020 | TBD | PENDING |
| 4 | BancsUpdateContactAdapter.java | 35 | Fire-and-forget stub | UMPClientes0005 | TBD | PENDING |

**Implemented stubs:** X/Y (Z%)
**Pending stubs:** N — require TX codes from Core Adapter team
````

---

### 2. Environment Variables Audit

Scan `application.yml`, `application-test.yml`, and all `helm/*.yml` files to generate a complete inventory of environment variables.

**Process:**

1. Extract every variable with pattern `${CCC_*}` from `application.yml`
2. For each variable, verify if it has a value assigned in `helm/dev.yml`, `helm/test.yml`, `helm/prod.yml`
3. Verify if `application-test.yml` defines defaults for local testing
4. Detect orphan variables: defined in Helm but not referenced in `application.yml`
5. Detect ghost variables: referenced in `application.yml` but without a value in any Helm file

**Output table:**

```markdown
## Environment Variables Audit (CCC\_\*)

| Variable                  | application.yml              | dev.yml            | test.yml           | prod.yml           | application-test.yml       | Status            |
| ------------------------- | ---------------------------- | ------------------ | ------------------ | ------------------ | -------------------------- | ----------------- |
| CCC_BANCS_BASE_URL        | ${CCC_BANCS_BASE_URL}        | http://service-... | http://service-... | http://service-... | http://localhost:8080/mock | OK                |
| CCC_BANCS_CONNECT_TIMEOUT | ${CCC_BANCS_CONNECT_TIMEOUT} | 500ms              | 500ms              | 500ms              | 500ms                      | OK                |
| CCC_LOG_COM_PICHINCHA_SP  | ${CCC_LOG_COM_PICHINCHA_SP}  | INFO               | INFO               | INFO               | DEBUG                      | OK                |
| CCC_TX_CODE_0003          | ${CCC_TX_CODE_0003}          | -                  | -                  | -                  | -                          | MISSING           |
| CCC_PAYLOAD_MODE          | ${CCC_PAYLOAD_MODE}          | FULL               | FULL               | PARTIAL            | FULL                       | OK (prod differs) |

### Status Legend

- **OK** — Variable defined in application.yml and with a value in all Helm environments
- **OK (prod differs)** — Variable present in all Helm files, but prod has a different value (verify if intentional)
- **MISSING** — Variable referenced in application.yml but without a value in one or more Helm files
- **ORPHAN** — Variable in Helm but not referenced in application.yml (candidate for removal)
- **NO TEST DEFAULT** — Variable without a value in application-test.yml (tests may fail locally)
```

**Critical warnings to emit:**

- Variable `CCC_BANCS_BASE_URL` in prod points to a different cluster than dev/test: verify that this is intentional
- Feature flag variable (`CCC_ACTUALIZAR_CT_MIGRACION`) with a different value in prod (`false`) vs dev/test (`true`): document why
- Circuit breaker variables identical in all environments: consider whether prod needs more conservative values

---

### 3. Credentials Inventory

List ALL credentials, tokens, and secrets needed to compile, run, and deploy the service.

**Scan:**

- `build.gradle` — Maven repositories with `credentials {}`
- `build.gradle` — Sonar tasks with `System.env.SONAR_*`
- `application.yml` — connection properties with pattern `${SPRING_DATASOURCE_*}` (WAS mode)
- `azure-pipelines.yml` (if it exists) — variable groups and referenced secrets
- `Dockerfile` — ARGs that receive secrets at build time

```markdown
## Credentials Inventory

| Credential        | Where it is used             | File                       | How to obtain it                                                  | Scope/Permissions  |
| ----------------- | ---------------------------- | -------------------------- | ----------------------------------------------------------------- | ------------------ |
| ARTIFACT_USERNAME | Azure Artifacts Maven repo   | build.gradle, line ~52     | Azure DevOps -> User Settings -> Personal Access Tokens -> Create | Packaging (Read)   |
| ARTIFACT_TOKEN    | Azure Artifacts Maven repo   | build.gradle, line ~53     | Same flow as the previous PAT — the token is the password         | Packaging (Read)   |
| SONAR_TOKEN       | SonarQube analysis task      | build.gradle (sonar block) | SonarQube -> My Account -> Security -> Generate Token             | Execute Analysis   |
| SONAR_PROJECT_KEY | SonarQube project identifier | build.gradle (sonar block) | SonarQube -> Project -> Settings -> General -> Project Key        | N/A (not a secret) |
| SONAR_HOST_URL    | SonarQube server URL         | build.gradle (sonar block) | Request from the quality team                                     | N/A (not a secret) |
| SONAR_ORG         | SonarQube organization       | build.gradle (sonar block) | Request from the quality team                                     | N/A (not a secret) |

### Pipeline Credentials (Azure DevOps)

| Variable Group          | Variables            | How to request access                                                       |
| ----------------------- | -------------------- | --------------------------------------------------------------------------- |
| [variable-group-name]   | [list of variables]  | Azure DevOps -> Pipelines -> Library -> request permissions from DevOps team |

### Notes

- Azure DevOps PATs expire. Set a reminder for renewal.
- NEVER commit credentials to the repository. Use local environment variables or `.env.local` (in .gitignore).
- For local execution, create a `.env.local` file (see First Execution Guide section).
```

---

### 4. First Execution Guide (step by step)

Generate a step-by-step guide for the developer taking over the project. Each step includes: exact command, expected output, and common error troubleshooting.

````markdown
## First Execution Guide

> Prerequisites: Java 21 (Temurin), Gradle 8.10+, Docker Desktop, Azure DevOps access

### Step 1: Clone the repository

```bash
git clone https://dev.azure.com/BancoPichinchaEC/<proyecto>/_git/<nombre-msa>
cd <nombre-msa>
```
````

### Step 2: Configure Azure Artifacts credentials

```bash
# Linux/macOS
export ARTIFACT_USERNAME="tu_usuario_pichincha"
export ARTIFACT_TOKEN="tu_pat_azure_devops"

# Windows (PowerShell)
$env:ARTIFACT_USERNAME="tu_usuario_pichincha"
$env:ARTIFACT_TOKEN="tu_pat_azure_devops"
```

> If you don't have a PAT: Azure DevOps -> User Settings -> Personal Access Tokens
> -> New Token -> Scope: Packaging (Read) -> Expiration: 90 days

### Step 3: Generate JAXB classes from WSDL

```bash
./gradlew generateFromWsdl
```

**Expected:** Classes generated in `build/generated/sources/wsdl/`
**If it fails:**

- `FileNotFoundException` on WSDL: verify that `src/main/resources/legacy/` contains the `.wsdl` and `.xsd` files
- Incorrect `schemaLocation`: XSDs must use relative paths to the `legacy/` directory
- CXF error: verify that the WSDL files are valid XML

### Step 4: Compile

```bash
./gradlew compileJava
```

**Expected:** `BUILD SUCCESSFUL`
**If it fails:**

- `cannot find symbol`: the generated JAXB classes are not found -> run `generateFromWsdl` first
- `package does not exist` in `infrastructure.soap.generated` imports: JAXB class names depend on post-processing in `gradle/postProcessWsdl.groovy` -> verify it ran correctly
- Java version error: verify `java -version` shows Java 21

### Step 5: Run tests

```bash
./gradlew test
```

**Expected:** `XX tests passed, 0 failed`
**If it fails:**

- `NoSuchBeanDefinitionException`: verify that `application-test.yml` has all necessary defaults
- `Connection refused`: some test is trying to connect to an external service -> it should be mocked
- Reactive test fails with timeout: increase timeout in `StepVerifier` or verify that mocks return `Mono.just()` correctly

### Step 6: Verify coverage

```bash
./gradlew jacocoTestReport
```

**Open report:** `build/reports/jacoco/test/html/index.html` (or `build/jacoco/html/index.html`)
**Expected:** Coverage >= 75% (instruction)
**If < 75%:** Review that classes excluded in `build.gradle` (`jacocoExclusions`) are correct:

- `Application.class` — excluded (main)
- `infrastructure/config/**` — excluded (Spring configuration)
- `infrastructure/soap/generated/**` — excluded (auto-generated classes)
- `mapper/*Impl.class` — excluded (MapStruct generates these)

### Step 7: Full build (quality gate)

```bash
./gradlew clean build
```

**Expected:** `BUILD SUCCESSFUL` including tests + JaCoCo verification
**If it fails on `jacocoTestCoverageVerification`:** coverage below 75% -> add tests, DO NOT lower the threshold

### Step 8: Create .env.local file

```bash
# Create .env.local with all CCC_* variables needed for local execution
cat > .env.local << 'EOF'
# === LOGGING ===
CCC_LOG_COM_PICHINCHA_SP=DEBUG
CCC_TRACE_LOGGER_ENABLED=true
CCC_CUSTOM_LEVEL_ENABLED=true
CCC_CUSTOM_LEVEL_INFO_ENABLED=true
CCC_CUSTOM_LEVEL_DEBUG_ENABLED=true
CCC_CUSTOM_LEVEL_WARN_ENABLED=true
CCC_CUSTOM_LEVEL_ERROR_ENABLED=true
CCC_PAYLOAD_MODE=FULL

# === BUSINESS (service-specific) ===
# [Include ALL service-specific CCC_* variables]
# Example for WSClientes0007:
# CCC_LONGITUD_CELULAR_NACIONAL=10
# CCC_LONGITUD_CELULAR_INTERNACIONAL=12
# CCC_LONGITUD_CELULAR_INTER_ERROR=13
# CCC_PREFIJO_INTERNACIONAL=593
# CCC_ACTUALIZAR_CT_MIGRACION=true

# === BANCS CLIENT ===
CCC_BANCS_BASE_URL=http://localhost:8081/bancs/trx/
CCC_BANCS_CONNECT_TIMEOUT=500ms
CCC_BANCS_READ_TIMEOUT=800ms
CCC_BANCS_MAX_CONNECTIONS=-1
CCC_BANCS_MAX_IDLE_TIME=10s
CCC_BANCS_PENDING_ACQUIRE_MAX_COUNT=-1
CCC_BANCS_PENDING_ACQUIRE_TIMEOUT=2s
CCC_BANCS_MAX_IN_MEMORY_SIZE=5MB
CCC_BANCS_LIB_SUPPORT_ENABLED=true

# === CIRCUIT BREAKER ===
CCC_BANCS_CIRCUIT_BREAKER_ENABLED=false
CCC_BANCS_CIRCUIT_BREAKER_SLIDING_WINDOW_SIZE=3000
CCC_BANCS_CIRCUIT_BREAKER_FAILURE_RATE_THRESHOLD=10
CCC_BANCS_CIRCUIT_BREAKER_WAIT_DURATION_IN_OPEN_STATE=20s
CCC_BANCS_CIRCUIT_BREAKER_PERMITTED_NUMBER_OF_CALLS_IN_HALF_OPEN_STATE=50
CCC_BANCS_CIRCUIT_BREAKER_SLOW_CALL_RATE_THRESHOLD=10
CCC_BANCS_CIRCUIT_BREAKER_SLOW_CALL_DURATION_THRESHOLD=1000ms
EOF
```

> IMPORTANT: `.env.local` must be in `.gitignore`. NEVER commit this file.
> For local execution without a real Core Adapter, consider using WireMock or mockserver.

### Step 9: Docker build

```bash
./gradlew bootJar
docker build -t <nombre-msa>:local .
docker run -p 8080:8080 --env-file .env.local <nombre-msa>:local
```

**Expected:** Container starts without errors, Spring Boot boots in ~10s
**If it fails:**

- `java.lang.UnsupportedOperationException`: an adapter stub was invoked -> disable the affected endpoint or configure a mock
- `PlaceholderResolutionException`: a CCC\_\* variable is missing from `.env.local` -> add it
- Port in use: change the mapping `-p 8083:8080`

### Step 10: Smoke test — health

```bash
curl -s http://localhost:8080/actuator/health
```

**Expected:**

```json
{ "status": "UP" }
```

**If it returns `DOWN` or does not respond:** Review container logs with `docker logs <container-id>`

### Step 11: Smoke test — SOAP endpoint

```bash
curl -X POST http://localhost:8080/IntegrationBus/soap/<ServiceName> \
  -H "Content-Type: text/xml" \
  -H "SOAPAction: http://bpichincha.com/servicios/<OperationName>" \
  -H "x-app: 00270" \
  -H "x-guid: 550e8400-e29b-41d4-a716-446655440000" \
  -H "x-channel: 02" \
  -H "x-medium: 020002" \
  -H "x-session: LOCAL-TEST-001" \
  -H "x-device: local-device" \
  -H "x-device-ip: 127.0.0.1" \
  -d @test-request.xml
```

> Create `test-request.xml` with a valid SOAP request (see QA: Use Cases section).
> Corporate headers (`x-app`, `x-guid`, etc.) are mandatory — without them the request will be rejected.

**Expected:** SOAP response with `error.codigo=0` (if BANCS adapters are implemented) or `error.codigo=<code>` if stubs are active.

````

---

### 5. QA Section: Use Cases for Testing

Based on the analysis document (`ANALISIS_<ServiceName>.md` or `MIGRATION_REPORT.md`), generate testing use cases in Given/When/Then format.

**Required format per case:**

```markdown
## Use Cases — <ServiceName>

Microservice: [full service name]
**Operation:** `<OperationName>`
**Endpoint:** `POST /IntegrationBus/soap/<ServiceName>`

---

## Successful Cases

---

### Case 1 — [Descriptive title: what scenario is being tested]

**Given** that the <ServiceName> service is active and [preconditions: backends available, config flags],
**When** a request is sent with [description of input data],
**Then** the service responds HTTP 200 OK with `error.codigo=0` and [description of expected result].

**Request:**
```xml
<soapenv:Envelope
    xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:bp="http://bpichincha.com/servicios">
    <soapenv:Header/>
    <soapenv:Body>
        <bp:<OperationName>>
            <headerIn>
                <dispositivo>test-device-token</dispositivo>
                <empresa>0010</empresa>
                <canal>02</canal>
                <medio>020002</medio>
                <aplicacion>00270</aplicacion>
                <agencia>00012</agencia>
                <tipoTransaccion>000000000</tipoTransaccion>
                <geolocalizacion>0000000</geolocalizacion>
                <usuario>USINTERT</usuario>
                <unicidad>test-unicidad-token</unicidad>
                <guid>550e8400e29b41d4a716446655440000</guid>
                <fechaHora>202604021200000000</fechaHora>
                <filler>00000000000000000000000000000000000000000000000000</filler>
                <idioma>es</idioma>
                <sesion>TESTSESSION001</sesion>
                <ip>192.168.1.100</ip>
                <idCliente>[identificacion]</idCliente>
                <tipoIdCliente>0001</tipoIdCliente>
                <bancs>
                    <teller>00008002</teller>
                    <terminal>1</terminal>
                    <institucion>0010</institucion>
                    <agencia>0012</agencia>
                    <estacion>1</estacion>
                    <aplicacion/>
                    <canal/>
                </bancs>
            </headerIn>
            <bodyIn>
                [request fields with example values]
            </bodyIn>
        </bp:<OperationName>>
    </soapenv:Body>
</soapenv:Envelope>
````

**Response:**

```xml
[Full expected response XML]
```

**HTTP Status:** 200
**error.codigo:** 0
**error.mensaje:** OK

````

**Required case categories:**

#### Successful cases (generate at least):
1. Basic happy path — all required fields present, backend responds OK
2. Optional fields absent — fields with `minOccurs="0"` omitted from the request
3. Optional fields empty — optional fields sent as `""` or spaces
4. Identification normalization — identification with leading zeros (if applicable to the service)
5. Fallback logic (if the service has it) — primary source fails, secondary responds

#### Error cases (generate at least):
1. Empty identification — `identificacion` empty or null
2. Identification with only spaces — trim results in empty
3. Null request — `bodyIn` absent from the SOAP envelope
4. Backend unavailable — timeout or connection error to Core Adapter
5. Client not found — backend returns "not found"
6. Unexpected technical error — uncontrolled internal error (SOAP Fault, HTTP 500)

#### For each case include:
- Complete request XML (do not use `<!-- same as Case 1 -->` for the first 3 cases of each category; then it can be referenced)
- Complete expected response XML
- Expected HTTP status
- Expected `error.codigo`
- Expected `error.mensaje`
- Notes on specific behavior (normalization, fallback, error propagation)

**Summary table at the end:**

```markdown
## Use Cases Summary

| Case | Condition | HTTP | error.codigo | bodyOut |
|---|---|---|---|---|
| 1 | Basic happy path | 200 | 0 | Present |
| 2 | Optional field absent | 200 | 0 | Present |
| ... | ... | ... | ... | ... |
| E1 | Empty identification | 200 | 1 | Absent |
| E2 | Null request (bodyIn absent) | 500 | -- | SOAP Fault |
| ... | ... | ... | ... | ... |
````

---

### 6. Readiness Score

Evaluate each aspect of the project and assign a score. The evaluation must be objective and based on evidence from the code.

```markdown
## Integration Readiness Score

| #   | Aspect                                          | Status                  | Detail                     | Score  |
| --- | ----------------------------------------------- | ----------------------- | -------------------------- | ------ |
| 1   | Compiles locally (`./gradlew compileJava`)      | YES / NO / NOT_VERIFIED | [output or error]          | 10%    |
| 2   | Tests pass (`./gradlew test`)                   | YES / NO / NOT_VERIFIED | [X tests passed, Y failed] | 10%    |
| 3   | JaCoCo >= 75%                                   | YES / NO / NOT_VERIFIED | [X.X%]                     | 10%    |
| 4   | BANCS adapters implemented (no stubs)           | X/Y implemented         | [list of pending]          | 15%    |
| 5   | Environment variables complete in Helm          | X/Y variables OK        | [list of missing]          | 10%    |
| 6   | Credentials documented                          | YES / NO                | [which ones are missing]   | 5%     |
| 7   | Helm probes configured (liveness + readiness)   | YES / NO                | [in which environments]    | 5%     |
| 8   | Prod replicas >= 2 and HPA enabled              | YES / NO                | [current replicaCount]     | 5%     |
| 9   | MIGRATION_REPORT.md complete                    | YES / NO                | [missing sections]         | 10%    |
| 10  | QA use cases generated                          | YES / NO                | [X successful, Y error]    | 10%    |
| 11  | Dockerfile with Java 21 and health check        | YES / NO                | [current base image]       | 5%     |
| 12  | Ports are interfaces (not abstract classes)      | YES / NO                | [violations found]         | 5%     |
|     | **TOTAL SCORE**                                 |                         |                            | **?%** |

### Score Calculation

- Each aspect has an assigned percentage weight
- YES = 100% of weight, NO = 0%, PARTIAL = proportional (e.g.: 3/4 adapters = 75% of weight)
- NOT_VERIFIED counts as 0% (it cannot be assumed to work without verification)

### Interpretation and Next Step

| Range      | Classification    | Required Action                                                                                                                                 |
| ---------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **>= 80%** | READY FOR DEV     | Deploy to DEV environment. Start integration tests with real Core Adapter. Coordinate with QA team for use case execution.                      |
| **60-79%** | ALMOST READY      | Implement missing BANCS adapters (obtain TX codes from Core Adapter team). Complete missing environment variables. Then re-evaluate.             |
| **40-59%** | IN PROGRESS       | Significant work pending. Prioritize: (1) compilation, (2) tests, (3) adapters. Estimate 2-4 additional days of work.                           |
| **< 40%**  | NEEDS ATTENTION   | The migration has significant gaps. Review whether the Phase 1 analysis was complete. Consider pair programming with the Core Adapter team.      |

### Identified Risks

[List concrete risks that could block integration, e.g.:]

- Pending BANCS TX codes for X adapters — blocking for integration tests
- CIF format not confirmed for UMPClientes0003 — risk of "client not found" in real environment
- Circuit breaker not tested under load — risk of false positives in production
```

---

## OUTPUT FILE

**Name:** `PENDIENTES_<ServiceName>.md`

**Example:** `PENDIENTES_WSClientes0007.md`

**Location:** Root of the migrated project (alongside `MIGRATION_REPORT.md`)

**Output file structure:**

```markdown
# Integration Pending Items: <ServiceName>

> Generated: [date]
> Service: [msa-name]
> Operation: [main operation]
> Mode: [BUS (WebFlux) | WAS (MVC)]
> Source: ANALISIS\_<ServiceName>.md + code scan

---

## 1. Pending Stubs and TODOs

[Remediation cards generated in Action 1]

## 2. Environment Variables Audit

[Table generated in Action 2]

## 3. Credentials Inventory

[Table generated in Action 3]

## 4. First Execution Guide

[Step-by-step guide generated in Action 4]

## 5. QA Use Cases

[Cases generated in Action 5]

## 6. Readiness Score

[Scorecard generated in Action 6]

---

## Appendix A: .env.local Template

[Complete .env.local file with ALL CCC_* variables]

## Appendix B: Cross-reference with Phase 1 Analysis

[Table linking each legacy UMP with its current status in the migrated project]

| Legacy UMP      | Operation                       | Output Port                   | Adapter                         | BANCS TX | Status       |
| --------------- | ------------------------------- | ----------------------------- | ------------------------------- | -------- | ------------ |
| UMPClientes0002 | ConsultarInformacionBasica01    | CustomerInfoOutputPort        | BancsCustomerInfoAdapter        | TX060480 | Implemented  |
| UMPClientes0003 | ConsultarContactoNotificacion01 | NotificationContactOutputPort | BancsNotificationContactAdapter | TBD      | Stub         |
| ...             | ...                             | ...                           | ...                             | ...      | ...          |
```

---

## EXECUTION RULES

1. **Execute all 6 actions in order.** Do not skip sections.
2. **Do not assume anything works.** If you cannot execute `./gradlew compileJava`, mark it as `NOT_VERIFIED`.
3. **Always cross-reference** against the Phase 1 analysis document. Each legacy UMP must have a corresponding adapter, or a documented justification for why it was excluded (e.g.: dead code).
4. **CIF format is critical.** Different BANCS transactions expect different formats (zero-padded vs integer-cast). Document the exact format for each adapter. An incorrect format causes silent errors in production.
5. **Environment variables with different values in prod:** explicitly document WHY (e.g.: `CCC_ACTUALIZAR_CT_MIGRACION=false` in prod because contact migration only runs in dev/test, `CCC_PAYLOAD_MODE=PARTIAL` in prod for performance).
6. **Corporate headers are mandatory.** QA use cases MUST include the headers `x-app`, `x-guid`, `x-channel`, `x-medium`, `x-session`, `x-device`, `x-device-ip` in each request. Without these headers, the service will reject the request.
7. **DO NOT modify source code.** This prompt is for documentation, not implementation. If you find a bug, document it as a finding but do not fix it.
8. **Evidence or TBD.** Every assertion about TX codes, formats, or behavior must have evidence from the code or be marked as `TBD` with an indication of whom to ask.

---

## FINAL CODE REVIEW (MANDATORY)

**Execute AFTER generating the pending items document and BEFORE delivering to the user.**

### 1. Non-Hallucination Review

```
[] NO TX code was invented — all come from the migrated code or the ANALISIS
[] NO request/response field in QA cases was invented
[] NO endpoint or URL was invented — all from source code or config
[] NO environment variable value was invented — all from Helm or config
[] All unverifiable data is marked TBD with indication of whom to ask
```

### 2. Completeness Review

```
[] Each UMP from the ANALISIS has an entry in Appendix B (cross-reference)
[] Each stub has its remediation card with TX code or TBD
[] Each ${CCC_*} is audited in the environment variables table
[] QA cases cover: happy path + at least 1 error per operation
[] SOAP XMLs in QA cases include ALL corporate headers
[] The readiness score is consistent with the stubs/pending items found
[] The first execution guide has all steps complete
```

### 3. Consistency Review with Phase 1 and 2

```
[] Each UMP from the ANALISIS appears as implemented or stub — NONE omitted
[] The fields in QA cases match those from the WSDL/XSD
[] The documented CIF formats match what the ANALISIS states
[] The mode (BUS/WAS) matches between ANALISIS, migrated project, and this document
```

### 4. Report Format

Add at the end of `PENDIENTES_<ServiceName>.md`:

```markdown
## Post-Migration Code Review

**Result:** PASS | FAIL
**Invented data found:** 0 (if > 0, FIX before delivering)
**Data marked TBD:** X items
**Consistency with Phase 1:** PASS | FAIL (list discrepancies)
**Consistency with Phase 2:** PASS | FAIL (list discrepancies)
```

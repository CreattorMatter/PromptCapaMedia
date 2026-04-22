# Prompt: Migrated vs Legacy Service Review + Acceptance Criteria Validation

Use this prompt when you need to review a migrated service (or its legacy counterpart) and validate that it meets the defined acceptance criteria. The output must generate a `.md` evidence file inside the migrated service repository.

---

## Prompt

```
Review the service (migrated or legacy) and generate a Markdown report following these rules strictly:

### Analysis Scope

1. Identify the service under review (name, endpoint, operation, version).
2. Capture the actual service response (HTTP status, response time, size, encoding, state: OK/ERROR).
3. Document the XML/JSON structure of the response.
4. Detail the output headers (headerOut) and, if applicable, the error block (code, type, resource, component, backend, businessMessage).
5. If there is an error, include an analysis with:
   - Detected exception
   - Backend HTTP status
   - Internal code
   - Affected class
   - Possible causes
   - Recommended actions

### Acceptance Criteria Validation (MANDATORY)

Once the service has been reviewed, you MUST validate that it meets ALL of the following acceptance criteria. For each criterion, indicate the result: ✅ PASS / ❌ FAIL / ⚠️ PARTIAL, with the corresponding evidence.

#### Functional Criteria

| # | Criterion | Validation |
|---|-----------|------------|
| AC-01 | **Functional equivalence with legacy:** The migrated service response must be functionally equivalent to the legacy one (same fields, same values, same semantics). | Compare field by field between legacy and migrated. |
| AC-02 | **Input contract:** The request complies with the published contract (structure, types, mandatory/optional fields). | Review WSDL/OpenAPI vs the sent request. |
| AC-03 | **Output contract:** The response complies with the published contract. | Review WSDL/OpenAPI vs the received response. |
| AC-04 | **Error handling:** Error codes, type, resource and component remain consistent with legacy. | Compare error block legacy vs migrated. |
| AC-05 | **Preserved business logic:** Business rules, validations and transformations from legacy are replicated. | Compare results using the same input data. |

#### Non-Functional Criteria

| # | Criterion | Validation |
|---|-----------|------------|
| AC-06 | **Response time:** The migrated service responds in a time equal to or lower than the legacy (defined SLA). | Measure and compare times. |
| AC-07 | **Availability / resilience:** The service correctly handles backend failures (timeouts, retries, fallbacks). | Simulate core unavailability. |
| AC-08 | **Security:** Authentication, authorization and token/session handling work correctly. | Review security and session headers. |
| AC-09 | **Logging and traceability:** Logs are generated at the correct level, with traces (traceId/correlationId) and without exposing sensitive data. | Review middleware logs. |
| AC-10 | **Encoding and format:** UTF-8 respected, valid and parseable XML/JSON structure. | Validate with parser. |

#### Code Quality Criteria (if code review applies)

| # | Criterion | Validation |
|---|-----------|------------|
| AC-11 | **Unit test coverage:** greater than 85%. | JaCoCo/SonarQube report. |
| AC-12 | **Duplicated code:** 0%. | SonarQube / duplication detector report. |
| AC-13 | **Complies with the unit test guidelines** (`UNIT_TEST_GUIDELINES.md`). | Manual review + linter rules. |
| AC-14 | **No critical/high vulnerabilities:** security analysis (SCA/SAST). | SonarQube / Snyk / similar report. |

### Generation of the Evidence MD File

At the end of the review you MUST generate a Markdown file with the validated criteria. This file will be stored in the **migrated service** repository under the path:

/docs/acceptance-criteria/<ServiceName>_<Operation>_AcceptanceCriteria.md

Example:
/docs/acceptance-criteria/WSClientes0006_ConsultarInformacionBasica01_AcceptanceCriteria.md

The generated file must follow this structure:

---

# Acceptance Criteria – <Service> / <Operation>

## General Information

| Field | Value |
|-------|-------|
| Service | <name> |
| Operation | <operation> |
| Migrated version | <vX.Y.Z> |
| Validation date | <YYYY-MM-DD> |
| Owner | <name> |
| Overall result | ✅ APPROVED / ❌ REJECTED / ⚠️ APPROVED WITH OBSERVATIONS |

## Results Summary

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| AC-01 | Functional equivalence | ✅ / ❌ / ⚠️ | Link or description |
| AC-02 | Input contract | ✅ / ❌ / ⚠️ | ... |
| ... | ... | ... | ... |
| AC-14 | No vulnerabilities | ✅ / ❌ / ⚠️ | ... |

## Detail per Criterion

### AC-01 – Functional equivalence with legacy
- **Result:** ✅ PASS
- **Evidence:** <description, links to logs, screenshots, comparisons>
- **Observations:** <if applicable>

(repeat for each criterion AC-01 ... AC-14)

## Findings / Deviations

- List of findings that do not pass or only partially pass.
- For each one: impact, severity, remediation plan and owner.

## Conclusion

Short paragraph indicating whether the migrated service is ready for production, requires adjustments or must be rejected.

---

### Golden Rules

- Do NOT mark a criterion as ✅ PASS without verifiable evidence.
- If a criterion does NOT apply, justify it explicitly (do not skip it).
- The generated `.md` file is the official approval artifact for the service migration.
- The overall result will be ❌ REJECTED if any functional criterion (AC-01 to AC-05) or security criterion (AC-08, AC-14) is ❌.
- The overall result will be ⚠️ APPROVED WITH OBSERVATIONS if all functional criteria are ✅ but there are ⚠️ in non-functional or code quality criteria.
```

# Unit Test Guidelines - Prompt for AI Code Generation

Use this prompt when generating or refactoring unit tests to ensure consistency across the project.

---

## Prompt

```
Generate unit tests following these rules strictly:

### Language
- Everything must be in English: method names, variable names, comments, Javadoc, nested class names, and string literals used as test data.
- Exception: production messages validated in assertions (e.g., error messages returned by the service) must remain exactly as defined in production code, even if they are in another language.

### Structure
- Do NOT use @DisplayName annotations.
- Test method names must follow the pattern: given[Context]_when[Action]_then[ExpectedResult]
- If there are additional preconditions, use: given[Context]_and[AdditionalContext]_when[Action]_then[ExpectedResult]

### Test Body
Each test method must be structured with three clearly separated sections using comments:

// Given — Setup: test data, mocks configuration (when(...).thenReturn(...))
// When  — Execution: single call to the method under test, assigned to a variable
// Then  — Verification: assertions (assertThat, assertEquals) and verifications (verify(...))

For reactive tests (Project Reactor):
- In "// When": assign the Mono/Flux to a variable (e.g., Mono<Customer> result = service.method(...))
- In "// Then": use StepVerifier.create(result) to verify the reactive chain

### Naming Conventions
- Variables: English, descriptive (e.g., requestWithNullId, customerFromStratio, expectedResponse)
- Constants: UPPER_SNAKE_CASE in English (e.g., TRANSACTION_ID, ERROR_HEADER)
- Nested test classes: English, descriptive (e.g., SuccessfulQuery, ServiceErrorTests, BancsClientExceptions)
- Comments and Javadoc: English only

### Coverage Requirements (MANDATORY)
- **Line coverage MUST be greater than 85%.**
- **Branch coverage MUST be greater than 85%.**
- **Method coverage MUST be greater than 85%.**
- Measure coverage with JaCoCo (or the project's configured tool) and attach the report as evidence.
- If coverage is below 85%, you MUST:
  1. Identify the uncovered lines/branches/methods.
  2. Add the missing tests (happy path, error path, edge cases, null/empty inputs, boundary values).
  3. Re-run the coverage report until the threshold is met.
- Exclude only DTOs/records without logic, generated code, and configuration classes. Document every exclusion in the test README or pom.xml/build.gradle with a justified reason.

### Code Duplication Requirements (MANDATORY)
- **Duplicated code in the test suite MUST be 0%.**
- Analyze duplication with SonarQube (or an equivalent tool such as PMD-CPD, jscpd) before delivering the tests.
- If duplication is detected, you MUST refactor using:
  - `@BeforeEach` / `@BeforeAll` for common setup
  - Helper methods or private factory methods for repeated test data
  - Test fixtures / builders (e.g., `CustomerTestDataBuilder`)
  - Parameterized tests (`@ParameterizedTest` with `@ValueSource`, `@CsvSource`, `@MethodSource`) for scenarios that only vary in input/expected output
  - Nested classes with shared context
- Do NOT duplicate:
  - Mock setups that repeat across tests → extract to helpers or `@BeforeEach`
  - Assertion blocks → extract to private `assertCustomer(...)` style helpers
  - Test data literals → promote to constants or builders
- The 0% duplication rule applies both within a single test class and across test classes in the same module.

### Example

@Test
void givenValidInput_whenGetCustomerByIdentification_thenReturnsCustomerSuccessfully() {
    // Given
    when(customerQueryStrategy.query(any(), any())).thenReturn(Mono.just(mockCustomer));

    // When
    Mono<Customer> result = customerService.getCustomerByIdentification(validRequest, null);

    // Then
    StepVerifier.create(result)
            .assertNext(customer -> {
                assertNotNull(customer);
                assertEquals("115678", customer.cif());
                assertEquals("0500563358", customer.identificacion());
            })
            .verifyComplete();

    verify(customerQueryStrategy, times(1)).query(any(), any());
}

### Example with And

@Test
void givenStratioFails_andBancsAvailable_whenQuery_thenReturnsCustomerFromBancs() {
    // Given
    when(stratioPort.getCustomerInfo(any(), isNull(), any()))
            .thenReturn(Mono.error(new RuntimeException("Stratio unreachable")));
    when(bancsPort.getCustomerInfo(any()))
            .thenReturn(Mono.just(customerFromBancs));

    // When
    Mono<Customer> result = strategy.query(validRequest, null);

    // Then
    StepVerifier.create(result)
            .expectNextMatches(customer -> {
                assertEquals("3860119", customer.cif());
                assertThat(customer.dataSource()).isEqualTo(DataSourceTypeEnum.BANCS);
                return true;
            })
            .verifyComplete();

    verify(bancsPort, times(1)).getCustomerInfo(validRequest);
}

### Example for non-reactive tests

@Test
void givenCompleteCustomerCollection_whenToCustomer_thenMapsAllFieldsCorrectly() {
    // Given
    CustomerCollection bancsCustomer = new CustomerCollection("115678", "JUAN PEREZ", ...);

    // When
    Customer result = mapper.toCustomer(bancsCustomer, request);

    // Then
    assertThat(result).isNotNull();
    assertThat(result.cif()).isEqualTo("115678");
    assertThat(result.nombre()).isEqualTo("JUAN PEREZ");
}

### Example for error scenarios

@Test
void givenNullIdentification_whenGetCustomerByIdentification_thenThrowsBusinessValidationException() {
    // Given
    CustomerRequest requestWithNullId = new CustomerRequest(null, "0001");

    // When
    Mono<?> result = service.getCustomerByIdentification(requestWithNullId, null);

    // Then
    StepVerifier.create(result)
            .expectErrorMatches(throwable -> {
                assertThat(throwable).isInstanceOf(BusinessValidationException.class);
                return true;
            })
            .verify();

    verify(strategy, never()).query(any(), any());
}

### Example for removing duplication with parameterized tests

@ParameterizedTest
@CsvSource({
    "null, 0001, BusinessValidationException",
    "'',   0001, BusinessValidationException",
    "123,  null, BusinessValidationException"
})
void givenInvalidInput_whenGetCustomerByIdentification_thenThrowsValidationException(
        String identification, String office, String expectedException) {
    // Given
    CustomerRequest invalidRequest = new CustomerRequest(identification, office);

    // When
    Mono<?> result = service.getCustomerByIdentification(invalidRequest, null);

    // Then
    StepVerifier.create(result)
            .expectError(BusinessValidationException.class)
            .verify();
}

### Delivery Checklist (before submitting the tests)

- [ ] All test method names follow given_when_then naming
- [ ] Every test has explicit // Given / // When / // Then sections
- [ ] All variables, comments, constants and nested classes are in English
- [ ] Line / branch / method coverage > 85% (JaCoCo report attached)
- [ ] Code duplication = 0% (SonarQube / jscpd report attached)
- [ ] No @DisplayName annotations used
- [ ] Shared setup extracted to @BeforeEach / builders / helpers
- [ ] Repeated scenarios converted to @ParameterizedTest where applicable
- [ ] Tests pass locally and in CI
```

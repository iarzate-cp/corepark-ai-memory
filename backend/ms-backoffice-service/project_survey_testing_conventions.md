---
name: Testing patterns learned building the /v1/surveys admin CRUD suite
description: JUnit 4 + Mockito recipes used for the 58-test survey admin suite. Extends smstemplate/compensations with survey-specific idioms (ArgumentCaptor for tenant binding, thenAnswer for private records, verify-never for ordering invariants)
type: project
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
Convention followed for the 58-test suite added on 2026-07-02. Extends the existing pattern from `SMSTemplateDaoTest` / `SMSTemplateServiceTest` / `SMSTemplateControllerTest` — this file records the survey-specific idioms so future test-writing sessions can copy them.

## Baseline setup

- **JUnit 4** (`org.junit.Test`, `Assert`, `Before`) with `junit-vintage-engine` on classpath.
- `@RunWith(MockitoJUnitRunner.class)` — no `MockitoExtension`, no JUnit 5.
- **Post-Wave 2 (2026-07-07): tests assert on `ApiException` + `ErrorCode` for admin slice**, not `ServiceException` + `MessageCode`. Current suite: **59 tests, all green**. The 2 previous tests for `BindingResult.hasErrors()` in the controller were dropped — that responsibility moved to `GlobalExceptionHandler` and belongs in a `@WebMvcTest`, not a unit test.
- Mocks declared as fields, passed to SUT constructor:

      private final SurveyAdminDao dao = mock(SurveyAdminDao.class);
      private final SurveyAdminService svc = new SurveyAdminServiceImpl(dao);

## DAO test pattern (`SurveyAdminDaoTest`, 19 tests)

- Mock `NamedParameterJdbcTemplate`.
- **Duplicate SQL constants in the test class** — mirror check. `eq(SQL_STRING)` matcher fires against the local copy; if production SQL changes and the test copy doesn't, tests fail loudly.
- `update` / `queryForObject` / `query` calls mocked with `when().thenReturn(...)`; assertions on the returned value.
- `batchUpdate` mocks return null/empty by default (fine); use `verify()` to assert the call happened.

### Testing private records via RowMapper

`SurveyAdminDaoImpl` has `private record InsertedSurvey(long id, UUID uuid)` returned from `queryForObject(SQL, params, RowMapper)`. Can't reference the record type from tests. Trick — let the actual production RowMapper produce it:

    when(namedParameterJdbcTemplate.queryForObject(anyString(), any(MapSqlParameterSource.class),
            any(RowMapper.class))).thenAnswer(invocation -> {
        RowMapper<?> mapper = invocation.getArgument(2);
        ResultSet rs = mock(ResultSet.class);
        when(rs.getLong("id")).thenReturn(1L);
        when(rs.getObject("uuid", UUID.class)).thenReturn(someUuid);
        return mapper.mapRow(rs, 0);
    });

The test never sees `InsertedSurvey`'s type; the production code constructs it from the mocked ResultSet.

### Verifying SQL param binding (tenant params)

For methods widened to `(op, loc, uuid)` in the cross-tenant fix, verify the actual params bound to the SQL — not just that the method was called:

    ArgumentCaptor<MapSqlParameterSource> captor = ArgumentCaptor.forClass(MapSqlParameterSource.class);
    when(jdbc.update(eq(QRY_DEACTIVATE_SURVEY), captor.capture())).thenReturn(1);
    dao.deactivateSurvey(operatorId, locationId, surveyUuid);
    Assert.assertEquals(operatorId, captor.getValue().getValue("operatorCompanyId"));
    Assert.assertEquals(locationId, captor.getValue().getValue("parkingLocationId"));
    Assert.assertEquals(surveyUuid, captor.getValue().getValue("surveyUuid"));

Applied to `deactivateSurvey`, `reactivateSurvey`, `getSurveyDetail` — the three uuid-based DAO methods that carry tenant scope.

### ResultSetExtractor (cartesian-join queries)

`getSurveyDetail` uses `query(sql, params, ResultSetExtractor)` (not `RowMapper`). Mock with `any(ResultSetExtractor.class)`:

    when(jdbc.query(eq(QRY_GET_SURVEY_DETAIL), any(MapSqlParameterSource.class),
            any(ResultSetExtractor.class))).thenReturn(surveyDetail);

Empty result path:

    when(jdbc.query(eq(SQL), any(MapSqlParameterSource.class), any(ResultSetExtractor.class)))
        .thenThrow(new EmptyResultDataAccessException(1));
    // → DAO catches this and rethrows as ServiceException(ERR_SURVEY_001)

## Service test pattern (`SurveyAdminServiceTest`, 28 tests)

- Mock the DAO; SUT via `new SurveyAdminServiceImpl(dao)`.
- Test happy paths + each declared `ERR_SURVEY_*` branch.
- Use `verify(dao, never()).method(...)` to encode ordering invariants — for example: "validation must run before deactivate":

      verify(dao, never()).deactivateSurvey(operatorId, locationId, surveyUuid);

- For `ApiException` with a specific code, use a helper (post-Wave 2 pattern for admin slice):

      private static void assertApiException(ErrorCode expected, Runnable action) {
          try {
              action.run();
              Assert.fail("Expected ApiException with code " + expected.getCode());
          } catch (ApiException e) {
              Assert.assertEquals(expected, e.getErrorCode());
          }
      }

  `ApiException` exposes `getErrorCode()` directly, so we compare the enum instance (stronger than string comparison). For the consumer slice still on legacy, keep the old `assertServiceException(MessageCode, Runnable)` variant that asserts on `e.getMessage()`.

- For "tenant IDs are forwarded" contract tests: just call the service and `verify(dao).method(op, loc, uuid)` with exact args. Mockito's exact-arg matching enforces the contract.

## Controller test pattern (`SurveyAdminControllerTest`, 9 tests)

- Instantiate controller directly. Mock the service + `BindingResult`.
- Assert `ResponseEntity` non-null and `HttpStatus` matches:

      Assert.assertEquals(HttpStatus.OK, result.getStatusCode());

- For binding errors:

      FieldError fieldError = new FieldError("SurveyWriteRequest", "languageIds",
          "'languageIds' must have at least one entry");
      when(bindingResult.hasErrors()).thenReturn(true);
      when(bindingResult.getFieldErrors()).thenReturn(List.of(fieldError));
      // then invoke controller
      Assert.assertEquals(HttpStatus.BAD_REQUEST, result.getStatusCode());
      Assert.assertTrue(result.getBody().getErrors().contains(fieldError.getDefaultMessage()));
      verify(service, never()).createSurvey(...);

- **Do NOT test JSR-380 depth here** — that's `@WebMvcTest` territory. This convention only checks that `bindingResult.hasErrors()` triggers the right branch, not that specific annotations produce specific errors.

## Handler test pattern (`RestExceptionHandlerControllerTest`, 2 tests, new file)

Precedent set by this session — no existing handler test class before.

- Instantiate the handler directly (no `@WebMvcTest` scaffolding).
- Call the handler method with a mocked exception.
- Assert response `HttpStatus` matches the mapped `MessageCode.status`.
- Assert response body's `code` field matches `MessageCode.getCode()`.

Example:

    RestExceptionHandlerController handler = new RestExceptionHandlerController();
    DuplicateKeyException exc = new DuplicateKeyException("duplicate key value violates unique constraint");
    ResponseEntity<Response> result = handler.handleDuplicateKeyException(exc);
    Assert.assertEquals(HttpStatus.CONFLICT, result.getStatusCode());
    Assert.assertEquals(MessageCode.ERR_SURVEY_005.getCode(), result.getBody().getCode());

## JaCoCo ad-hoc (no pom mutation)

    ./mvnw org.jacoco:jacoco-maven-plugin:0.8.12:prepare-agent test \
        org.jacoco:jacoco-maven-plugin:0.8.12:report \
        -Dtest='SurveyAdmin*,RestExceptionHandlerControllerTest'

Report at `target/site/jacoco/`. Filter the CSV to survey/common classes with `awk`. Coverage numbers appear in `jacoco.csv` per class.

## What is NOT testable under this convention (intentional trade-offs)

- **SQL correctness** — mocks bypass the real JdbcTemplate. SQL bugs surface in QA manual only.
- **Cartesian-join extractor internals** (`extractSurveyDetail`, RowMappers) — never executed under mocks. Would need Testcontainers.
- **`@Transactional` rollback** — Spring runtime behavior, needs integration tests.
- **JSR-380 depth** — needs `@WebMvcTest`.
- **Concurrent race conditions** — needs load tests.

## Reference test classes (canonical examples in the repo)

- `SMSTemplateDaoTest`, `SMSTemplateServiceTest`, `SMSTemplateControllerTest` — simple baseline.
- `CompensationsDaoTest`, `CompensationServiceTest` — moderate SQL + business logic.
- `SurveyAdminDaoTest`, `SurveyAdminServiceTest`, `SurveyAdminControllerTest`, `RestExceptionHandlerControllerTest` — the pattern extended with `ArgumentCaptor`, `thenAnswer` on `RowMapper`, `verify-never` for ordering, and the handler test class.

## Requires

- JDK 21 (macOS: `/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`). JDK 26 breaks Lombok annotation processing on this codebase.

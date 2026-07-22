---
name: ms-backoffice-service has its own CLAUDE.md with backend conventions
description: The ms-backoffice-service repo carries a CLAUDE.md at its root that dictates backend conventions — Lombok constructor injection, @Slf4j logging, ApiResponse<T>/ErrorCode envelope for new code, OpenAPI docs required, JUnit4+Mockito tests. Read it before writing or reviewing Java code in that repo
type: reference
originSessionId: a2019106-3f08-4f27-8756-0bb019dfebd1
---
# Backoffice backend conventions

There is a CLAUDE.md at `~/Dev/Back-End/ms-backoffice-service/CLAUDE.md` that establishes the coding conventions for the backoffice service. Because the working directory of most sessions is `~/Dev/frontend-commerce`, this backend CLAUDE.md is NOT automatically loaded — read it explicitly when starting Java work in that repo.

## Key rules (as of 2026-07-06)

- **DI:** Lombok constructor injection with `private final` + `@RequiredArgsConstructor`. Never `@Autowired` field injection, never a hand-written constructor.
- **Logging:** `@Slf4j` on the class + `log.info(...)`. Never `private static final Logger LOGGER = LoggerFactory.getLogger(...)`.
- **Beans:** Lombok accessors — `@Getter @Setter` on request/response/row beans, `@NoArgsConstructor` on row beans (BeanPropertyRowMapper needs no-arg + setters).
- **DAO:** One operation per method. Use `NamedParameterJdbcTemplate` + `MapSqlParameterSource` (named params, never string concat). Map rows with `BeanPropertyRowMapper` (relies on snake_case → camelCase). SQL in `private static final String QRY_*` text blocks.
- **Controllers (new code):** `ApiResponse<T>` envelope + `ErrorCode` enum. Located at `com.corepark.common.response.ApiResponse` and `com.corepark.common.exception.ErrorCode`. Return `ResponseEntity.status(response.getStatus()).body(response)` — the status comes from the response, not from parsing a code back to `HttpStatus`. Reference slice: `crm/aggregators/controller/RulesController`.
- **Controllers (legacy):** `Response` + `MessageCode` from `com.corepark.backoffice.common.*` or `com.corepark.crm.commons.*`. Match this only when editing legacy slices — do NOT introduce it in new code.
- **OpenAPI docs required** on every REST endpoint. `@Tag` on class, `@Operation` + one `@ApiResponse` per documented status. **Externalize the JSON examples** into a `<Feature>ApiExamples` class in the controller package — final class with private no-args constructor holding `public static final String` text-block constants. Reference: `backoffice/parkinglayout/ParkingLayoutApiExamples`.
- **Validation:** `@Valid @RequestBody` + `BindingResult`. On `hasErrors()`, return `ApiResponse.error(ErrorCode.VALIDATION_FAILED, <field errors>)`.
- **Request context:** Read `Operator-Id` / location headers via `RequestContext` (`com.corepark.crm.commons.beans.request.RequestContext`) — use its builder + `requireOperatorCompanyId()` / `requireParkingLocationId()`.
- **Domain exceptions:** services throw them, `com.corepark.common.exception.GlobalExceptionHandler` renders as `ApiResponse`. Do not hand-catch `DataAccessException` per method in new code.
- **Package layout:** `com.corepark.backoffice.<feature>.*` (or `com.corepark.crm.<area>.<feature>.*` for CRM). Slice = `controller/` · `service/` + `service/impl/` · `dao/` + `dao/impl/` · beans. Backoffice uses `bean/` (singular) subtree (`request`, `response`, `row`, `entity`, `dto`, `enumerators`); CRM uses `beans/` (plural).
- **Reuse existing slices** — don't create parallel `*Controller`/`*Service`/`*Dao` classes for a domain that already has one. New package only for a genuinely new domain.
- **Testing:** JUnit 4 + Mockito (`@RunWith(MockitoJUnitRunner.class)`). One test per layer (`*ControllerTest`, `*ServiceTest` / `*ServiceImplTest`, `*DaoTest` / `*DaoImplTest`). Focus on DAOs and services; presentational glue is covered by integration.
- **Branching:** feature branches off `main` (never off `feature/staging` or another feature branch). PRs target `main`, always in English.
- **JVM setup:** `source "$HOME/.sdkman/bin/sdkman-init.sh" && sdk use java 21.0.8-amzn` before Maven/Java commands. **Java 21 is a hard requirement, not a preference.** With Java 24+ (Israel's system default is OpenJDK 26 as of 2026-07-06), the stricter access-restriction policies silently break Lombok's annotation processor, cascading into 100+ "cannot find symbol log" / "cannot find symbol builder()" errors across the whole codebase. If a build starts producing that pattern, the first thing to check is `java -version` and `$JAVA_HOME`. Corretto 21 works; `export JAVA_HOME=$(/usr/libexec/java_home -v 21)` is enough for a single shell session (Maven's wrapper reads `$JAVA_HOME`, not the shell's `java` in PATH).
- **Date-time filters:** `OffsetDateTime` + `@DateTimeFormat(iso = ISO.DATE_TIME)`. Never date-only for filters against `timestamptz` columns.

## When it applies

- Any code being added under `com.corepark.backoffice.*` or `com.corepark.crm.*`.
- Editing existing legacy code: match the file's local style (per the doc's opening paragraph), don't retrofit.
- Reviewing PRs in the backoffice repo — enforce these rules.

## Where to find it

`~/Dev/Back-End/ms-backoffice-service/CLAUDE.md`, tracked in the repo, ~276 lines. Was last touched in commit `627aea1 docs: add DAO single-operation rule and OpenAPI examples convention` on 2026-06-29 range.

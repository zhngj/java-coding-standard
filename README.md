# Java coding standard

*A formatter settles formatting, so nobody has to. This standard covers what a tool can't decide: how we handle errors, nulls, configuration, threads and tests, and which parts of Java 17 we lean on. We adopt established standards by reference and write down only our own decisions.*

| Applies to | Owner | Platform | Adopted | Next review | Status |
|---|---|---|---|---|---|
| All Java services (API pod) | API pod leader | Java 17 · Spring Boot 3.x · Maven | October 2026 | November 2026 | Draft for team review |

## How this standard works

Three layers, each catching what the one before can't:

1. **Formatting is automated.** `google-java-format` decides, and the build fails on unformatted code. Reviewers never comment on whitespace, wrapping or import order. If the check passed, formatting is done.
2. **Mechanical rules are tool-checked.** SonarQube, already wired into the EDP pipeline, flags bugs, smells and vulnerabilities. Its quality gate applies to new code.
3. **Judgment is reviewed.** The rules below are what reviewers look for, alongside the pull request standards.

A reference implementation accompanies this page: one feature slice that exercises every rule below and proves it with its own build (formatter check, tests, coverage). See *Java coding standard: reference example*.

### Adopted by reference

| Area | We adopt | Checked by |
|---|---|---|
| Formatting, imports, naming, Javadoc format | [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html), in its 4-space (AOSP) variant | google-java-format via Spotless |
| Programming practices | Google Java Style §6; *Effective Java* (3rd ed.) breaks ties on design questions | Review |
| Bugs, smells, vulnerabilities | SonarQube "Sonar way" Java profile and quality gate | EDP pipeline |
| Framework conventions | Spring Boot reference documentation | Review |
| Secure coding | OWASP Java cheat sheets; dependency scanning in the pipeline | Review · Veracode |

Where this page and a referenced source disagree, this page wins. Where neither says anything, *Effective Java* wins.

### What changes for us

| Today | Standard |
|---|---|
| Hand-formatted, no line limit, wildcard imports | Formatter-enforced Google style, 100 columns, no wildcards |
| `@Autowired` on fields | Constructor injection, `final` fields |
| Scattered `@Value("${…}")` lookups | Typed `@ConfigurationProperties` records |
| `catch (Exception e)` returning `null`, `""` or `false`; bare `RuntimeException` | Specific exceptions; one `@RestControllerAdvice` producing `ProblemDetail` |
| DTOs as mutable classes with setters | Records |
| Hand-built threads and executors | Spring-managed executors and `@Scheduled` |
| `RestTemplate` and OkHttp side by side | `RestClient`, one client per service |
| No tests; Sonar excludes `**/*.java` | Tests with every behavior change; Sonar scanning with a new-code gate |

## 1. Formatting

- **Google Java Style, applied by google-java-format** in AOSP mode: 4-space indent, 100-column lines, Google import order, no wildcard imports. The formatter's output is the specification. There are no hand-written formatting rules beyond it.
- **Enforced in the build.** Spotless runs `spotless:check` during `verify`, so an unformatted file fails CI. `mvn spotless:apply` fixes a whole repo locally.
- **Pin the formatter pair.** Spotless and google-java-format move independently and not every pair works: 3.10.4 with 1.37.0 fails on every file, with 1.36.1 it passes. Bump them together, run the build, then commit.
- **IDE setup.** Install the google-java-format plugin in IntelliJ and enable it for the project. Set *class count to use import with `*`* to 999 so wildcards never appear. The shared `.editorconfig` is committed in every repo.
- **One reformat PR per repo, no logic.** Run `spotless:apply`, merge that PR first, then add its squash commit hash to `.git-blame-ignore-revs`. Git and GitHub honor the file, so blame skips the reformat.
- **Reviewers do not comment on formatting.** The check either passed or it didn't.

> **Decision needed: indent width.** Pure Google style is 2 spaces; AOSP is the same style at 4. Both are built-in google-java-format modes. Our code is 4-space today, like most of the Spring ecosystem, so this draft picks AOSP. Switching is one word in the config.

```xml
<plugin>
  <groupId>com.diffplug.spotless</groupId>
  <artifactId>spotless-maven-plugin</artifactId>
  <version>3.10.4</version>
  <configuration>
    <java>
      <googleJavaFormat>
        <version>1.36.1</version>
        <style>AOSP</style>
        <reflowLongStrings>true</reflowLongStrings>
      </googleJavaFormat>
    </java>
  </configuration>
  <executions>
    <execution>
      <goals><goal>check</goal></goals>
    </execution>
  </executions>
</plugin>
```

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.java]
indent_style = space
indent_size = 4
max_line_length = 100
```

## 2. Language: Java 17

- **Records for data.** Request and response DTOs, configuration properties, events and value objects are records. Validation annotations go on the components. Use a class with setters only when a framework forces it, and say so in a comment.
- **Fields are `final` unless they must change.** No static mutable state. Don't sprinkle `final` on locals and parameters; it adds noise, not safety.
- **Sealed interfaces for closed sets.** A type with a fixed set of variants (result kinds, routing targets, progress steps) is a `sealed interface` with record implementations. Dispatch with `instanceof` patterns or a method on the interface. Pattern matching in `switch` is a preview feature in 17, and preview features are never enabled.
- **Switch expressions, arrow form.** No fall-through. Over an enum, leave out `default` so the compiler fails when a constant is added and points at every switch to update.
- **`var` only when the type is on the same line.** `var client = new DatabricksClient(cfg)` reads fine; `var result = service.process(x)` hides the type. When in doubt, write the type.
- **Text blocks** for multi-line SQL, JSON and templates.
- **Streams for short pipelines, loops for everything else.** No side effects inside a stream; collect with `toList()`. If a pipeline needs a comment to follow, write a loop.
- **Collections.** `List.of` / `Map.of` / `Set.of` for fixed content. Declare interfaces (`List`, not `ArrayList`). Never return `null` for a collection; return an empty one. Never mutate a collection you were given.
- **Time.** `java.time` only, never `Date` or `Calendar`. `Instant` or `OffsetDateTime` for timestamps; `LocalDate` only where the zone truly doesn't matter. Code that reads the current time takes a `Clock` so tests can control it.
- **Strings.** `equals`, never `==`. Charsets are always explicit (`StandardCharsets.UTF_8`). `isBlank()` over `trim().isEmpty()`; `formatted()` or a text block over long concatenation.
- **Never:** raw types; `System.out` / `System.err`; `System.exit`; `finalize`; Java serialization for transport; `@SuppressWarnings` without a one-line reason beside it.

## 3. Nulls and Optional

- **Null never crosses a boundary silently.** A method that may return or accept `null` says so with `@Nullable` (Spring's while we are on Boot 3, JSpecify when we move to Boot 4). Anything unannotated is non-null.
- **`Optional` is a return type**, for results that may legitimately be absent. Never a field, parameter or collection element, and never `Optional<Collection>`: return an empty collection instead.
- **Consume it without `get()`.** `orElseThrow()`, `orElse`, `map`, `ifPresentOrElse`. `isPresent()` followed by `get()` is a null check in disguise.
- **Fail at construction.** Constructor injection makes dependencies non-null by construction. For anything else, `Objects.requireNonNull(x, "x")` at the entry point, not a null check deep inside.
- **Validate at the edge, trust inside.** Bean Validation (`@Valid`, `@NotNull`, `@NotBlank`) on request DTOs. Past the controller, code assumes valid, non-null data.

## 4. Exceptions and error responses

- **Unchecked by default.** Our exceptions extend `RuntimeException`. We don't define checked exceptions, and we translate the JDK's (`IOException`, `SQLException`) where they occur, so `throws IOException` never appears on a service method.
- **Throw something specific.** Each service has a small hierarchy under one base exception, for example `BundleDeployException` or `DownstreamUnavailableException`. Never `throw new RuntimeException(...)` or `throw new Exception(...)`.
- **Keep the cause and add context.** `throw new BundleDeployException("deploying bundle " + id, e)`: what was being attempted, plus the original exception. Converting an exception to `e.getMessage()` throws the stack trace away.
- **Catch narrowly, where you can act.** Catch the specific type you can retry, fall back from or translate. `catch (Exception e)` is allowed only at a boundary, meaning the controller advice or the top of a background task, and there it logs the exception object and translates it.
- **No sentinel returns.** Returning `null`, `""`, `false` or `"unknown"` because something threw hides the failure from the caller and from monitoring. Let it propagate.
- **One place maps exceptions to HTTP.** A single `@RestControllerAdvice` per service turns exceptions into `ProblemDetail` responses (RFC 9457). Enable `spring.mvc.problemdetails.enabled=true`. Controllers never `try/catch` to build an error `ResponseEntity`.
- **Don't leak internals.** An error response carries a stable `type` and `title`, a status, a safe `detail` and a correlation id. Raw `e.getMessage()`, stack traces and class names stay in the logs.
- **Log or rethrow, not both.** The layer that handles the exception logs it. Everything below adds context and rethrows.
- **Close what you open.** try-with-resources for every `AutoCloseable`: JDBC, streams, JGit, processes.
- **Exceptions are not control flow.** Check first; don't catch to implement a branch.

## 5. Logging

- **SLF4J via `LoggerFactory`:** `private static final Logger logger = LoggerFactory.getLogger(Foo.class);`. Lowercase `logger`, because Google style §5.2.4 classes a Logger as a non-constant. One name everywhere.
- **Parameterized messages, exception last.** `logger.warn("Deploy {} failed for workspace {}", id, ws, e)`. Never concatenate, and never pass `e.getMessage()` into a placeholder instead of `e`; that drops the stack trace.
- **Levels mean something.** ERROR: a request or task failed and someone may need to act. WARN: degraded but handled (a retry succeeded, a fallback was used). INFO: one line per business event or lifecycle transition. DEBUG: diagnostics, off in production.
- **Never log secrets or customer data.** Tokens, PATs, connection strings, credentials, request bodies that may carry them, and personal data never reach a log at any level. A record or class that holds a secret overrides `toString()` to redact it; a record's generated `toString()` prints every component.
- **Context goes in MDC, not in the message.** Request id, task id and workspace id are set once in a filter or at task start and appear on every line automatically. No `[Workspace]`-style prefixes in messages. Managed executors carry MDC across threads (see section 6).
- **Structured output.** Boot 3.4+ emits JSON with `logging.structured.format.console=ecs`. Switch it on once the log platform ingests JSON.

## 6. Spring Boot

- **Constructor injection only.** One constructor, `private final` fields, no `@Autowired` on fields or setters (Spring wires a single constructor without the annotation). Dependencies become visible, testable and never null.
- **Typed configuration.** Related settings form a `@ConfigurationProperties` record with a prefix, registered with `@ConfigurationPropertiesScan` and checked with `@Validated`. `@Value("${…}")` is for a single one-off at most. Missing or invalid configuration fails startup, not the first request.
- **Use the container's beans.** Inject the managed `ObjectMapper`, `RestClient.Builder`, `JdbcClient` and executors. `new ObjectMapper()` in production code bypasses our Jackson configuration and is not allowed.
- **Thin controllers.** A controller validates (`@Valid`), delegates to a service and returns a DTO. No business logic, no `try/catch`, no HTTP clients, no hand-parsed JSON. Services know nothing about HTTP.
- **Records at the API boundary.** Request and response types are records in the feature package. Internal and persistence types are never serialized directly. API documentation lives in OpenAPI annotations (`@Operation`, `@Schema`) and is not repeated in Javadoc.
- **One HTTP client.** `RestClient`, built from the injected builder, with explicit connect and read timeouts (`spring.http.client.*`). `RestTemplate` is maintenance-only and OkHttp is used only where a library requires it. Neither gets new call sites.
- **Background work runs on managed executors.** A named `ThreadPoolTaskExecutor` bean per workload: bounded queue, rejection policy, graceful shutdown, and a `TaskDecorator` that propagates MDC. Use `@Async("name")` or inject it. Periodic work (heartbeats, keep-alives) is `@Scheduled`; polling and retry use a scheduled executor or a retry library. Never `new Thread`, `new ThreadPoolExecutor`, `Executors.new*` or `Thread.sleep` in application code. They escape lifecycle, shutdown, metrics and MDC.
- **Prefer immutability to locks.** Each `synchronized` or `volatile` carries a comment naming the invariant it protects. Prefer `java.util.concurrent` types and confining state to one thread.
- **Persistence through Spring.** `JdbcClient` with named parameters. `SQLException` doesn't leave the repository class; Spring translates it to `DataAccessException`.
- **Fail at startup, not at the first request.** Check external prerequisites in a startup validator where feasible, so a misconfigured pod fails its readiness probe instead of failing a customer.

## 7. Security

Our services hold customer credentials and act on customer workspaces. The bar is a bank's.

- **Secrets come from configuration, never from code.** No tokens, keys or connection strings in source, tests, Javadoc or example payloads, including `ghp_…`-style placeholders. Types that carry secrets redact `toString()`.
- **Outside input is hostile until validated.** Bean Validation on every request DTO; size limits on bodies and lists; allow-lists for anything enumerable.
- **Paths and commands.** A file path or command argument derived from a request is normalized and confined to an allow-listed base directory. Processes start from argument arrays, never a shell string.
- **SQL is parameterized.** Always. No string-built queries, including for identifiers.
- **Deserialization.** Jackson only, default typing off. Never `ObjectInputStream` on external data. `SecureRandom` for anything security-relevant.
- **Dependencies are scanned and patched.** Dependency scanning runs in the pipeline. A dependency with a known CVE is bumped within the sprint, in its own PR.
- **Deny by default.** Security configuration permits specific endpoints. Actuator endpoints beyond health and info are not public.

## 8. Testing

- **Every behavior change ships with tests that would fail without it.** The PR's *How was this verified* section names them. A PR with no tests says why.
- **Stack: JUnit 5, AssertJ, Mockito**, all from `spring-boot-starter-test`. No extra pins; `mockito-inline` is obsolete, Mockito 5 mocks final classes by default. `assertThat` over `assertEquals` for readable failures.
- **Pyramid.** Unit tests (plain JUnit, no Spring context) for services, parsers and resolvers carry most of the weight. `@WebMvcTest` covers controllers: mapping, validation errors, and the error-response shape from the advice. `@SpringBootTest` is for wiring checks, one or two per service. External systems (Databricks CLI, GitHub, Git, JDBC) sit behind an interface so tests substitute a fake. End-to-end journeys belong to the test automation strategy.
- **Test profile.** Every Spring-context test (`@SpringBootTest`, `@WebMvcTest`, `@JdbcTest`) activates the `test` profile, which supplies a value for every setting production takes from the environment. Slices evaluate production configuration too, and an unresolved `${…}` placeholder fails them.
- **Naming:** `methodName_condition_expectedResult`, for example `resolveTarget_unknownTenant_throwsNotFound`. Google style allows underscores in test method names for exactly this.
- **Shape.** One behavior per test. Arrange, act and assert separated by blank lines. No conditionals or loops in a test body. Assert on outcomes; verify a mock interaction only when the interaction is the behavior (a callback sent, a command run).
- **Deterministic.** No `Thread.sleep`, no real clock (inject `Clock`), no network, no filesystem outside `@TempDir`, no shared mutable state or order dependence between tests.
- **Coverage.** JaCoCo already produces the report the pipeline reads. The Sonar quality gate requires 80% coverage on new code, the Sonar way default. There is no retroactive target for existing code.
- **Test code is code.** Same formatter, same review. Fixtures come from small builders or text-block JSON, not from a giant shared setup.

> **Decision needed: when the coverage gate blocks.** With no tests today, a blocking new-code gate stops PRs from the first day. Options: block from day one, or report-only for the first month and block from the next review.

## 9. Structure, naming and comments

- **Google naming rules apply:** `UpperCamelCase` types, `lowerCamelCase` members, `UPPER_SNAKE_CASE` constants, no Hungarian prefixes, no `I`-prefixed interfaces.
- **Package by feature.** A feature package (`catalog`, `precheck`, `routing`, …) holds its controller, service, DTOs and exceptions; `config` holds cross-cutting wiring. No `controller` / `service` / `dto` layer packages for new code.
- **Role suffixes:** `*Controller`, `*Service`, `*Client` (talks to an external system), `*Properties` (configuration), `*Request` / `*Response` (API DTOs), `*Exception`. No `*Impl`: with one implementation, drop the interface or name the class by what makes it different, as `StaticRoutingResolver` does. `*Util` only for a `final` class of static methods with a private constructor.
- **Member order:** constants, fields, constructors, public methods, private methods, nested types. A private helper sits below the first method that uses it.
- **Size has a bound.** Sonar's cognitive-complexity limit (15 per method) is enforced. A class that needs scrolling to find its responsibilities has more than one; split along them when you touch it.
- **Comments say why.** Intent, constraints and non-obvious decisions, never a restatement of the code. Javadoc on every public type stating its responsibility; on public methods only when the signature doesn't say it all. No Javadoc that repeats the signature.
- **No commented-out code.** Git has it. A `TODO` names a ticket, `// TODO(TDVIP-123): …`, or it doesn't merge.

> **Decision needed: package layout.** Our two services differ today, one by feature and one by layer. This draft picks by feature: a change usually lands in one package, and the package list shows what the service does. Existing layer packages migrate as classes are touched.

## 10. Build and dependencies

- **Maven, Java 17, the Spring Boot parent.** The Boot BOM manages versions; never pin a version the BOM already manages. Stay on a Spring Boot line that is in open-source support and take patch releases as they come.
- **No Lombok.** Records, constructor injection and `LoggerFactory` cover what it would do, without an annotation processor in the build.
- **A shared parent POM.** Both services carry near-identical POMs. A `tdvip-parent` holds plugin configuration (Spotless, JaCoCo, Surefire) and third-party versions, so a bump happens once.
- **Green locally before pushing.** `mvn -B verify` runs formatting, compilation and tests. It passes before a PR asks for review.
- **Every dependency earns its place.** Adding one needs a reason in the PR; remove what nothing uses. `spring-boot-devtools` stays out of the runtime image.
- **Final features only.** Preview features are never enabled. Moving to Java 21 is a team decision recorded on this page.

## 11. Enforcement and rollout

Per repository, in this order:

1. Commit `.editorconfig` and the Spotless plugin. Open the reformat PR (`spotless:apply`, nothing else), merge it, record its hash in `.git-blame-ignore-revs`.
2. Remove `sonar.exclusions: "**/*.java"` from the component CI config so Sonar analyzes Java. Confirm the quality gate is the Sonar way default on new code.
3. From then on, new and changed code follows this page in full. When a PR touches a class, bring the touched parts up to standard in the same PR if that is small (field injection to constructor, a broad catch to a specific one). Otherwise open a ticket and link it from the PR.
4. Once the build is green under steps 1 and 2, add Error Prone for compile-time bug patterns.

Who catches what:

| Caught by | Examples |
|---|---|
| Formatter | indentation, wrapping, import order, braces, blank lines |
| Sonar | naming, empty or broad catch, unused code, cognitive complexity, known vulnerabilities, new-code coverage |
| Reviewer | exception design, sentinel returns, logging content, thread usage, test quality, naming meaning, package placement |

**Deviations.** `@SuppressWarnings("rule") // reason` on the smallest possible scope, reason on the same line, approver's agreement in the PR. A rule that keeps getting suppressed is a rule to revisit.

## 12. Changing this standard

Propose a change by commenting on this page or in the team channel: the rule, the reason, an example. Pod leaders decide. The changelog below records what changed and why.

- **October 2026:** first draft for team review.
- **October 2026:** reference implementation added; formatter versions corrected to a verified pair.

*Companion standards: Pull request standards · Design doc standard · Test automation and deployment strategy. Reference implementation: Java coding standard: reference example.*

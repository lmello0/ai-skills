---
name: java-refactor
description: >
  Use this skill for ANY task involving legacy Java codebases on Windows, Linux, or macOS:
  analyzing bad practices, detecting code smells, finding critical bugs, inspecting Spring Boot
  anti-patterns, HTTP/REST design violations, over-engineered code, high cyclomatic complexity,
  and performing complete safe refactors backed by TDD. Trigger this skill whenever the user
  mentions: refactoring Java code, analyzing Java for issues, cleaning up legacy Java, Spring Boot
  problems, REST API anti-patterns (POST acting as GET, etc.), over-engineering, complexity too
  high, improving Java code quality, finding Java bugs or anti-patterns, writing JUnit tests
  before refactoring, or anything involving "legacy Java", "Java code smells", "Java tech debt",
  "safe Java refactor", or "Java TDD refactor". Works on Windows (cmd/PowerShell), Linux, and
  macOS — all commands are provided for every platform. This is the authoritative guide for the
  full analyze → test → refactor → verify cycle on Java projects.
---

# Java Legacy Codebase Refactor Skill

A complete, phase-by-phase workflow for safely analyzing, testing, and refactoring legacy Java codebases without breaking existing behavior. Fully cross-platform: **Windows (cmd / PowerShell), Linux, and macOS**.

---

## Platform Notes

All shell commands are given in three variants where they differ. Use the one matching your environment:

| Platform | Shell | Maven wrapper | Gradle wrapper |
|----------|-------|---------------|----------------|
| Linux / macOS | bash | `./mvnw` | `./gradlew` |
| Windows cmd | cmd | `mvnw.cmd` | `gradlew.bat` |
| Windows PowerShell | pwsh | `.\mvnw` | `.\gradlew` |

> **Windows gotcha**: Use `findstr` instead of `grep`, `dir /s /b` instead of `find`, and `type` instead of `cat`. All per-command alternatives are shown inline.

---

## Overview

The workflow has **4 phases** executed in strict order:

```
Phase 1: ANALYSIS     → Catalog all issues (smells, bugs, anti-patterns, complexity, Spring Boot, HTTP)
Phase 2: TEST CAPTURE → Write characterization + unit tests BEFORE touching code
Phase 3: REFACTOR     → Incrementally fix issues, running tests after every change
Phase 4: VERIFY       → Final audit, coverage check, regression confirmation
```

**Analysis covers**:
- Classic code smells & bugs (NPE, resource leaks, swallowed exceptions, etc.)
- Spring Boot anti-patterns (God `@Service`, anemic controllers, missing `@Transactional`, etc.)
- HTTP / REST design violations (POST-as-GET, wrong status codes, chatty APIs, etc.)
- Over-engineered code (unnecessary abstraction, premature generalization, pattern soup)
- High cyclomatic & cognitive complexity

> **Golden Rule**: Never modify production code until Phase 2 tests are green. Never finish Phase 3 without all tests still green.

---

## Phase 1: Analysis

**Goal**: Produce a complete issue inventory before writing a single line of fix.

### 1.1 — Codebase Reconnaissance

**Linux / macOS**
```bash
find . -name "*.java" | wc -l                          # total Java files
find . -name "*.java" | head -60                       # sample filenames
ls pom.xml build.gradle build.gradle.kts 2>/dev/null  # build system
find . -path "*/test*/*.java" | wc -l                  # existing test count
grep -r "sourceCompatibility\|<java.version>\|release" pom.xml build.gradle 2>/dev/null | head -10
```

**Windows cmd**
```cmd
dir /s /b *.java | find /c /v ""
dir /s /b *.java
if exist pom.xml echo Maven & if exist build.gradle echo Gradle
dir /s /b *.java | findstr "\\test\\"
findstr /s /i "sourceCompatibility java.version" pom.xml build.gradle
```

**Windows PowerShell**
```powershell
(Get-ChildItem -Recurse -Filter *.java).Count
Get-ChildItem -Recurse -Filter *.java | Select-Object -First 60 FullName
@("pom.xml","build.gradle","build.gradle.kts") | Where-Object { Test-Path $_ }
(Get-ChildItem -Recurse -Filter *.java | Where-Object FullName -match "test").Count
Select-String -Path pom.xml,build.gradle -Pattern "sourceCompatibility|java.version|release" | Select-Object -First 10
```

**Detect Spring Boot** (any platform — look in pom.xml / build.gradle):
```bash
# Linux/macOS
grep -r "spring-boot\|spring-web\|spring-data" pom.xml build.gradle 2>/dev/null | head -10

# Windows cmd
findstr /s "spring-boot spring-web spring-data" pom.xml build.gradle

# Windows PowerShell
Select-String -Path pom.xml,build.gradle -Pattern "spring-boot|spring-web|spring-data" | Select-Object -First 10
```
If Spring Boot is detected, also run the Spring Boot & HTTP checks in §1.5 and §1.6.

### 1.2 — Static Analysis Setup

Install and run static analysis tools. 
**Quick checklist** (run whichever are available/applicable):
- [ ] Compile with `-Xlint:all` warnings enabled
- [ ] Run SpotBugs / FindBugs for bug patterns
- [ ] Run PMD for code style and complexity (including cyclomatic complexity report)
- [ ] Run Checkstyle for formatting violations
- [ ] Run SonarLint locally if available
- [ ] Grep manually for known anti-pattern signatures (see §1.3)
- [ ] Spring Boot checks (see §1.5) — if Spring Boot detected
- [ ] HTTP / REST checks (see §1.6) — if REST controllers detected

### 1.3 — Manual Anti-Pattern Grep

All commands below shown for Linux/macOS (`grep`) and Windows (`findstr`). PowerShell users can use `Select-String -Recurse -Path "*.java" -Pattern "<pattern>"`.

```bash
# ── Linux/macOS ──────────────────────────────────────────────────
# Null pointer risks
grep -rn "\.get(\|\.iterator()" --include="*.java" | grep -v "//\|test"

# Raw types (generics erasure)
grep -rn "List \|Map \|Set \|ArrayList\b\|HashMap\b" --include="*.java" | grep -v "<\|import\|//\|test"

# Mutable statics (global state)
grep -rn "static [^f].*=\s*new\|static [^f].*(List\|Map\|Set)" --include="*.java"

# Swallowed exceptions
grep -rn "catch.*{$\|catch.*{\s*}" --include="*.java" -A 1

# String concatenation in loops
grep -rn 'for\|while' --include="*.java" -A 5 | grep -B2 '".*+\|+.*"'

# System.out / printStackTrace in production
grep -rn "System\.out\.\|e\.printStackTrace()" --include="*.java" | grep -v "test\|Test"

# Magic numbers
grep -rn '[^a-zA-Z0-9_][2-9][0-9]\+[^0-9L]' --include="*.java" | grep -v "//\|test\|@\|import" | head -20

# Deprecated API usage
grep -rn "@Deprecated\|new Date(\|new Integer(\|new Boolean(" --include="*.java"

# Thread-safety red flags
grep -rn "static.*List\|static.*Map\|synchronized\b" --include="*.java" | head -20
```

```cmd
:: ── Windows cmd ─────────────────────────────────────────────────
findstr /s /n "\.get(" *.java | findstr /v "//"
findstr /s /n "List \|Map \|Set \|ArrayList\b\|HashMap\b" *.java | findstr /v "<\|import\|//"
findstr /s /n "static " *.java | findstr "= new\|List\|Map\|Set" | findstr /v "final"
findstr /s /n "catch" *.java
findstr /s /n "System\.out\.\|printStackTrace" *.java | findstr /v "test\|Test"
findstr /s /n "@Deprecated\|new Date(\|new Integer(\|new Boolean(" *.java
```

### 1.4 — Issue Classification

For each finding, classify it in a markdown table:

| ID | File | Line | Category | Severity | Description |
|----|------|------|----------|----------|-------------|
| S001 | Foo.java | 42 | Code Smell | Medium | Long method (>50 lines) |
| B001 | Bar.java | 17 | Bug | Critical | NPE risk: unchecked return |
| AP001 | Baz.java | 88 | Anti-Pattern | High | God class (>500 lines, >20 methods) |
| SB001 | OrderSvc.java | 12 | Spring Boot | High | Missing `@Transactional` on multi-step write |
| HTTP001 | UserCtrl.java | 55 | HTTP Design | High | POST used to fetch data (should be GET) |
| CC001 | PriceCalc.java | 30 | Complexity | High | Cyclomatic complexity 24 (threshold: 10) |
| OE001 | RepoFactory.java | 1 | Over-Engineering | Medium | Abstract factory for a single implementation |

**Severity levels**: Critical → High → Medium → Low  
**Categories**: Bug, Code Smell, Anti-Pattern, Security, Performance, Maintainability, Spring Boot, HTTP Design, Complexity, Over-Engineering


---

### 1.5 — Spring Boot Anti-Pattern Checks

Run these if Spring Boot is present. 
```bash
# ── Linux/macOS ──────────────────────────────────────────────────
# God @Service / @Component classes (>300 lines)
find . -name "*.java" -exec grep -l "@Service\|@Component\|@Repository" {} \; | \
  xargs wc -l | sort -rn | head -10

# Business logic leaking into @RestController
grep -rn "@RestController\|@Controller" --include="*.java" -l | \
  xargs grep -l "repository\.\|\.save(\|\.findBy\|if.*==\|for.*:"

# Missing @Transactional on methods that do multiple writes
grep -rn "\.save(\|\.delete(\|\.update(" --include="*.java" | grep -v "@Transactional\|test"

# @Autowired on fields (prefer constructor injection)
grep -rn "@Autowired" --include="*.java" | grep -v "//\|test\|constructor"

# Returning entity objects directly from controllers (exposes DB schema)
grep -rn "@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping" --include="*.java" -A 5 | \
  grep "return.*Entity\|return.*entity"

# @Transactional on private methods (has no effect with Spring proxy)
grep -rn "@Transactional" --include="*.java" -A 1 | grep "private "

# Hardcoded values that should be in application.properties
grep -rn '"http://\|"jdbc:\|"localhost\|"8080\|"secret\|"password' --include="*.java" | grep -v "test\|//"

# N+1 query risk: loops calling repository methods
grep -rn "for\|\.forEach" --include="*.java" -A 3 | grep "repository\.\|\.findBy\|\.getById"
```

```cmd
:: ── Windows cmd ─────────────────────────────────────────────────
findstr /s /n "@Service @Component @Repository" *.java
findstr /s /n "@Autowired" *.java | findstr /v "//"
findstr /s /n "@Transactional" *.java
findstr /s /n "http:// jdbc: localhost 8080 secret password" *.java | findstr /v "test //"
```

**Key Spring Boot smells to flag**:

| Smell | Signal | Severity |
|-------|--------|----------|
| God `@Service` | >300 lines, >15 methods | High |
| Fat controller | Business logic / repo calls inside `@RestController` | High |
| Missing `@Transactional` | Multiple `.save()` / `.delete()` calls without it | Critical |
| Field injection (`@Autowired` on field) | Not constructor injection | Medium |
| Entity returned from controller | Leaks DB schema, breaks API contract | High |
| `@Transactional` on `private` method | Spring proxy ignores it — silently broken | Critical |
| Hardcoded config | URLs, ports, secrets in source code | Critical |
| N+1 queries | `findById` inside a loop | High |
| Exception swallowed in `@ExceptionHandler` | Returns 200 with error body | High |
| No input validation | Missing `@Valid` / `@Validated` on controller params | High |

---

### 1.6 — HTTP / REST Design Checks

```bash
# ── Linux/macOS ──────────────────────────────────────────────────
# POST used for retrieval (should be GET)
grep -rn "@PostMapping" --include="*.java" -A 5 | grep -i "get\|fetch\|find\|search\|list\|query"

# GET used for mutations (should be POST/PUT/DELETE)
grep -rn "@GetMapping" --include="*.java" -A 5 | grep -i "create\|update\|delete\|save\|remove\|add"

# Wrong HTTP status codes (e.g. returning 200 for creation instead of 201)
grep -rn "ResponseEntity\.ok\|HttpStatus\.OK" --include="*.java" -B 3 | grep "create\|save\|post\|new"

# Returning null instead of 404
grep -rn "return null\|return ResponseEntity\.ok(null)" --include="*.java" | grep -v "test\|//"

# Overly generic endpoints (catch-all /api/process, /api/execute)
grep -rn "@RequestMapping\|@PostMapping\|@GetMapping" --include="*.java" | \
  grep -i "process\|execute\|handle\|do\|run\|action"

# Exposing stack traces / internal errors to HTTP response
grep -rn "e\.getMessage()\|e\.toString()" --include="*.java" | grep "ResponseEntity\|return\|body"

# Missing pagination on collection endpoints
grep -rn "@GetMapping" --include="*.java" -A 10 | grep "List<\|Collection<" | grep -v "Pageable\|Page<\|pageable"

# Verbs in resource URLs (REST anti-pattern)
grep -rn "@RequestMapping\|@GetMapping\|@PostMapping" --include="*.java" | \
  grep -i '"/.*get\|/.*create\|/.*delete\|/.*update\|/.*fetch\|/.*list'
```

```cmd
:: ── Windows cmd ─────────────────────────────────────────────────
findstr /s /n "@PostMapping" *.java
findstr /s /n "@GetMapping" *.java
findstr /s /n "return null" *.java | findstr /v "test //"
findstr /s /n "e.getMessage e.toString" *.java | findstr "ResponseEntity return body"
```

**Key HTTP / REST violations to flag**:

| Violation | Signal | Correct Approach | Severity |
|-----------|--------|-----------------|----------|
| POST as GET | `@PostMapping` for data retrieval | Use `@GetMapping` with query params | High |
| GET as mutation | `@GetMapping` creates/updates/deletes | Use `@PostMapping` / `@PutMapping` / `@DeleteMapping` | Critical |
| Wrong 200 for creation | `ResponseEntity.ok()` after create | Return `201 Created` with `Location` header | Medium |
| Null instead of 404 | `return null` or `return ok(null)` | `ResponseEntity.notFound().build()` | High |
| No 404 for missing resource | Returns empty 200 | Return `404 Not Found` | High |
| Verbs in URLs | `/api/getUsers`, `/api/createOrder` | `/api/users` (GET), `/api/orders` (POST) | Medium |
| Chatty API (N calls for one screen) | No composite endpoints | Introduce aggregate endpoint or GraphQL | Medium |
| Unbounded collection response | Returns `List<X>` with no paging | Use `Page<X>` + `Pageable` | High |
| Internal error exposure | Stack trace / message in response body | Generic error DTO + log internally | High |
| Missing input validation | No `@Valid` / `@NotNull` on body/params | Add Bean Validation annotations | High |

---

### 1.7 — Cyclomatic & Cognitive Complexity

**Cyclomatic complexity** = number of independent paths through a method = `1 + (if + for + while + case + catch + && + ||)`. Threshold: **≤ 10** per method.

**Cognitive complexity** (SonarQube metric) measures how hard the code is to understand. Threshold: **≤ 15** per method.

**Automated measurement (PMD)**:
```bash
# Linux/macOS — generate complexity report
mvn pmd:pmd
# Then open: target/site/pmd.html → filter by "CyclomaticComplexity"

# Or directly with PMD CLI (cross-platform with Java)
java -jar pmd-bin/lib/pmd-core-*.jar check \
  -d src/main/java \
  -R rulesets/java/design.xml \
  -f text 2>&1 | grep "Cyclomatic\|NPath\|Cognitive" | sort -t: -k3 -rn | head -20
```

**Manual grep to find high-complexity candidates** (methods with many branches):
```bash
# Linux/macOS — find methods with 5+ if/else/for/while/case/catch on nearby lines
grep -rn "if \|else \|for \|while \|case \|catch " --include="*.java" | \
  awk -F: '{print $1}' | sort | uniq -c | sort -rn | head -20

# Windows PowerShell
Get-ChildItem -Recurse -Filter *.java | ForEach-Object {
  $count = (Select-String -Path $_.FullName -Pattern "\bif\b|\bfor\b|\bwhile\b|\bcase\b|\bcatch\b").Count
  [PSCustomObject]@{File=$_.Name; Branches=$count}
} | Sort-Object Branches -Descending | Select-Object -First 20
```

**What to look for**:

| Complexity | Action |
|-----------|--------|
| CC ≤ 10 | OK |
| CC 11–15 | Smell — consider extracting methods |
| CC 16–20 | High — must refactor |
| CC > 20 | Critical — immediate refactor required |

**Common causes and fixes** :
- Deeply nested `if/else` → Extract Method + early return (guard clauses)
- Long `switch` / `if-else` chain → Strategy or polymorphism
- Boolean flag parameters driving branching → Split into two methods
- Complex loop bodies → Extract to named method

---

### 1.8 — Over-Engineering Detection

Over-engineering is as harmful as under-engineering — it creates accidental complexity, slows onboarding, and makes simple changes hard.

```bash
# Linux/macOS — find suspiciously abstract structures
# Interfaces with a single implementation
grep -rln "interface " --include="*.java" src/main/java | while read iface; do
  name=$(grep "interface " "$iface" | head -1 | awk '{print $2}')
  count=$(grep -rl "implements $name\b" --include="*.java" src/main/java | wc -l)
  [ "$count" -eq 1 ] && echo "Single-impl interface: $name in $iface"
done

# Abstract classes with one subclass
grep -rln "abstract class" --include="*.java" | head -20

# Factory for a single product type
grep -rn "Factory\|Builder\b" --include="*.java" -l | xargs grep -l "return new " | head -10

# Deeply nested packages (>4 levels = often over-structured)
find . -name "*.java" | awk -F/ '{print NF}' | sort -rn | head -5

# Excessive use of design pattern names (may signal pattern abuse)
grep -rn "Factory\|Builder\|Facade\|Proxy\|Decorator\|Visitor\|Observer\|Strategy\|Command\|Mediator" \
  --include="*.java" -l | wc -l
```

```cmd
:: Windows cmd
findstr /s /n "interface " *.java
findstr /s /n "abstract class" *.java
findstr /s /n "Factory Builder" *.java
```

**Over-engineering red flags**:

| Pattern | Red Flag Signal | When It's Actually OK |
|---------|----------------|----------------------|
| Interface + single impl | `IOrderService` → `OrderServiceImpl` | Multiple implementations expected, or external API |
| Abstract class + 1 subclass | `AbstractProcessor` → `ConcreteProcessor` | Template Method genuinely needed |
| Factory for one product | `OrderFactory.create()` always returns `Order` | Multiple products vary at runtime |
| Generic base class for 1 use | `BaseEntityProcessor<T>` used once | Reused across 3+ types |
| Layers of delegation | A → B → C → D all doing nothing | Separation of concerns is real |
| DTOs wrapping DTOs | `ResponseDto<DataDto<ItemDto>>` | API versioning or external contract |
| Event bus for local calls | Event fired and consumed in same service | True decoupling across bounded contexts |
| Micro-classes | `EmailAddress` with 1 field, no validation | Value object with real invariants |
| Config objects for 1 value | `DatabaseConfig { String url; }` | Multiple related values, externalized config |

---

## Phase 2: Test Capture (TDD Foundation)

**Goal**: Lock in existing behavior with tests so refactoring can't silently break anything.

> This is the most important phase. Do not skip or rush it.

### 2.1 — Build & Baseline

```bash
# Linux/macOS — Maven
./mvnw clean test 2>&1 | tail -20

# Linux/macOS — Gradle
./gradlew clean test 2>&1 | tail -20
```
```cmd
:: Windows cmd — Maven
mvnw.cmd clean test 2>&1 | more

:: Windows cmd — Gradle
gradlew.bat clean test 2>&1 | more
```
```powershell
# Windows PowerShell
.\mvnw clean test | Select-Object -Last 20
.\gradlew clean test | Select-Object -Last 20
```

Record the **baseline**: test count, pass count, build time. You'll compare at end of Phase 4.

### 2.2 — Characterization Tests

For every class/method that will be refactored, write **characterization tests** — tests that document what the code *currently does*, even if it's wrong. These tests act as a behavioral safety net.

```java
/**
 * Characterization test for LegacyOrderProcessor.
 * Documents EXISTING behavior before refactor — do NOT change these assertions
 * unless the behavior is explicitly being fixed as part of the refactor.
 */
class LegacyOrderProcessorCharacterizationTest {

    @Test
    void processOrder_withNullCustomer_returnsMinusOne() {
        // This documents a known bug: should throw, but currently returns -1
        // DO NOT FIX until Phase 3 with explicit sign-off
        LegacyOrderProcessor sut = new LegacyOrderProcessor();
        int result = sut.processOrder(null, List.of(new Item("SKU1", 10.0)));
        assertEquals(-1, result);
    }

    @Test
    void calculateDiscount_over100Items_appliesLegacyTieredRate() {
        LegacyOrderProcessor sut = new LegacyOrderProcessor();
        double discount = sut.calculateDiscount(150);
        // Documents the magic number behavior: 150 items → 7.5% discount
        assertEquals(0.075, discount, 0.001);
    }
}
```

**Rules for characterization tests**:
- One test per observable behavior (not per line of code)
- Name tests with `_currentBehavior_` or use comments marking them as characterization
- Include edge cases: null inputs, empty collections, boundary values, exception paths
- Cover every public method of every class being refactored

### 2.3 — Unit Tests for Bug Fixes

For each **Bug** found in Phase 1, write a **failing test** that demonstrates the bug *before* fixing it:

```java
@Test
@DisplayName("BUG-B001: processOrder should throw IllegalArgumentException for null customer, not return -1")
void processOrder_withNullCustomer_shouldThrowIllegalArgumentException() {
    LegacyOrderProcessor sut = new LegacyOrderProcessor();
    // This test FAILS before the fix — that's expected and correct
    assertThrows(IllegalArgumentException.class,
        () -> sut.processOrder(null, List.of(new Item("SKU1", 10.0))));
}
```

Label these clearly. They will be **red** at commit time and turn **green** after the Phase 3 fix.

### 2.4 — Coverage Gate

```bash
# Linux/macOS — Maven + JaCoCo
./mvnw jacoco:report
# Report at: target/site/jacoco/index.html

# Linux/macOS — Gradle
./gradlew jacocoTestReport
# Report at: build/reports/jacoco/test/html/index.html
```
```cmd
:: Windows cmd
mvnw.cmd jacoco:report
start target\site\jacoco\index.html

gradlew.bat jacocoTestReport
start build\reports\jacoco\test\html\index.html
```

**Do not proceed to Phase 3 unless**:
- [ ] All characterization tests pass
- [ ] All bug-fix tests are written (failing is OK for now)
- [ ] Line coverage ≥ 80% on refactor targets (or documented why not achievable)
- [ ] Spring Boot: `@RestController`, `@Service`, `@Repository` layers each have at least smoke tests
- [ ] HTTP: at least one `MockMvc` / `WebTestClient` test per controller endpoint being refactored


---

## Phase 3: Refactor

**Goal**: Fix every issue from Phase 1, one at a time, keeping the test suite green after each change.

### 3.1 — Refactor Order (follow this sequence)

Refactor in this order to minimize cascading breakage:

1. **Extract constants** — Replace magic numbers/strings with named constants. Lowest risk.
2. **Fix critical bugs** — Address Severity: Critical items. Turn red bug-fix tests green.
3. **Fix HTTP design violations** — Correct HTTP methods, status codes, missing pagination. Low structural risk.
4. **Fix Spring Boot anti-patterns** — Add missing `@Transactional`, fix field injection, split fat services.
5. **Reduce cyclomatic complexity** — Extract methods, introduce guard clauses, replace switch with Strategy.
6. **Extract methods** — Break long methods into smaller, named methods.
7. **Rename** — Rename to reveal intent. Use IDE refactor → rename for safety.
8. **Extract classes** — Split God classes, separate concerns. Apply SRP.
9. **Remove over-engineering** — Delete unused abstractions, single-impl interfaces, empty delegators.
10. **Replace anti-patterns** — Raw types → generics, Date → LocalDate, StringBuffer → StringBuilder.
11. **Fix exception handling** — Never swallow; add proper logging or rethrowing.
12. **Performance fixes** — StringBuilder in loops, fix N+1, cache expensive calls.
13. **Apply design patterns** — Only where they genuinely simplify.

### 3.2 — Per-Refactor Checklist

For every single change:

```
[ ] Change is the smallest possible unit (one method, one class, one pattern)
[ ] Run: mvn test (or ./gradlew test)
[ ] All characterization tests still GREEN
[ ] Bug-fix tests: green if the bug was just fixed, otherwise still red (expected)
[ ] Commit with message format: "refactor(S001): extract calculateDiscount to DiscountCalculator"
```

**Stop immediately if any characterization test turns red**. Undo the last change, understand why it broke, then either fix the refactor or update the characterization test (with a comment explaining the intentional behavior change).

### 3.3 — Common Refactors with Examples

- God Class → SRP decomposition
- Long Method → Extract Method
- Feature Envy → Move Method
- Primitive Obsession → Value Objects
- Switch Statements → Strategy/Polymorphism
- Data Clumps → Parameter Objects
- Shotgun Surgery → Cohesion fixes
- Raw types → Generics
- Checked/Unchecked exception design
- Reducing cyclomatic complexity (guard clauses, early return, decomposition)
- Spring Boot fat controller → thin controller + service layer
- Field injection → constructor injection
- HTTP method corrections and status code fixes
- Removing over-engineered abstractions safely

### 3.4 — Dependency & Architecture Fixes

If the analysis revealed architectural problems (circular dependencies, tight coupling):

```bash
# Linux/macOS
./mvnw dependency:tree
jdeps --multi-release 11 -recursive target/classes/
```
```cmd
:: Windows cmd
mvnw.cmd dependency:tree
jdeps --multi-release 11 -recursive target\classes\
```

Apply dependency inversion: depend on interfaces, not implementations. Introduce constructor injection if not present.

---

## Phase 4: Verify

**Goal**: Confirm nothing broke, coverage held, and all issues from Phase 1 are resolved.

### 4.1 — Full Test Suite

```bash
# Linux/macOS
./mvnw clean verify
./gradlew clean check
```
```cmd
:: Windows cmd
mvnw.cmd clean verify
gradlew.bat clean check
```

**Success criteria**:
- [ ] All characterization tests GREEN
- [ ] All bug-fix tests GREEN (were red before Phase 3 fixes, now green)
- [ ] No new test failures introduced
- [ ] Test count ≥ starting count (you added tests, not removed them)
- [ ] Spring Boot: `MockMvc` / `WebTestClient` tests confirm correct HTTP verbs and status codes
- [ ] Complexity: PMD report shows no method with CC > 10

### 4.2 — Coverage Report

```bash
# Linux/macOS
./mvnw jacoco:report
open target/site/jacoco/index.html
```
```cmd
:: Windows cmd
mvnw.cmd jacoco:report
start target\site\jacoco\index.html
```

- [ ] Line coverage ≥ 80% on all refactored classes
- [ ] Branch coverage ≥ 70% on classes with complex logic
- [ ] No regression from pre-refactor baseline

### 4.3 — Issue Audit

Go back to the Phase 1 issue table. For each item:

| ID | Status | Notes |
|----|--------|-------|
| S001 | ✅ Fixed | Extracted to DiscountCalculator, test added |
| B001 | ✅ Fixed | Now throws IllegalArgumentException, bug test green |
| AP001 | ✅ Fixed | Split into OrderProcessor + CustomerValidator + ItemValidator |

### 4.4 — Static Analysis Recheck

Re-run the same tools from Phase 1. New issue count should be 0 for all Critical/High items, and lower overall for Medium/Low.

### 4.5 — Final Deliverables

Produce a refactor summary report containing:

1. **Before/After metrics**: Line count, method count, cyclomatic complexity, test count, coverage %
2. **Issues resolved**: Count by category and severity
3. **Issues deferred**: Any items intentionally left for a follow-up (with rationale)
4. **Test suite summary**: New tests added, characterization tests, bug-fix tests
5. **Architecture changes**: Any structural changes made (new classes, interfaces, packages)
6. **Known remaining risks**: Anything that couldn't be safely refactored without deeper changes

---

## Quick Reference: Issue Severity Decision Tree

```
Is it a NullPointerException / ClassCastException risk in production?      → Critical
Does it cause silent data corruption or incorrect calculations?             → Critical
Is it a security vulnerability (SQL injection, path traversal, etc.)?      → Critical
Is @Transactional missing on a multi-step write operation?                  → Critical
Does a GET endpoint mutate state?                                           → Critical
Is @Transactional on a private method (silently ignored by Spring proxy)?  → Critical
Does it make code untestable or block future changes?                       → High
Is cyclomatic complexity > 20?                                              → High
Does an HTTP endpoint return wrong status codes (e.g. 200 for errors)?     → High
Is a collection endpoint unbounded (no pagination)?                         → High
Is a Spring @Service / @Component > 300 lines with mixed concerns?         → High
Is there an interface with a single implementation and no extension plan?   → Medium (over-engineering)
Is cyclomatic complexity 11–20?                                             → Medium
Is there a readability/maintainability problem with no correctness risk?    → Medium
Is it a style preference or minor naming issue?                             → Low
```

---

---

## Appendices

All reference material is included below. No external files required.

---

## Appendix A — Code Smell Catalog

A comprehensive reference for identifying code smells, anti-patterns, and bug patterns in legacy Java codebases.

---

## Table of Contents

1. [Method-Level Smells](#1-method-level-smells)
2. [Class-Level Smells](#2-class-level-smells)
3. [Architecture-Level Smells](#3-architecture-level-smells)
4. [Critical Bug Patterns](#4-critical-bug-patterns)
5. [Performance Anti-Patterns](#5-performance-anti-patterns)
6. [Security Anti-Patterns](#6-security-anti-patterns)
7. [Concurrency Anti-Patterns](#7-concurrency-anti-patterns)
8. [Modern Java Migration Issues](#8-modern-java-migration-issues)

---

## 1. Method-Level Smells

### Long Method
- **Detection**: Methods > 20-30 lines (strict), or > 10 lines with multiple responsibilities
- **Grep**: `awk '/\{/{depth++} /\}/{depth--} depth>4{print FILENAME ":" NR}' *.java`
- **Fix**: Extract Method — identify logical sections and name them
- **Severity**: Medium

### Long Parameter List
- **Detection**: > 3-4 parameters in a method signature
- **Grep**: `grep -n "([^)]*,[^)]*,[^)]*,[^)]*)" --include="*.java" -r`
- **Fix**: Introduce Parameter Object or Builder pattern
- **Severity**: Medium

### Flag Arguments (Boolean Parameters)
- **Detection**: `boolean` parameters in public methods
- **Grep**: `grep -rn "boolean [a-z]" --include="*.java" | grep "def\|void\|int\|String"`
- **Fix**: Split into two separate, clearly named methods
- **Severity**: Medium
- **Example (bad)**: `void render(boolean isSummary)`
- **Example (good)**: `void renderSummary()` and `void renderDetail()`

### Divergent Change
- **Detection**: One class changes for multiple unrelated reasons
- **Symptom**: Commits touching same file for different features repeatedly
- **Fix**: Split class along lines of change
- **Severity**: High

### Shotgun Surgery
- **Detection**: One logical change requires edits to many classes
- **Symptom**: Feature changes touch 5+ files every time
- **Fix**: Move related behavior together; improve cohesion
- **Severity**: High

### Dead Code
- **Grep**: 
  ```bash
  # Unused private methods
  grep -rn "private " --include="*.java" | grep -v "//\|test" | \
    awk -F'(' '{print $1}' | awk '{print $NF}' > /tmp/private_methods.txt
  
  # Commented-out blocks
  grep -rn "//.*[;{}]" --include="*.java" | head -30
  ```
- **Fix**: Delete. Source control is your undo button.
- **Severity**: Low-Medium

### Speculative Generality
- **Detection**: Abstract classes with one implementation, unused parameters, unnecessary delegation
- **Fix**: YAGNI — remove until needed
- **Severity**: Low

---

## 2. Class-Level Smells

### God Class (Blob)
- **Detection**: > 300-500 lines, > 20 methods, touches many unrelated data types
- **Grep**: 
  ```bash
  # Count lines per file
  find . -name "*.java" -exec wc -l {} + | sort -rn | head -20
  
  # Count methods per file
  grep -c "public\|private\|protected" $(find . -name "*.java") | sort -rn | head -20
  ```
- **Fix**: Extract Class, identify SRP responsibilities
- **Severity**: High

### Data Class
- **Detection**: Class with only fields, getters, setters — no behavior
- **Symptom**: All logic lives in classes that use it (Feature Envy)
- **Fix**: Move behavior into the data class; add validation in setters
- **Severity**: Medium

### Refused Bequest
- **Detection**: Subclass inherits methods it doesn't need or overrides them to throw exceptions
- **Example**: `throw new UnsupportedOperationException()` in an override
- **Fix**: Replace inheritance with delegation/composition
- **Severity**: High

### Parallel Inheritance Hierarchies
- **Detection**: Every time you add a subclass of X, you must also add a subclass of Y
- **Fix**: Merge hierarchies or use Strategy
- **Severity**: High

### Primitive Obsession
- **Detection**: Using int/String/double to represent domain concepts (Money, PhoneNumber, Email, ZIP)
- **Grep**: `grep -rn "String.*email\|String.*phone\|double.*price\|int.*id" --include="*.java"`
- **Fix**: Extract Value Objects with validation and behavior
- **Severity**: Medium

### Data Clumps
- **Detection**: Same 3+ variables appear together repeatedly in parameter lists or fields
- **Fix**: Extract Parameter Object or domain class
- **Severity**: Medium

---

## 3. Architecture-Level Smells

### Feature Envy
- **Detection**: A method uses more data from another class than from its own
- **Symptom**: `other.getX()`, `other.getY()`, `other.getZ()` dominating a method body
- **Fix**: Move Method to the class being envied
- **Severity**: Medium-High

### Inappropriate Intimacy
- **Detection**: Two classes access each other's private fields/internals excessively
- **Fix**: Move behavior, introduce interface, use accessor methods
- **Severity**: High

### Message Chains
- **Detection**: `a.getB().getC().getD().doSomething()` — Law of Demeter violations
- **Grep**: `grep -rn "\.\w\+().\w\+().\w\+()" --include="*.java"` 
- **Fix**: Hide delegation; add a method on the intermediate object
- **Severity**: Medium

### Middle Man
- **Detection**: Class that does nothing but delegate to another class
- **Fix**: Remove delegation and call the actual object directly
- **Severity**: Low-Medium

### Circular Dependencies
- **Detection**: Package A imports Package B which imports Package A
- **Tool**: `jdeps`, `mvn dependency:analyze`
- **Fix**: Extract shared interface, apply Dependency Inversion
- **Severity**: High

---

## 4. Critical Bug Patterns

### Null Pointer Dereference Risk
- **Pattern**: Using return values without null checks; `.get()` on Optional without `isPresent()`
- **Grep**:
  ```bash
  grep -rn "\.get()" --include="*.java" | grep -v "Optional\|Map\|//\|test" | head -20
  grep -rn "return.*\.get\b\|= .*\.get(" --include="*.java" | head -20
  ```
- **Fix**: Use `Optional`, add null checks, use `Objects.requireNonNull` for preconditions
- **Severity**: Critical

### equals() / hashCode() Contract Violation
- **Pattern**: Overriding `equals()` without overriding `hashCode()` (or vice versa)
- **Grep**: 
  ```bash
  # Classes that override equals but not hashCode
  grep -rln "equals(Object" --include="*.java" | while read f; do
    grep -L "hashCode()" "$f" && echo "$f: has equals but no hashCode"
  done
  ```
- **Fix**: Always override both, or neither. Use IDE generation or Lombok `@EqualsAndHashCode`
- **Severity**: Critical (causes silent bugs in HashMaps/Sets)

### String Comparison with ==
- **Pattern**: `if (str == "literal")` or `if (str1 == str2)`
- **Grep**: `grep -rn '== "' --include="*.java" | grep -v "//\|@"`
- **Fix**: Use `.equals()` or `Objects.equals()` for null-safety
- **Severity**: Critical

### Integer Overflow
- **Pattern**: `int` used for calculations that could exceed 2,147,483,647
- **Common cases**: milliseconds, byte counts, large multiplications
- **Fix**: Use `long`, `BigInteger`, or `Math.multiplyExact()`
- **Severity**: Critical

### Resource Leak
- **Pattern**: Streams, connections, readers not closed in `finally` or try-with-resources
- **Grep**: 
  ```bash
  grep -rn "new FileInputStream\|new Connection\|new BufferedReader\|getConnection()" \
    --include="*.java" | grep -v "try\|//" | head -20
  ```
- **Fix**: Use try-with-resources (`try (InputStream is = ...)`)
- **Severity**: Critical

### Swallowed Exceptions
- **Pattern**: Empty catch block, or catch block that only calls `e.printStackTrace()`
- **Grep**:
  ```bash
  grep -rn "catch" --include="*.java" -A 2 | grep -B1 "^--$\|} *$\|printStackTrace"
  ```
- **Fix**: Log properly, rethrow as RuntimeException, or handle explicitly
- **Severity**: Critical

### Mutable Static Fields (Non-Thread-Safe Global State)
- **Pattern**: `static List<X> cache = new ArrayList<>()` modified by multiple threads
- **Grep**: `grep -rn "static [^f].*new\b" --include="*.java" | grep -v "//\|test"`
- **Fix**: Use `Collections.synchronizedList()`, `ConcurrentHashMap`, or eliminate global state
- **Severity**: Critical (in multi-threaded environments)

### Float/Double for Money
- **Pattern**: `double price`, `float total` for financial calculations
- **Grep**: `grep -rn "double.*price\|double.*amount\|float.*cost\|float.*total" --include="*.java"`
- **Fix**: Use `BigDecimal` for all financial calculations
- **Severity**: Critical

### Integer Division Truncation
- **Pattern**: `int result = 7 / 2;` → `3` not `3.5`
- **Fix**: Cast to `double` before division: `(double) a / b`
- **Severity**: High

---

## 5. Performance Anti-Patterns

### String Concatenation in Loops
- **Pattern**: `String s = ""; for (...) { s = s + item; }`
- **Grep**: `grep -rn '".*+\|+.*"' --include="*.java" -B5 | grep "for\|while" | head -20`
- **Fix**: Use `StringBuilder` with `.append()`
- **Severity**: High (O(n²) vs O(n))

### Repeated Map Lookup
- **Pattern**: `if (map.containsKey(k)) { map.get(k).doSomething(); }`
- **Fix**: `map.getOrDefault(k, default)` or `map.computeIfAbsent(k, k -> new ArrayList<>())`
- **Severity**: Medium

### Collection Copying Unnecessarily
- **Pattern**: `new ArrayList<>(existingList)` inside a loop just to iterate
- **Fix**: Use views, iterators, or stream operations
- **Severity**: Medium

### N+1 Query Pattern
- **Pattern**: Fetching a list, then querying for each element inside a loop
- **Fix**: Use JOIN queries, batch fetching, or `IN` clauses
- **Severity**: High (in DB-heavy apps)

### Boxing/Unboxing in Tight Loops
- **Pattern**: `Integer` instead of `int` in hot paths; `List<Integer>` instead of `int[]`
- **Fix**: Use primitives in performance-sensitive code
- **Severity**: Medium

---

## 6. Security Anti-Patterns

### SQL Injection
- **Pattern**: String concatenation to build SQL queries
- **Grep**: `grep -rn "\"SELECT\|\"INSERT\|\"UPDATE\|\"DELETE" --include="*.java" | grep "+"`
- **Fix**: Use `PreparedStatement` with parameterized queries, or JPA/Hibernate
- **Severity**: Critical

### Path Traversal
- **Pattern**: `new File(userInput)` or `Paths.get(userInput)` without validation
- **Grep**: `grep -rn "new File(\|Paths.get(" --include="*.java" | grep -v "//\|test" | head -10`
- **Fix**: Validate and sanitize file paths; use allow-lists
- **Severity**: Critical

### Hardcoded Credentials
- **Grep**: 
  ```bash
  grep -rni "password\s*=\s*\"[^\"]\|secret\s*=\s*\"[^\"\|apikey\s*=\s*\"[^\"" \
    --include="*.java" | grep -v "//\|test\|placeholder\|example"
  ```
- **Fix**: Move to environment variables or secrets management
- **Severity**: Critical

### Insecure Random
- **Pattern**: `java.util.Random` used for security tokens, session IDs, or passwords
- **Grep**: `grep -rn "new Random(\|Math.random()" --include="*.java"`
- **Fix**: Use `java.security.SecureRandom`
- **Severity**: Critical

### Verbose Error Messages to Users
- **Pattern**: Stack traces or internal details sent to UI/API response
- **Fix**: Log internally; return generic error codes/messages to clients
- **Severity**: High

---

## 7. Concurrency Anti-Patterns

### Double-Checked Locking (without volatile)
- **Pattern**: Singleton with double-check but field not declared `volatile`
- **Grep**: `grep -rn "if.*null.*synchronized\|synchronized.*if.*null" --include="*.java"`
- **Fix**: Add `volatile`, or use enum singleton, or `Holder` pattern
- **Severity**: Critical

### Calling Overridable Methods in Constructor
- **Pattern**: Calling `this.someMethod()` in a constructor where `someMethod` is not `final`
- **Fix**: Use factory methods instead of constructors for complex initialization
- **Severity**: High

### Lock on Wrong Object
- **Pattern**: `synchronized (new Object())` — locking on a local/new object, providing no mutual exclusion
- **Grep**: `grep -rn "synchronized.*new " --include="*.java"`
- **Fix**: Lock on shared field; use `ReentrantLock`
- **Severity**: Critical

### Shared Mutable State Without Synchronization
- **Pattern**: `ArrayList` or `HashMap` modified from multiple threads without locking
- **Fix**: `ConcurrentHashMap`, `CopyOnWriteArrayList`, or explicit synchronization
- **Severity**: Critical

---

## 8. Modern Java Migration Issues

### Legacy Date/Time API
- **Pattern**: `java.util.Date`, `java.util.Calendar`, `SimpleDateFormat`
- **Grep**: `grep -rn "import java.util.Date\|new Date(\|Calendar.getInstance\|SimpleDateFormat" --include="*.java"`
- **Fix**: Replace with `java.time.*`: `LocalDate`, `LocalDateTime`, `ZonedDateTime`, `DateTimeFormatter`
- **Severity**: Medium-High

### Raw Types
- **Pattern**: `List list`, `Map map`, `Set set` without generic parameters
- **Grep**: `grep -rn "\bList \b\|\bMap \b\|\bSet \b\|\bIterator \b" --include="*.java" | grep -v "//\|import\|<"`
- **Fix**: Add proper generic type parameters
- **Severity**: Medium

### Pre-Java-8 Loops Replaceable with Streams
- **Pattern**: Loops that filter, map, or reduce collections
- **Fix**: Use `stream().filter().map().collect()` where it improves clarity (not always — don't force it)
- **Severity**: Low (style)

### Pre-Java-7 Exception Handling
- **Pattern**: Multiple catch blocks for the same handling logic
- **Fix**: Use multi-catch: `catch (IOException | SQLException e)`
- **Severity**: Low

### Manual Null Checks Replaceable with Optional
- **Pattern**: `if (x != null) { ... }` chains that could use `Optional`
- **Fix**: Return `Optional<T>` from methods that may return null, use `.map().orElse()`
- **Note**: Don't use `Optional` as method parameter — only as return type
- **Severity**: Low-Medium

### String.format vs Text Blocks (Java 13+)
- **Pattern**: Multi-line string building with `\n` and `String.format()`
- **Fix**: Use text blocks `""" ... """`
- **Severity**: Low


---

## Appendix B — Static Analysis Tools (Cross-Platform Setup)

Setup, commands, and output interpretation for static analysis tools used in Phase 1.

> **Platform note**: Maven (`mvn`) and Gradle (`./gradlew`) commands work on all platforms via their wrappers.
> On **Windows cmd** use `mvnw.cmd` / `gradlew.bat`; on **PowerShell** use `.\mvnw` / `.\gradlew`.
> To open HTML reports: Linux/macOS use `open <path>`; Windows cmd use `start <path>`; PowerShell use `Invoke-Item <path>`.

---

## Table of Contents

1. [Compiler Warnings](#1-compiler-warnings-free-always-run-first)
2. [SpotBugs](#2-spotbugs)
3. [PMD](#3-pmd)
4. [Checkstyle](#4-checkstyle)
5. [JDepend / jdeps](#5-jdepend--jdeps)
6. [JaCoCo (Coverage)](#6-jacoco-coverage)
7. [SonarQube / SonarLint](#7-sonarqube--sonarlint)
8. [Interpreting Combined Results](#8-interpreting-combined-results)

---

## 1. Compiler Warnings (Free, Always Run First)

```bash
# Maven — enable all lint warnings
mvn clean compile -Dmaven.compiler.compilerArgument="-Xlint:all" 2>&1 | grep "warning:"

# Gradle — add to build.gradle
# tasks.withType(JavaCompile) {
#     options.compilerArgs << "-Xlint:all"
# }
./gradlew clean compileJava 2>&1 | grep "warning:"
```

**Key warning categories to prioritize**:
| Warning | Meaning | Severity |
|---------|---------|----------|
| `unchecked` | Raw generic type used | High |
| `deprecation` | Deprecated API used | Medium |
| `fallthrough` | switch case falls through without break | High |
| `divzero` | Possible division by zero | Critical |
| `null` | Potential null dereference | Critical |
| `cast` | Unsafe cast | High |
| `serial` | Serializable class missing serialVersionUID | Low |

---

## 2. SpotBugs

SpotBugs (successor to FindBugs) finds real bugs — null dereferences, infinite loops, incorrect use of APIs.

### Maven Setup

```xml
<!-- pom.xml -->
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.2.0</version>
    <configuration>
        <effort>Max</effort>
        <threshold>Low</threshold>
        <xmlOutput>true</xmlOutput>
    </configuration>
</plugin>
```

```bash
mvn spotbugs:check 2>&1
mvn spotbugs:gui  # Opens visual report
```

### Gradle Setup

```groovy
// build.gradle
plugins {
    id 'com.github.spotbugs' version '6.0.9'
}

spotbugs {
    effort = 'max'
    reportLevel = 'low'
}
```

```bash
./gradlew spotbugsMain
# Report at: build/reports/spotbugs/main.html
```

### Critical SpotBugs Bug Codes to Prioritize

| Code | Description | Severity |
|------|-------------|----------|
| `NP_NULL_ON_SOME_PATH` | Possible null pointer dereference | Critical |
| `NP_ALWAYS_NULL` | Value is always null | Critical |
| `EC_UNRELATED_TYPES` | equals() on unrelated types always false | Critical |
| `HE_EQUALS_NO_HASHCODE` | equals() without hashCode() | Critical |
| `OS_OPEN_STREAM` | Stream/connection never closed | Critical |
| `DM_DEFAULT_ENCODING` | Uses default charset (platform-dependent) | High |
| `ST_WRITE_TO_STATIC_FROM_INSTANCE_METHOD` | Static field written from instance method | High |
| `DC_DOUBLECHECK` | Double-checked locking without volatile | Critical |
| `IS2_INCONSISTENT_SYNC` | Inconsistent synchronization | Critical |
| `SQL_NONCONSTANT_STRING_PASSED_TO_EXECUTE` | SQL injection risk | Critical |

---

## 3. PMD

PMD finds code style issues, overly complex code, dead code, and common anti-patterns.

### Maven Setup

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-pmd-plugin</artifactId>
    <version>3.21.2</version>
    <configuration>
        <rulesets>
            <ruleset>/rulesets/java/maven-pmd-plugin-default.xml</ruleset>
        </rulesets>
        <failOnViolation>false</failOnViolation>
        <printFailingErrors>true</printFailingErrors>
    </configuration>
</plugin>
```

```bash
mvn pmd:check 2>&1 | grep "PMD Failure"
mvn pmd:pmd  # Generates HTML report at target/site/pmd.html
```

### Gradle Setup

```groovy
plugins {
    id 'pmd'
}

pmd {
    toolVersion = '7.0.0'
    ruleSets = ["category/java/bestpractices.xml", "category/java/errorprone.xml",
                "category/java/design.xml", "category/java/performance.xml"]
}
```

### Key PMD Rules to Watch

| Rule | Issue | Action |
|------|-------|--------|
| `CyclomaticComplexity > 10` | Method too complex | Extract Method |
| `GodClass` | Class does too much | Extract Class |
| `TooManyMethods > 20` | Class too large | Extract Class |
| `AvoidCatchingGenericException` | Catches Exception/Throwable | Be specific |
| `EmptyCatchBlock` | Swallowed exception | Handle or rethrow |
| `UseStringBufferForStringAppends` | String concat in loop | Use StringBuilder |
| `UseCollectionIsEmpty` | `size() == 0` instead of `isEmpty()` | Cosmetic but fix |
| `UnusedImports` | Dead import | Clean up |
| `UnusedFormalParameter` | Unused method param | Remove or `_` |
| `AvoidDuplicateLiterals` | Same string repeated 4+ times | Extract constant |

---

## 4. Checkstyle

Checkstyle enforces coding standards and naming conventions.

### Maven Setup

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.1</version>
    <configuration>
        <configLocation>google_checks.xml</configLocation>
        <failsOnError>false</failsOnError>
        <consoleOutput>true</consoleOutput>
    </configuration>
</plugin>
```

```bash
mvn checkstyle:check 2>&1 | grep "WARN\|ERROR" | head -40
```

**Note**: For legacy codebases, use Checkstyle as signal, not a hard gate. Focus on:
- Missing `@Override` annotations
- Missing Javadoc on public APIs
- Inconsistent naming conventions that suggest confusion

---

## 5. JDepend / jdeps

For analyzing package and module dependencies — essential for finding circular dependencies.

### jdeps (Built into JDK 8+)

```bash
# After compiling: check dependencies for your main packages
jdeps -verbose:class target/classes/ 2>&1 | head -60

# Check for internal API usage
jdeps --jdk-internals target/classes/ 2>&1

# For multi-release JARs
jdeps --multi-release 11 target/myapp.jar

# Generate dot graph for visualization
jdeps --dot-output /tmp/deps target/classes/
dot -Tsvg /tmp/deps/summary.dot -o /tmp/deps.svg
```

### Maven Dependency Plugin

```bash
# Show all transitive dependencies
mvn dependency:tree

# Find unused declared dependencies and used undeclared dependencies
mvn dependency:analyze 2>&1 | grep "WARNING"
```

---

## 6. JaCoCo (Coverage)

### Maven Setup

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```bash
mvn clean test jacoco:report
open target/site/jacoco/index.html
```

### Coverage Targets for Refactor Work

| Coverage Type | Target | Notes |
|--------------|--------|-------|
| Line Coverage | ≥ 80% | Per refactored class |
| Branch Coverage | ≥ 70% | For classes with conditional logic |
| Method Coverage | ≥ 90% | All public methods should have at least one test |

### Reading the JaCoCo Report

- **Green**: Fully covered lines
- **Yellow**: Partially covered branches (one path taken, not the other)
- **Red**: Uncovered — priority for new characterization tests

---

## 7. SonarQube / SonarLint

### SonarLint (IDE Plugin — Recommended for Dev)

Install SonarLint in IntelliJ IDEA or Eclipse. It gives real-time feedback as you refactor.

Key rules to pay attention to:
- `java:S1172` — Unused method parameters
- `java:S1135` — TODO comments (track these)
- `java:S2629` — Inefficient String formatting in logger calls
- `java:S3776` — Cognitive Complexity too high
- `java:S1448` — Too many methods in a class

### SonarQube (CI Integration)

```bash
# If SonarQube server is available
mvn sonar:sonar \
  -Dsonar.projectKey=my-project \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=mytoken
```

---

## 8. Interpreting Combined Results

When multiple tools report overlapping issues, use this priority matrix:

| Source | Trust Level | Use For |
|--------|-------------|---------|
| SpotBugs | Very High — finds real runtime bugs | Bug catalog |
| Compiler warnings | High — definitely wrong | Bug catalog |
| PMD complexity | High | Refactor prioritization |
| JaCoCo uncovered | High | Test gap identification |
| PMD style | Medium | Backlog |
| Checkstyle | Lower | Style guide enforcement only |

### Combining into Your Issue Table

```
Severity: Critical  → Any SpotBugs bug-class finding, compiler null/cast warning
Severity: High      → PMD complexity > 15, missing hashCode, resource leaks
Severity: Medium    → PMD 10-15 complexity, raw types, deprecated API
Severity: Low       → Checkstyle violations, naming, unused imports
```

**Rule**: If SpotBugs and PMD agree something is wrong, it definitely is. Fix it in Phase 3.


---

## Appendix C — Spring Boot Anti-Patterns & HTTP/REST Design

Detailed detection, explanation, and fix examples for Spring Boot and REST API issues.

---

## Table of Contents

1. [Spring Boot Anti-Patterns](#1-spring-boot-anti-patterns)
2. [HTTP / REST Design Violations](#2-http--rest-design-violations)
3. [Spring Boot Testing Patterns](#3-spring-boot-testing-patterns)
4. [Complexity & Over-Engineering in Spring Boot Context](#4-complexity--over-engineering-in-spring-boot-context)

---

## 1. Spring Boot Anti-Patterns

### 1.1 — Fat Controller (Business Logic in @RestController)

**Problem**: Controllers contain `if`/`for`, direct repository calls, and business rules. Controllers should only translate HTTP ↔ domain — nothing more.

```java
// BAD — controller does everything
@RestController
@RequestMapping("/orders")
public class OrderController {
    @Autowired
    private OrderRepository orderRepo;
    @Autowired
    private InventoryRepository inventoryRepo;

    @PostMapping
    public ResponseEntity<Order> placeOrder(@RequestBody OrderRequest req) {
        // Business logic in controller — WRONG
        if (req.getItems().isEmpty()) return ResponseEntity.badRequest().build();
        double total = 0;
        for (var item : req.getItems()) {
            Inventory inv = inventoryRepo.findBySku(item.getSku());
            if (inv.getStock() < item.getQty()) throw new RuntimeException("Out of stock");
            total += inv.getPrice() * item.getQty();
        }
        Order order = new Order(req.getCustomerId(), total);
        orderRepo.save(order);
        emailService.send(req.getEmail(), order);
        return ResponseEntity.ok(order);
    }
}

// GOOD — controller delegates to service
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> placeOrder(@Valid @RequestBody OrderRequest req) {
        OrderResponse response = orderService.placeOrder(req);
        return ResponseEntity.status(HttpStatus.CREATED)
            .header(HttpHeaders.LOCATION, "/orders/" + response.getOrderId())
            .body(response);
    }
}
```

### 1.2 — Missing @Transactional on Multi-Step Writes

**Problem**: Multiple DB writes without a transaction — if step 2 fails, step 1 is committed. Data corruption.

```java
// BAD — no transaction, partial writes possible
@Service
public class TransferService {
    public void transfer(String fromId, String toId, BigDecimal amount) {
        Account from = accountRepo.findById(fromId).orElseThrow();
        Account to = accountRepo.findById(toId).orElseThrow();
        from.debit(amount);
        accountRepo.save(from);    // ← committed here
        to.credit(amount);         // ← if this throws, money is gone
        accountRepo.save(to);
    }
}

// GOOD — atomic transaction
@Service
public class TransferService {
    @Transactional  // All-or-nothing
    public void transfer(String fromId, String toId, BigDecimal amount) {
        Account from = accountRepo.findById(fromId).orElseThrow();
        Account to = accountRepo.findById(toId).orElseThrow();
        from.debit(amount);
        to.credit(amount);
        accountRepo.save(from);
        accountRepo.save(to);
    }
}
```

**@Transactional on private method — silently ignored**:
```java
// BAD — Spring proxy can't intercept private methods; @Transactional has no effect
@Service
public class OrderService {
    @Transactional  // ← IGNORED. No transaction will be created.
    private void saveOrderAndItems(Order order, List<Item> items) {
        orderRepo.save(order);
        itemRepo.saveAll(items);
    }
}

// FIX — make the method package-private or public, or restructure
@Service
public class OrderService {
    @Transactional
    public void saveOrderAndItems(Order order, List<Item> items) { // public or package-private
        orderRepo.save(order);
        itemRepo.saveAll(items);
    }
}
```

### 1.3 — Field Injection (@Autowired on Fields)

**Problem**: Makes classes untestable without Spring context; hides dependencies; can lead to NPEs.

```java
// BAD — field injection
@Service
public class ReportService {
    @Autowired
    private CustomerRepository customerRepo;  // Hidden dependency
    @Autowired
    private EmailSender emailSender;
}

// GOOD — constructor injection (testable, immutable, explicit)
@Service
public class ReportService {
    private final CustomerRepository customerRepo;
    private final EmailSender emailSender;

    public ReportService(CustomerRepository customerRepo, EmailSender emailSender) {
        this.customerRepo = customerRepo;
        this.emailSender = emailSender;
    }
}

// In tests — no Spring context needed
class ReportServiceTest {
    @Test
    void generateReport_sendsEmail() {
        var sut = new ReportService(mockCustomerRepo, mockEmailSender);
        // ...
    }
}
```

### 1.4 — Returning Entity Objects from Controllers

**Problem**: Exposes DB schema, breaks API contract on every schema change, can cause `LazyInitializationException`, leaks sensitive fields.

```java
// BAD — entity returned directly
@GetMapping("/{id}")
public ResponseEntity<UserEntity> getUser(@PathVariable String id) {
    return ResponseEntity.ok(userRepository.findById(id).orElseThrow());
    // ↑ Exposes: passwordHash, internalIds, DB timestamps, ALL fields
}

// GOOD — map to response DTO
@GetMapping("/{id}")
public ResponseEntity<UserResponse> getUser(@PathVariable String id) {
    UserEntity user = userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));
    return ResponseEntity.ok(UserResponse.from(user));
}

// DTO — only what the API contract needs
public record UserResponse(String id, String name, String email) {
    public static UserResponse from(UserEntity e) {
        return new UserResponse(e.getId(), e.getFullName(), e.getEmail());
    }
}
```

### 1.5 — Hardcoded Configuration Values

```java
// BAD
@Service
public class PaymentService {
    private final String apiUrl = "https://payments.example.com/api/v2";
    private final String apiKey = "sk_live_abc123secret";  // ← CRITICAL security issue
    private final int timeoutMs = 5000;
}

// GOOD — externalized to application.properties / environment
@Service
public class PaymentService {
    @Value("${payment.api.url}")
    private String apiUrl;

    @Value("${payment.api.key}")
    private String apiKey;

    @Value("${payment.timeout.ms:5000}")  // 5000 default
    private int timeoutMs;
}

// Even better — typed configuration properties
@ConfigurationProperties(prefix = "payment")
@Validated
public record PaymentProperties(
    @NotBlank String apiUrl,
    @NotBlank String apiKey,
    @Positive int timeoutMs
) {}
```

### 1.6 — N+1 Query Pattern

```java
// BAD — 1 query for orders + N queries for each customer
@GetMapping("/orders")
public List<OrderSummary> getAllOrders() {
    List<Order> orders = orderRepo.findAll();  // 1 query
    return orders.stream()
        .map(o -> {
            Customer c = customerRepo.findById(o.getCustomerId()).orElseThrow(); // N queries!
            return new OrderSummary(o, c.getName());
        })
        .toList();
}

// GOOD — join in the query
@GetMapping("/orders")
public List<OrderSummary> getAllOrders() {
    return orderRepo.findAllWithCustomer();  // 1 JOIN query
}

// Repository
public interface OrderRepository extends JpaRepository<Order, String> {
    @Query("SELECT new com.example.dto.OrderSummary(o.id, o.total, c.name) " +
           "FROM Order o JOIN o.customer c")
    List<OrderSummary> findAllWithCustomer();
}
```

### 1.7 — Missing Input Validation

```java
// BAD — no validation, garbage in
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@RequestBody UserRequest req) {
    // req.email could be null, req.age could be -999
    return ResponseEntity.ok(userService.create(req));
}

// GOOD — Bean Validation annotations + @Valid
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@Valid @RequestBody UserRequest req) {
    return ResponseEntity.status(HttpStatus.CREATED).body(userService.create(req));
}

public record UserRequest(
    @NotBlank(message = "Name is required") String name,
    @Email(message = "Invalid email format") @NotBlank String email,
    @Min(value = 18, message = "Must be 18 or older") @Max(150) int age
) {}

// Global exception handler for validation errors
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return ResponseEntity.badRequest().body(new ErrorResponse("Validation failed", errors));
    }
}
```

---

## 2. HTTP / REST Design Violations

### 2.1 — POST Used for Retrieval

**Rule**: GET for retrieval (safe, idempotent, cacheable). POST for creation or operations with side effects.

```java
// BAD — POST to search/fetch data
@PostMapping("/users/search")
public List<UserResponse> searchUsers(@RequestBody SearchRequest req) { ... }

// GOOD — GET with query parameters
@GetMapping("/users")
public Page<UserResponse> searchUsers(
        @RequestParam(required = false) String name,
        @RequestParam(required = false) String email,
        Pageable pageable) { ... }

// Exception: when search criteria is complex and has sensitive data (e.g., PII)
// that shouldn't be in URL/logs — POST is acceptable. Document why.
```

### 2.2 — GET Used for Mutations

**Rule**: GET must be safe (no side effects) and idempotent.

```java
// BAD — GET mutates state
@GetMapping("/orders/{id}/cancel")
public ResponseEntity<Void> cancelOrder(@PathVariable String id) {
    orderService.cancel(id);  // ← side effect on GET!
    return ResponseEntity.ok().build();
}

// GOOD — POST or DELETE for mutations
@DeleteMapping("/orders/{id}")
public ResponseEntity<Void> cancelOrder(@PathVariable String id) {
    orderService.cancel(id);
    return ResponseEntity.noContent().build();  // 204
}
// Or if it's a state transition (not a delete): POST to a sub-resource
@PostMapping("/orders/{id}/cancellation")
public ResponseEntity<OrderResponse> cancelOrder(@PathVariable String id) {
    OrderResponse updated = orderService.cancel(id);
    return ResponseEntity.ok(updated);
}
```

### 2.3 — Wrong HTTP Status Codes

```java
// BAD — wrong status codes throughout
@PostMapping("/orders")
public ResponseEntity<Order> create(@RequestBody OrderRequest req) {
    Order order = orderService.create(req);
    return ResponseEntity.ok(order);           // ← 200 for creation, should be 201
}

@GetMapping("/orders/{id}")
public ResponseEntity<Order> get(@PathVariable String id) {
    Order order = orderService.find(id);
    return ResponseEntity.ok(order);           // ← 200 even if null (NPE risk or wrong contract)
}

// GOOD — semantically correct status codes
@PostMapping("/orders")
public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderRequest req) {
    OrderResponse order = orderService.create(req);
    URI location = URI.create("/orders/" + order.getId());
    return ResponseEntity.created(location).body(order);  // 201 Created + Location header
}

@GetMapping("/orders/{id}")
public ResponseEntity<OrderResponse> get(@PathVariable String id) {
    return orderService.find(id)
        .map(ResponseEntity::ok)                           // 200 if found
        .orElse(ResponseEntity.notFound().build());        // 404 if not found
}

@DeleteMapping("/orders/{id}")
public ResponseEntity<Void> delete(@PathVariable String id) {
    orderService.delete(id);
    return ResponseEntity.noContent().build();             // 204 No Content
}
```

**Status code cheatsheet**:
| Code | Use for |
|------|---------|
| 200 OK | Successful GET, PUT, PATCH |
| 201 Created | Successful POST that created a resource |
| 204 No Content | Successful DELETE or action with no response body |
| 400 Bad Request | Invalid input (validation failure) |
| 401 Unauthorized | Not authenticated |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | State conflict (duplicate, optimistic lock) |
| 422 Unprocessable Entity | Semantically invalid (valid JSON but business rule violation) |
| 500 Internal Server Error | Unexpected server error (never expose details to client) |

### 2.4 — Verbs in Resource URLs

```java
// BAD — verbs in URLs (RPC-style, not REST)
@GetMapping("/getUsers")
@PostMapping("/createOrder")
@GetMapping("/fetchProductById/{id}")
@PostMapping("/deleteUser/{id}")

// GOOD — nouns only, HTTP verb carries the action
@GetMapping("/users")             // GET    /users        → list
@GetMapping("/users/{id}")        // GET    /users/123    → single
@PostMapping("/orders")           // POST   /orders       → create
@GetMapping("/products/{id}")     // GET    /products/123 → fetch
@DeleteMapping("/users/{id}")     // DELETE /users/123    → delete
@PutMapping("/users/{id}")        // PUT    /users/123    → full update
@PatchMapping("/users/{id}")      // PATCH  /users/123    → partial update
```

### 2.5 — Unbounded Collection Responses

```java
// BAD — returns all records, no limit
@GetMapping("/orders")
public List<Order> getAllOrders() {
    return orderRepo.findAll();  // Could be millions of rows
}

// GOOD — paginated response
@GetMapping("/orders")
public Page<OrderResponse> getOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt,desc") String sort) {
    Pageable pageable = PageRequest.of(page, size, Sort.by(sort.split(",")));
    return orderRepo.findAll(pageable).map(OrderResponse::from);
}
// Response includes: content[], page, size, totalElements, totalPages, last
```

### 2.6 — Exposing Internal Error Details

```java
// BAD — leaks stack traces, DB details, internal structure to API consumers
@ExceptionHandler(Exception.class)
public ResponseEntity<String> handleAll(Exception e) {
    return ResponseEntity.internalServerError()
        .body(e.getMessage());  // "Column 'user_id' cannot be null" — exposes DB schema!
}

// GOOD — generic error to client, full details to logs
@RestControllerAdvice
public class GlobalExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex) {
        return ResponseEntity.status(422).body(new ErrorResponse("BUSINESS_ERROR", ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception ex, HttpServletRequest req) {
        String correlationId = UUID.randomUUID().toString();
        log.error("Unhandled exception [correlationId={}] on {} {}: ",
            correlationId, req.getMethod(), req.getRequestURI(), ex);
        return ResponseEntity.status(500)
            .body(new ErrorResponse("INTERNAL_ERROR",
                "An unexpected error occurred. Reference: " + correlationId));
        // Client gets a safe message + correlation ID to report; no internals leaked
    }
}
```

---

## 3. Spring Boot Testing Patterns

### Controller (MockMvc)

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @MockBean OrderService orderService;

    @Test
    void placeOrder_withValidRequest_returns201WithLocation() throws Exception {
        given(orderService.placeOrder(any())).willReturn(new OrderResponse("ORD-1", new BigDecimal("99.99")));

        mockMvc.perform(post("/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(new OrderRequest("CUST-1", List.of()))))
            .andExpect(status().isCreated())                          // 201
            .andExpect(header().string("Location", "/orders/ORD-1")) // Location header
            .andExpect(jsonPath("$.orderId").value("ORD-1"))
            .andExpect(jsonPath("$.total").value(99.99));
    }

    @Test
    void placeOrder_whenOrderNotFound_returns404() throws Exception {
        given(orderService.find("MISSING")).willThrow(new ResourceNotFoundException("Order not found"));

        mockMvc.perform(get("/orders/MISSING"))
            .andExpect(status().isNotFound());                        // 404
    }

    @Test
    void placeOrder_withInvalidBody_returns400() throws Exception {
        mockMvc.perform(post("/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))  // Missing required fields
            .andExpect(status().isBadRequest())                       // 400
            .andExpect(jsonPath("$.errors").isArray());
    }
}
```

### Service with @Transactional

```java
@SpringBootTest
@Transactional  // Rolls back after each test
class TransferServiceIntegrationTest {

    @Autowired TransferService transferService;
    @Autowired AccountRepository accountRepo;

    @Test
    void transfer_whenSourceHasFunds_movesMoneyAtomically() {
        Account from = accountRepo.save(new Account("A1", new BigDecimal("100.00")));
        Account to = accountRepo.save(new Account("A2", BigDecimal.ZERO));

        transferService.transfer("A1", "A2", new BigDecimal("50.00"));

        assertThat(accountRepo.findById("A1").get().getBalance()).isEqualByComparingTo("50.00");
        assertThat(accountRepo.findById("A2").get().getBalance()).isEqualByComparingTo("50.00");
    }

    @Test
    void transfer_whenSourceInsufficientFunds_rollsBackBothAccounts() {
        Account from = accountRepo.save(new Account("A1", new BigDecimal("10.00")));
        Account to = accountRepo.save(new Account("A2", BigDecimal.ZERO));

        assertThrows(InsufficientFundsException.class,
            () -> transferService.transfer("A1", "A2", new BigDecimal("50.00")));

        // Both balances unchanged
        assertThat(accountRepo.findById("A1").get().getBalance()).isEqualByComparingTo("10.00");
        assertThat(accountRepo.findById("A2").get().getBalance()).isEqualByComparingTo("0.00");
    }
}
```

---

## 4. Complexity & Over-Engineering in Spring Boot Context

### Reducing Cyclomatic Complexity — Guard Clauses

```java
// BAD — deeply nested, CC = 9
public OrderResult processOrder(Order order, Customer customer, Inventory inventory) {
    if (order != null) {
        if (customer != null) {
            if (customer.isActive()) {
                if (inventory.hasStock(order.getSku(), order.getQty())) {
                    if (order.getTotal().compareTo(customer.getCreditLimit()) <= 0) {
                        // actual work buried 5 levels deep
                        return doProcess(order, customer);
                    } else {
                        return OrderResult.creditExceeded();
                    }
                } else {
                    return OrderResult.outOfStock();
                }
            } else {
                return OrderResult.inactiveCustomer();
            }
        } else {
            throw new IllegalArgumentException("Customer required");
        }
    } else {
        throw new IllegalArgumentException("Order required");
    }
}

// GOOD — guard clauses, CC = 5
public OrderResult processOrder(Order order, Customer customer, Inventory inventory) {
    Objects.requireNonNull(order, "Order required");
    Objects.requireNonNull(customer, "Customer required");

    if (!customer.isActive())                                       return OrderResult.inactiveCustomer();
    if (!inventory.hasStock(order.getSku(), order.getQty()))        return OrderResult.outOfStock();
    if (order.getTotal().compareTo(customer.getCreditLimit()) > 0)  return OrderResult.creditExceeded();

    return doProcess(order, customer);  // Happy path at the bottom, no nesting
}
```

### Removing Over-Engineered Spring Abstractions

```java
// BAD — over-engineered: interface + impl for a single Spring service
public interface IUserService { User findById(String id); void save(User u); }
public class UserServiceImpl implements IUserService { ... }  // Only ever one implementation

// FIX — just a @Service class (add interface when second impl is actually needed)
@Service
public class UserService { ... }

// BAD — generic repository factory for one type
@Component
public class RepositoryFactory {
    public <T> JpaRepository<T, String> getRepository(Class<T> type) { ... }
}
// Usage: repositoryFactory.getRepository(User.class).findAll()

// FIX — just inject the repository directly
@Repository
public interface UserRepository extends JpaRepository<User, String> {}
// Usage: userRepository.findAll()

// BAD — event-driven for in-process calls with no async or decoupling benefit
@Service
public class OrderService {
    public void createOrder(Order order) {
        eventBus.publish(new OrderCreatedEvent(order)); // Why? It's just calling OrderEmailService
    }
}
@EventListener
class OrderEmailService {
    public void on(OrderCreatedEvent event) { emailSender.send(...); }
}

// FIX — direct call (use events only for real cross-module or async decoupling)
@Service
public class OrderService {
    private final OrderEmailService emailService;
    public void createOrder(Order order) {
        orderRepo.save(order);
        emailService.sendConfirmation(order);
    }
}
```


---

## Appendix D — Testing Patterns for Legacy Code

Advanced patterns for writing characterization and unit tests on hard-to-test legacy codebases.

---

## Table of Contents

1. [Test Structure & Naming Conventions](#1-test-structure--naming-conventions)
2. [Characterization Tests](#2-characterization-tests)
3. [Breaking Dependencies for Testability](#3-breaking-dependencies-for-testability)
4. [Mocking Strategies](#4-mocking-strategies)
5. [Testing Static Methods & Singletons](#5-testing-static-methods--singletons)
6. [Testing Time-Dependent Code](#6-testing-time-dependent-code)
7. [Testing Exception Paths](#7-testing-exception-paths)
8. [Testing Multithreaded Code](#8-testing-multithreaded-code)
9. [Parameterized Tests for Legacy Logic](#9-parameterized-tests-for-legacy-logic)
10. [Integration Test Patterns](#10-integration-test-patterns)

---

## 1. Test Structure & Naming Conventions

### Base Test Class Setup (JUnit 5)

```java
// pom.xml — ensure JUnit 5 + Mockito
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.7.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.1</version>
    <scope>test</scope>
</dependency>
```

### Naming Convention

Use: `methodName_stateUnderTest_expectedBehavior()`

```java
class OrderProcessorTest {
    // Good names — tell a story
    void processOrder_withValidOrderAndCustomer_returnsOrderId()
    void processOrder_withNullCustomer_throwsIllegalArgumentException()
    void processOrder_withEmptyItemList_throwsIllegalArgumentException()
    void calculateDiscount_withMoreThan100Items_appliesVolumeDiscount()
    void calculateDiscount_withExactly100Items_appliesBoundaryRate()
    void calculateDiscount_withZeroItems_returnsZeroDiscount()
}
```

### AAA Pattern (Arrange-Act-Assert)

```java
@Test
void calculateDiscount_withMoreThan100Items_appliesVolumeDiscount() {
    // Arrange
    OrderProcessor processor = new OrderProcessor(mockInventory, mockLogger);
    int itemCount = 150;

    // Act
    double discount = processor.calculateDiscount(itemCount);

    // Assert
    assertThat(discount).isEqualByComparingTo(0.075);
}
```

---

## 2. Characterization Tests

Characterization tests capture existing behavior, bugs and all. They prevent silent regressions.

### Template

```java
/**
 * CHARACTERIZATION TESTS for LegacyPaymentService
 *
 * These tests document CURRENT behavior of LegacyPaymentService as of [date].
 * They were written BEFORE refactoring began to create a safety net.
 *
 * IMPORTANT:
 * - Do NOT change these assertions without explicit sign-off that behavior is intentionally changing.
 * - If a test fails after a refactor, STOP. Investigate before proceeding.
 * - Some assertions may capture BUGS — see inline comments.
 */
@Tag("characterization")
class LegacyPaymentServiceCharacterizationTest {

    private LegacyPaymentService sut; // System Under Test

    @BeforeEach
    void setUp() {
        sut = new LegacyPaymentService(/* real or minimal dependencies */);
    }

    @Test
    void charge_withZeroAmount_returnsSuccessInsteadOfThrowingBUG() {
        // NOTE: This is a BUG. Charging zero should throw or be a no-op.
        // Current behavior returns SUCCESS. Bug ID: B-042.
        // The bug-fix test is in LegacyPaymentServiceBugFixTest.
        PaymentResult result = sut.charge(customerId, BigDecimal.ZERO);
        assertEquals(PaymentResult.SUCCESS, result);
    }

    @Test
    void charge_withNegativeAmount_returnsSuccessInsteadOfThrowingBUG() {
        // BUG: Negative amounts should be rejected. Bug ID: B-043.
        PaymentResult result = sut.charge(customerId, new BigDecimal("-10.00"));
        assertEquals(PaymentResult.SUCCESS, result);
    }

    @Test
    void getTransactionHistory_forNewCustomer_returnsNullNotEmptyListBUG() {
        // BUG: Should return empty list, not null. Bug ID: B-044.
        List<Transaction> history = sut.getTransactionHistory("new-customer-id");
        assertNull(history); // Documents the null return — callers must null-check
    }
}
```

### Discovering Behavior to Characterize

When you can't read the code easily, write golden-master tests:

```java
@Test
void goldenMaster_processAllSampleOrders_matchesRecordedOutput() throws Exception {
    // Load sample inputs
    List<Order> sampleOrders = loadTestFixture("src/test/resources/sample-orders.json");
    
    // Record current output (run once, then commit the expected file)
    List<String> actualOutputs = sampleOrders.stream()
        .map(o -> sut.process(o).toString())
        .collect(toList());

    // Compare against recorded golden output
    List<String> expectedOutputs = Files.readAllLines(
        Paths.get("src/test/resources/golden-master-output.txt"));

    assertThat(actualOutputs).isEqualTo(expectedOutputs);
}
```

**Workflow**: Run once → save output → commit as golden master → refactor → run again to verify.

---

## 3. Breaking Dependencies for Testability

### Problem: Constructor Creates Dependencies (Hard to Test)

```java
// LEGACY — hard to test, creates real DB connection
public class OrderService {
    private final DatabaseRepository repo = new DatabaseRepository("jdbc:mysql://...");
    private final EmailSender email = new EmailSender("smtp.company.com");
}
```

### Solution: Constructor Injection (for new tests without modifying production code yet)

```java
// STEP 1: Add overloaded constructor (don't change existing one — characterization tests use it)
public class OrderService {
    private final DatabaseRepository repo;
    private final EmailSender email;

    // Existing constructor — keep for backward compatibility
    public OrderService() {
        this.repo = new DatabaseRepository("jdbc:mysql://...");
        this.email = new EmailSender("smtp.company.com");
    }

    // New constructor for testing — package-private is fine
    OrderService(DatabaseRepository repo, EmailSender email) {
        this.repo = repo;
        this.email = email;
    }
}

// TEST
class OrderServiceTest {
    @Mock DatabaseRepository mockRepo;
    @Mock EmailSender mockEmail;

    @Test
    void placeOrder_callsRepoAndSendsEmail() {
        // Uses the new injectable constructor
        OrderService sut = new OrderService(mockRepo, mockEmail);
        // ...
    }
}
```

### Extract and Override (for testing private behavior)

```java
// Legacy class with untestable private method
public class ReportGenerator {
    public String generate(ReportConfig config) {
        String data = fetchFromLegacySystem(); // Can't test — calls external system
        return format(data, config);
    }

    private String fetchFromLegacySystem() {
        // ... legacy system call
    }
}

// Testable subclass — override the problematic method
class TestableReportGenerator extends ReportGenerator {
    private final String stubbedData;

    TestableReportGenerator(String stubbedData) {
        this.stubbedData = stubbedData;
    }

    @Override
    protected String fetchFromLegacySystem() { // Make protected in production first
        return stubbedData;
    }
}

// Now you can test format() via generate() without the external call
@Test
void generate_withValidData_formatsCorrectly() {
    var sut = new TestableReportGenerator("{\"items\":[]}");
    String result = sut.generate(new ReportConfig("CSV"));
    assertThat(result).startsWith("items,");
}
```

---

## 4. Mocking Strategies

### Basic Mockito Setup (JUnit 5)

```java
@ExtendWith(MockitoExtension.class)
class OrderProcessorTest {

    @Mock
    private InventoryService mockInventory;

    @Mock
    private NotificationService mockNotification;

    @InjectMocks
    private OrderProcessor sut; // Mockito injects mocks automatically

    @Test
    void processOrder_whenItemInStock_reservesItem() {
        // Arrange
        given(mockInventory.isAvailable("SKU-123", 2)).willReturn(true);

        // Act
        sut.processOrder(new Order("SKU-123", 2));

        // Assert behavior (verify interaction)
        then(mockInventory).should().reserve("SKU-123", 2);
        then(mockNotification).should().sendConfirmation(any());
    }

    @Test
    void processOrder_whenItemOutOfStock_throwsOutOfStockException() {
        given(mockInventory.isAvailable("SKU-123", 2)).willReturn(false);

        assertThrows(OutOfStockException.class,
            () -> sut.processOrder(new Order("SKU-123", 2)));

        then(mockInventory).should(never()).reserve(any(), anyInt());
    }
}
```

### Capturing Arguments

```java
@Test
void sendReport_passesCorrectEmailAddress() {
    ArgumentCaptor<EmailMessage> emailCaptor = ArgumentCaptor.forClass(EmailMessage.class);

    sut.sendDailyReport(customer);

    verify(mockEmailService).send(emailCaptor.capture());
    EmailMessage sentEmail = emailCaptor.getValue();
    assertThat(sentEmail.getTo()).isEqualTo("customer@example.com");
    assertThat(sentEmail.getSubject()).contains("Daily Report");
}
```

---

## 5. Testing Static Methods & Singletons

### Option A: Mockito-inline (JUnit 5, Mockito 4+)

```java
// Mock static method
try (MockedStatic<LegacyUtil> mockUtil = mockStatic(LegacyUtil.class)) {
    mockUtil.when(() -> LegacyUtil.getCurrentTimestamp()).thenReturn(12345L);
    
    String result = sut.processWithTimestamp("data");
    
    assertThat(result).contains("12345");
}
```

### Option B: Wrapper Interface (Permanent, Better Design)

```java
// 1. Extract interface
public interface TimeProvider {
    long currentTimeMillis();
}

// 2. Production implementation
public class SystemTimeProvider implements TimeProvider {
    public long currentTimeMillis() { return System.currentTimeMillis(); }
}

// 3. Use in production class (inject it)
public class OrderService {
    private final TimeProvider clock;
    // ...
}

// 4. Test with lambda
@Test
void createOrder_timestampsWithCurrentTime() {
    TimeProvider fixedClock = () -> 1_700_000_000_000L;
    var sut = new OrderService(fixedClock, mockRepo);
    Order order = sut.createOrder(items);
    assertThat(order.getCreatedAt()).isEqualTo(1_700_000_000_000L);
}
```

### Testing Singletons

```java
// Reset singleton state between tests (use reflection if no reset method)
@BeforeEach
void resetSingleton() throws Exception {
    Field instance = LegacyRegistry.class.getDeclaredField("instance");
    instance.setAccessible(true);
    instance.set(null, null);
}
```

---

## 6. Testing Time-Dependent Code

### Java 8+ `Clock` Injection (Best Pattern)

```java
// Production class
public class SubscriptionService {
    private final Clock clock;
    
    public SubscriptionService(Clock clock) { this.clock = clock; }
    public SubscriptionService() { this(Clock.systemUTC()); }  // default for prod

    public boolean isExpired(Subscription sub) {
        return sub.getExpiryDate().isBefore(LocalDate.now(clock));
    }
}

// Test with fixed clock
@Test
void isExpired_whenExpiryIsYesterday_returnsTrue() {
    Clock fixedClock = Clock.fixed(
        Instant.parse("2024-03-15T10:00:00Z"), ZoneOffset.UTC);
    var sut = new SubscriptionService(fixedClock);
    
    Subscription sub = new Subscription(LocalDate.of(2024, 3, 14)); // yesterday
    assertThat(sut.isExpired(sub)).isTrue();
}
```

---

## 7. Testing Exception Paths

```java
// JUnit 5 assertThrows — preferred
@Test
void processOrder_withNullCustomer_throwsIllegalArgumentException() {
    IllegalArgumentException ex = assertThrows(
        IllegalArgumentException.class,
        () -> sut.processOrder(null, validItems)
    );
    assertThat(ex.getMessage()).contains("Customer must not be null");
}

// Testing that NO exception is thrown
@Test
void processOrder_withValidInput_doesNotThrow() {
    assertDoesNotThrow(() -> sut.processOrder(validCustomer, validItems));
}

// Testing chained / wrapped exceptions
@Test
void processOrder_whenDbFails_throwsServiceExceptionWrappingOriginal() {
    given(mockRepo.save(any())).willThrow(new SQLException("Connection refused"));

    ServiceException ex = assertThrows(ServiceException.class,
        () -> sut.processOrder(validCustomer, validItems));

    assertThat(ex.getCause()).isInstanceOf(SQLException.class);
    assertThat(ex.getCause().getMessage()).contains("Connection refused");
}
```

---

## 8. Testing Multithreaded Code

```java
@Test
void counter_concurrentIncrements_reachesExpectedTotal() throws InterruptedException {
    int threadCount = 100;
    int incrementsPerThread = 1000;
    ThreadSafeCounter counter = new ThreadSafeCounter();
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(threadCount);

    for (int i = 0; i < threadCount; i++) {
        Thread t = new Thread(() -> {
            try {
                startLatch.await(); // All threads start simultaneously
                for (int j = 0; j < incrementsPerThread; j++) {
                    counter.increment();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                doneLatch.countDown();
            }
        });
        t.start();
    }

    startLatch.countDown(); // Fire!
    assertThat(doneLatch.await(10, TimeUnit.SECONDS)).isTrue();
    assertThat(counter.get()).isEqualTo(threadCount * incrementsPerThread);
}
```

---

## 9. Parameterized Tests for Legacy Logic

Perfect for testing legacy calculation methods with many input combinations:

```java
@ParameterizedTest(name = "{0} items → {1}% discount")
@CsvSource({
    "0,   0.000",
    "1,   0.000",
    "50,  0.000",
    "99,  0.000",
    "100, 0.050",
    "101, 0.050",
    "150, 0.075",
    "200, 0.100",
    "999, 0.100",
})
void calculateDiscount_variousQuantities_returnsCorrectRate(int quantity, double expectedRate) {
    assertThat(sut.calculateDiscount(quantity)).isCloseTo(expectedRate, within(0.001));
}

// For complex objects — use @MethodSource
@ParameterizedTest
@MethodSource("invalidOrderCases")
void processOrder_withInvalidInput_throwsException(Order invalidOrder, Class<? extends Exception> expected) {
    assertThrows(expected, () -> sut.processOrder(invalidOrder));
}

static Stream<Arguments> invalidOrderCases() {
    return Stream.of(
        Arguments.of(null, IllegalArgumentException.class),
        Arguments.of(new Order(null, List.of()), IllegalArgumentException.class),
        Arguments.of(new Order("customer", List.of()), EmptyOrderException.class)
    );
}
```

---

## 10. Integration Test Patterns

### In-Memory DB for Repository Tests (H2)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@ActiveProfiles("test")
// application-test.properties: spring.datasource.url=jdbc:h2:mem:testdb
class OrderRepositoryIntegrationTest {

    @Autowired
    private OrderRepository repo;

    @Test
    @Transactional
    void save_andRetrieve_roundTripsCorrectly() {
        Order saved = repo.save(new Order("CUST-1", List.of(new Item("SKU-1", 10.0))));
        Order loaded = repo.findById(saved.getId()).orElseThrow();
        assertThat(loaded.getCustomerId()).isEqualTo("CUST-1");
        assertThat(loaded.getItems()).hasSize(1);
    }
}
```

### WireMock for External HTTP Calls

```java
@WireMockTest(httpPort = 8089)
class PaymentGatewayClientTest {

    @Test
    void charge_whenGatewayReturnsSuccess_returnsApproved(WireMockRuntimeInfo wmRuntimeInfo) {
        stubFor(post(urlEqualTo("/v1/charge"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"status\":\"approved\",\"transactionId\":\"TX-999\"}")));

        var client = new PaymentGatewayClient("http://localhost:8089");
        PaymentResponse result = client.charge("CARD-123", new BigDecimal("99.99"));

        assertThat(result.getStatus()).isEqualTo("approved");
        assertThat(result.getTransactionId()).isEqualTo("TX-999");
    }
}
```


---

## Appendix E — Refactor Patterns (Before/After Examples)

Detailed before/after examples for every common Java refactoring. Use during Phase 3.

---

## Table of Contents

1. [Extract Method](#1-extract-method)
2. [Extract Class / God Class Decomposition](#2-extract-class--god-class-decomposition)
3. [Replace Magic Numbers with Constants](#3-replace-magic-numbers-with-constants)
4. [Replace Primitive Obsession with Value Objects](#4-replace-primitive-obsession-with-value-objects)
5. [Replace Raw Types with Generics](#5-replace-raw-types-with-generics)
6. [Replace Legacy Date/Time](#6-replace-legacy-datetime)
7. [Fix Exception Handling](#7-fix-exception-handling)
8. [Replace String Concatenation in Loops](#8-replace-string-concatenation-in-loops)
9. [Replace Conditional with Polymorphism / Strategy](#9-replace-conditional-with-polymorphism--strategy)
10. [Introduce Dependency Injection](#10-introduce-dependency-injection)
11. [Fix Null Handling with Optional](#11-fix-null-handling-with-optional)
12. [Fix equals() and hashCode()](#12-fix-equals-and-hashcode)
13. [Replace Data Class with Behavior-Rich Object](#13-replace-data-class-with-behavior-rich-object)
14. [Fix Resource Leaks](#14-fix-resource-leaks)
15. [Fix Financial Calculations](#15-fix-financial-calculations)

---

## 1. Extract Method

**When**: Method is > 20 lines, or has a comment explaining what a block does (the comment = name of the new method).

```java
// BEFORE — one giant method
public OrderConfirmation processOrder(Customer customer, List<Item> items) {
    // Validate customer
    if (customer == null) throw new IllegalArgumentException("Customer null");
    if (customer.getEmail() == null || customer.getEmail().isEmpty())
        throw new IllegalArgumentException("Email required");
    if (!customer.getEmail().contains("@"))
        throw new IllegalArgumentException("Invalid email");

    // Calculate totals
    double subtotal = 0;
    for (Item item : items) {
        subtotal += item.getPrice() * item.getQuantity();
    }
    double tax = subtotal * 0.08;
    double total = subtotal + tax;

    // Apply discount
    double discount = 0;
    if (items.size() > 100) discount = total * 0.05;
    else if (items.size() > 50) discount = total * 0.025;
    total -= discount;

    // Save to database
    Order order = new Order(customer.getId(), total);
    orderRepo.save(order);

    // Send email
    emailService.sendConfirmation(customer.getEmail(), order.getId(), total);

    return new OrderConfirmation(order.getId(), total);
}

// AFTER — each section extracted
public OrderConfirmation processOrder(Customer customer, List<Item> items) {
    validateCustomer(customer);
    double subtotal = calculateSubtotal(items);
    double discount = calculateDiscount(items.size(), subtotal);
    double total = applyTaxAndDiscount(subtotal, discount);
    Order order = saveOrder(customer, total);
    emailService.sendConfirmation(customer.getEmail(), order.getId(), total);
    return new OrderConfirmation(order.getId(), total);
}

private void validateCustomer(Customer customer) {
    if (customer == null) throw new IllegalArgumentException("Customer must not be null");
    if (customer.getEmail() == null || customer.getEmail().isEmpty())
        throw new IllegalArgumentException("Customer email is required");
    if (!customer.getEmail().contains("@"))
        throw new IllegalArgumentException("Customer email is invalid: " + customer.getEmail());
}

private double calculateSubtotal(List<Item> items) {
    return items.stream()
        .mapToDouble(i -> i.getPrice() * i.getQuantity())
        .sum();
}

private double calculateDiscount(int itemCount, double subtotal) {
    if (itemCount > 100) return subtotal * VOLUME_DISCOUNT_LARGE;
    if (itemCount > 50)  return subtotal * VOLUME_DISCOUNT_MEDIUM;
    return 0.0;
}

private double applyTaxAndDiscount(double subtotal, double discount) {
    return (subtotal * (1 + TAX_RATE)) - discount;
}
```

---

## 2. Extract Class / God Class Decomposition

**When**: Class > 300 lines with unrelated responsibilities.

```java
// BEFORE — CustomerService does everything
public class CustomerService {
    // Customer CRUD
    public Customer findById(String id) { ... }
    public void save(Customer c) { ... }
    public void delete(String id) { ... }

    // Email notifications
    public void sendWelcomeEmail(Customer c) { ... }
    public void sendPasswordReset(String email) { ... }
    public void sendOrderConfirmation(Customer c, Order o) { ... }

    // Reporting
    public List<Customer> getTopCustomers(int limit) { ... }
    public Map<String, Integer> getOrdersByRegion() { ... }
    public BigDecimal getTotalRevenue() { ... }

    // Validation
    public boolean isValidEmail(String email) { ... }
    public boolean isValidPhoneNumber(String phone) { ... }
}

// AFTER — separate classes, each with one responsibility
public class CustomerRepository {      // Persistence
    public Customer findById(String id) { ... }
    public void save(Customer c) { ... }
    public void delete(String id) { ... }
}

public class CustomerNotificationService { // Notifications
    private final EmailSender emailSender;
    public void sendWelcomeEmail(Customer c) { ... }
    public void sendPasswordReset(String email) { ... }
    public void sendOrderConfirmation(Customer c, Order o) { ... }
}

public class CustomerReportService {   // Reporting
    private final CustomerRepository repo;
    public List<Customer> getTopCustomers(int limit) { ... }
    public Map<String, Integer> getOrdersByRegion() { ... }
    public BigDecimal getTotalRevenue() { ... }
}

public class CustomerValidator {       // Validation
    public void validateEmail(String email) { ... }  // Throws, not returns bool
    public void validatePhoneNumber(String phone) { ... }
}
```

---

## 3. Replace Magic Numbers with Constants

```java
// BEFORE
double discount = 0;
if (items.size() > 100) discount = total * 0.05;
else if (items.size() > 50) discount = total * 0.025;
double tax = subtotal * 0.08;
Thread.sleep(30000);

// AFTER
private static final int LARGE_ORDER_THRESHOLD = 100;
private static final int MEDIUM_ORDER_THRESHOLD = 50;
private static final double LARGE_ORDER_DISCOUNT_RATE = 0.05;
private static final double MEDIUM_ORDER_DISCOUNT_RATE = 0.025;
private static final double TAX_RATE = 0.08;
private static final long RETRY_DELAY_MS = 30_000L;

double discount = 0;
if (items.size() > LARGE_ORDER_THRESHOLD)       discount = total * LARGE_ORDER_DISCOUNT_RATE;
else if (items.size() > MEDIUM_ORDER_THRESHOLD) discount = total * MEDIUM_ORDER_DISCOUNT_RATE;
double tax = subtotal * TAX_RATE;
Thread.sleep(RETRY_DELAY_MS);
```

---

## 4. Replace Primitive Obsession with Value Objects

```java
// BEFORE — primitives scattered everywhere
public void createOrder(String customerId, String email, double price, String currencyCode) { ... }
public boolean validateEmail(String email) { ... }
public double convertCurrency(double amount, String fromCurrency, String toCurrency) { ... }

// AFTER — Value Objects with built-in validation
public final class Email {
    private final String value;

    public Email(String value) {
        if (value == null || !value.contains("@"))
            throw new IllegalArgumentException("Invalid email: " + value);
        this.value = value.toLowerCase().trim();
    }

    public String getValue() { return value; }

    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { return value; }
}

public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount must be non-negative");
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = Objects.requireNonNull(currency);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Cannot add different currencies");
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(double factor) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(factor)), this.currency);
    }
}

// Usage is now self-documenting
public void createOrder(CustomerId customerId, Email email, Money price) { ... }
```

---

## 5. Replace Raw Types with Generics

```java
// BEFORE — raw types, unchecked casts everywhere
List orders = new ArrayList();
orders.add(new Order());
orders.add("this is clearly wrong and compiles"); // ← runtime ClassCastException
for (Object obj : orders) {
    Order order = (Order) obj; // ClassCastException waiting to happen
    process(order);
}

Map cache = new HashMap();
cache.put("key", new Order());
Order o = (Order) cache.get("key"); // Unchecked cast

// AFTER — parameterized generics
List<Order> orders = new ArrayList<>();
orders.add(new Order());
// orders.add("this is clearly wrong"); ← compile error ✓
for (Order order : orders) {
    process(order); // No cast needed
}

Map<String, Order> cache = new HashMap<>();
cache.put("key", new Order());
Order o = cache.get("key"); // No cast needed
```

---

## 6. Replace Legacy Date/Time

```java
// BEFORE — java.util.Date nightmare
Date now = new Date();
Date tomorrow = new Date(now.getTime() + 86400000L); // Magic number: ms per day
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
String formatted = sdf.format(now); // NOT thread-safe!
Calendar cal = Calendar.getInstance();
cal.setTime(now);
cal.add(Calendar.MONTH, 1);
Date nextMonth = cal.getTime();

// AFTER — java.time (Java 8+), thread-safe and readable
LocalDate today = LocalDate.now();
LocalDate tomorrow = today.plusDays(1);
String formatted = today.format(DateTimeFormatter.ISO_LOCAL_DATE); // Thread-safe
LocalDate nextMonth = today.plusMonths(1);

// For timestamps
Instant now = Instant.now();
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("America/New_York"));

// Comparison
boolean isAfterDeadline = LocalDate.now().isAfter(deadline);
long daysBetween = ChronoUnit.DAYS.between(start, end);

// Converting legacy code that still uses java.util.Date (e.g. from DB/API)
Date legacyDate = resultSet.getDate("created_at");
LocalDate modern = legacyDate.toInstant().atZone(ZoneOffset.UTC).toLocalDate();
```

---

## 7. Fix Exception Handling

```java
// BEFORE — swallowed exceptions, System.out, vague catches
public Order processOrder(OrderRequest req) {
    try {
        Order order = orderRepo.save(req.toOrder());
        emailService.send(order);
        return order;
    } catch (Exception e) {
        e.printStackTrace();         // Never in production
        System.out.println("Error"); // Never in production
        return null;                 // Caller can't tell it failed
    }
}

// AFTER — proper exception handling with logging
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

public Order processOrder(OrderRequest req) {
    Objects.requireNonNull(req, "OrderRequest must not be null");
    
    try {
        Order order = orderRepo.save(req.toOrder());
        emailService.send(order);
        log.info("Order {} processed successfully for customer {}", order.getId(), req.getCustomerId());
        return order;
    } catch (DatabaseException e) {
        log.error("Failed to persist order for customer {}: {}", req.getCustomerId(), e.getMessage(), e);
        throw new OrderProcessingException("Order could not be saved", e);
    } catch (EmailException e) {
        // Order was saved — don't rollback, but log the email failure
        log.warn("Order saved but confirmation email failed for customer {}: {}", 
            req.getCustomerId(), e.getMessage());
        // Re-fetch and return the saved order
        return orderRepo.findByRequestId(req.getRequestId());
    }
}
```

---

## 8. Replace String Concatenation in Loops

```java
// BEFORE — O(n²) because each + creates a new String object
public String buildReport(List<Transaction> transactions) {
    String report = "TRANSACTION REPORT\n";
    report += "==================\n";
    for (Transaction t : transactions) {
        report += t.getDate() + " | " + t.getDescription() + " | $" + t.getAmount() + "\n";
    }
    report += "Total: " + transactions.size() + " transactions\n";
    return report;
}

// AFTER — O(n) with StringBuilder
public String buildReport(List<Transaction> transactions) {
    StringBuilder sb = new StringBuilder();
    sb.append("TRANSACTION REPORT\n")
      .append("==================\n");
    for (Transaction t : transactions) {
        sb.append(t.getDate())
          .append(" | ").append(t.getDescription())
          .append(" | $").append(t.getAmount())
          .append('\n');
    }
    sb.append("Total: ").append(transactions.size()).append(" transactions\n");
    return sb.toString();
}

// Or with Streams (Java 8+, most readable for simple cases)
public String buildReport(List<Transaction> transactions) {
    String header = "TRANSACTION REPORT\n==================\n";
    String rows = transactions.stream()
        .map(t -> t.getDate() + " | " + t.getDescription() + " | $" + t.getAmount())
        .collect(Collectors.joining("\n"));
    return header + rows + "\nTotal: " + transactions.size() + " transactions\n";
}
```

---

## 9. Replace Conditional with Polymorphism / Strategy

```java
// BEFORE — giant switch/if-else that grows with every new type
public double calculateShipping(Order order) {
    if (order.getShippingMethod().equals("STANDARD")) {
        return order.getWeight() * 0.50;
    } else if (order.getShippingMethod().equals("EXPRESS")) {
        return order.getWeight() * 1.25 + 5.00;
    } else if (order.getShippingMethod().equals("OVERNIGHT")) {
        return order.getWeight() * 2.00 + 15.00;
    } else if (order.getShippingMethod().equals("FREE")) {
        return 0.0;
    } else {
        throw new IllegalArgumentException("Unknown shipping: " + order.getShippingMethod());
    }
}

// AFTER — Strategy pattern
public interface ShippingStrategy {
    BigDecimal calculateCost(double weightKg);
}

public class StandardShipping implements ShippingStrategy {
    private static final BigDecimal RATE_PER_KG = new BigDecimal("0.50");

    @Override
    public BigDecimal calculateCost(double weightKg) {
        return RATE_PER_KG.multiply(BigDecimal.valueOf(weightKg));
    }
}

public class ExpressShipping implements ShippingStrategy {
    private static final BigDecimal RATE_PER_KG = new BigDecimal("1.25");
    private static final BigDecimal BASE_FEE = new BigDecimal("5.00");

    @Override
    public BigDecimal calculateCost(double weightKg) {
        return RATE_PER_KG.multiply(BigDecimal.valueOf(weightKg)).add(BASE_FEE);
    }
}

// Factory / Registry to replace the switch
public class ShippingStrategyFactory {
    private static final Map<String, ShippingStrategy> STRATEGIES = Map.of(
        "STANDARD",  new StandardShipping(),
        "EXPRESS",   new ExpressShipping(),
        "OVERNIGHT", new OvernightShipping(),
        "FREE",      weight -> BigDecimal.ZERO
    );

    public static ShippingStrategy forMethod(String method) {
        ShippingStrategy strategy = STRATEGIES.get(method);
        if (strategy == null)
            throw new IllegalArgumentException("Unknown shipping method: " + method);
        return strategy;
    }
}

// Caller
BigDecimal shipping = ShippingStrategyFactory.forMethod(order.getShippingMethod())
                                              .calculateCost(order.getWeight());
```

---

## 10. Introduce Dependency Injection

```java
// BEFORE — hard-wired dependencies, untestable
public class ReportService {
    private final DatabaseRepo repo = new DatabaseRepo("jdbc:mysql://prod-db:3306/reports");
    private final S3Client s3 = new S3Client("us-east-1", System.getenv("AWS_KEY"), System.getenv("AWS_SECRET"));
    private final SlackNotifier slack = new SlackNotifier("https://hooks.slack.com/T00/B00/xxx");
}

// AFTER — constructor injection (no framework needed)
public class ReportService {
    private final ReportRepository repo;
    private final StorageClient storage;
    private final Notifier notifier;

    // For production
    public ReportService(ReportRepository repo, StorageClient storage, Notifier notifier) {
        this.repo = Objects.requireNonNull(repo);
        this.storage = Objects.requireNonNull(storage);
        this.notifier = Objects.requireNonNull(notifier);
    }

    // Convenience factory for production (keeps calling code clean)
    public static ReportService createDefault() {
        return new ReportService(
            new DatabaseRepo(System.getenv("DB_URL")),
            new S3StorageClient(System.getenv("AWS_REGION")),
            new SlackNotifier(System.getenv("SLACK_WEBHOOK_URL"))
        );
    }
}

// Tests use mocks via the constructor
@Test
void generateReport_savesToStorage() {
    var sut = new ReportService(mockRepo, mockStorage, mockNotifier);
    // ...
}
```

---

## 11. Fix Null Handling with Optional

```java
// BEFORE — null returned, callers forget to check
public Customer findCustomerByEmail(String email) {
    return customerRepo.findByEmail(email); // Returns null if not found
}

// Usage — NPE waiting to happen
Customer c = service.findCustomerByEmail(email);
c.sendEmail("Welcome!"); // NPE if c is null

// AFTER — Optional communicates intent
public Optional<Customer> findCustomerByEmail(String email) {
    return Optional.ofNullable(customerRepo.findByEmail(email));
}

// Usage — forced to handle the absent case
Optional<Customer> customerOpt = service.findCustomerByEmail(email);

// Option A: throw business exception if missing
Customer c = customerOpt.orElseThrow(() ->
    new CustomerNotFoundException("No customer with email: " + email));

// Option B: default value
Customer c = customerOpt.orElse(Customer.anonymous());

// Option C: conditional action
customerOpt.ifPresent(c -> c.sendEmail("Welcome!"));

// Option D: transform
String name = customerOpt.map(Customer::getName).orElse("Unknown");
```

---

## 12. Fix equals() and hashCode()

```java
// BEFORE — broken: no hashCode, or wrong equals logic
public class Product {
    private String sku;
    private String name;

    @Override
    public boolean equals(Object o) {
        return this.sku.equals(((Product)o).sku); // NullPointerException, ClassCastException risks
    }
    // hashCode() MISSING — breaks in HashSet/HashMap
}

// AFTER — correct implementation
public class Product {
    private final String sku;
    private final String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Product)) return false;  // Handles null too
        Product that = (Product) o;
        return Objects.equals(this.sku, that.sku); // Null-safe
    }

    @Override
    public int hashCode() {
        return Objects.hash(sku); // Must use same fields as equals!
    }
}

// Alternative: Use Java 16+ record (immutable value object)
public record Product(String sku, String name) {
    // equals(), hashCode(), toString() all generated correctly
    public Product {
        Objects.requireNonNull(sku, "sku is required");
        Objects.requireNonNull(name, "name is required");
    }
}
```

---

## 13. Replace Data Class with Behavior-Rich Object

```java
// BEFORE — anemic data class, logic scattered in service layer
public class Order {
    private String id;
    private List<OrderLine> lines;
    private String status;
    private double totalAmount;
    // getters, setters only
}

// In OrderService:
public void applyDiscount(Order order) {
    if (order.getStatus().equals("PENDING") && order.getLines().size() > 5) {
        order.setTotalAmount(order.getTotalAmount() * 0.9);
    }
}

// AFTER — logic moved into the domain object
public class Order {
    private final String id;
    private final List<OrderLine> lines;
    private OrderStatus status;

    public boolean isEligibleForBulkDiscount() {
        return status == OrderStatus.PENDING && lines.size() > 5;
    }

    public Money calculateTotal() {
        return lines.stream()
            .map(OrderLine::lineTotal)
            .reduce(Money.ZERO, Money::add);
    }

    public Money calculateTotalWithDiscount() {
        Money base = calculateTotal();
        return isEligibleForBulkDiscount() ? base.multiply(0.9) : base;
    }

    public void confirm() {
        if (status != OrderStatus.PENDING)
            throw new IllegalStateException("Only PENDING orders can be confirmed");
        this.status = OrderStatus.CONFIRMED;
    }
}
```

---

## 14. Fix Resource Leaks

```java
// BEFORE — streams, connections not closed
public String readFile(String path) throws IOException {
    FileInputStream fis = new FileInputStream(path);
    BufferedReader reader = new BufferedReader(new InputStreamReader(fis));
    StringBuilder sb = new StringBuilder();
    String line;
    while ((line = reader.readLine()) != null) sb.append(line);
    // If exception thrown above, reader never closed!
    reader.close();
    return sb.toString();
}

// AFTER — try-with-resources guarantees close() on all AutoCloseable
public String readFile(String path) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        StringBuilder sb = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) sb.append(line);
        return sb.toString();
    }
    // reader.close() called automatically even if exception thrown
}

// Modern Java — even simpler
public String readFile(String path) throws IOException {
    return Files.readString(Paths.get(path)); // Java 11+
}

// For DB connections (very common legacy bug)
// BEFORE
Connection conn = dataSource.getConnection();
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT ...");
// ... process ... (if exception: conn never closed → connection pool leak)
rs.close(); stmt.close(); conn.close();

// AFTER
try (Connection conn = dataSource.getConnection();
     Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("SELECT ...")) {
    // All three closed automatically
}
```

---

## 15. Fix Financial Calculations

```java
// BEFORE — floating point errors in money
double price = 19.99;
double tax = price * 0.08;
double total = price + tax; // 21.5892 — will round unpredictably
String formatted = "$" + total; // "$21.589199999999998" — embarrassing

// AFTER — BigDecimal for all financial math
BigDecimal price = new BigDecimal("19.99");
BigDecimal taxRate = new BigDecimal("0.08");
BigDecimal tax = price.multiply(taxRate).setScale(2, RoundingMode.HALF_UP);
BigDecimal total = price.add(tax);
String formatted = "$" + total.toPlainString(); // "$21.59" — correct

// Money class pattern (use your Value Object from refactor #4)
Money price = new Money(new BigDecimal("19.99"), Currency.getInstance("USD"));
Money tax = price.multiply(0.08);
Money total = price.add(tax);
// toString() → "$21.59"

// Key rounding rules:
// HALF_UP: 2.5 → 3, 2.45 → 2.5 → rounds to 2 (half-up on second decimal)
//          Use for financial "round half up" (common in retail)
// HALF_EVEN: 2.5 → 2, 3.5 → 4 (Banker's rounding, minimizes statistical bias)
//            Use for accounting and scientific calculations
// CEILING: Always rounds toward positive infinity (use for taxes charged)
// FLOOR:   Always rounds toward negative infinity (use for interest credited)
```

---

## 16. Reduce Cyclomatic Complexity

**When**: PMD/SonarQube reports CC > 10. Or any method with 5+ nested levels.

### Guard Clauses (Early Return)

```java
// BEFORE — CC = 8, 4 levels deep
public double applyDiscount(Customer customer, Order order) {
    double discount = 0;
    if (customer != null) {
        if (order != null) {
            if (customer.isPremium()) {
                if (order.getTotal() > 100) {
                    discount = order.getTotal() * 0.15;
                } else {
                    discount = order.getTotal() * 0.10;
                }
            } else {
                if (order.getTotal() > 200) {
                    discount = order.getTotal() * 0.05;
                }
            }
        }
    }
    return discount;
}

// AFTER — CC = 4, flat structure
public double applyDiscount(Customer customer, Order order) {
    if (customer == null || order == null) return 0;

    if (customer.isPremium()) {
        return order.getTotal() > 100
            ? order.getTotal() * PREMIUM_HIGH_DISCOUNT
            : order.getTotal() * PREMIUM_LOW_DISCOUNT;
    }

    return order.getTotal() > 200 ? order.getTotal() * STANDARD_DISCOUNT : 0;
}
```

### Decompose Conditional into Named Methods

```java
// BEFORE — one method tries to express all logic inline
public String categorizeRisk(Loan loan) {
    if (loan.getAmount() > 500_000 && loan.getDurationYears() > 10
            && (loan.getApplicant().getCreditScore() < 650
                || loan.getApplicant().getDebtToIncomeRatio() > 0.43)
            && !loan.hasCollateral()) {
        return "HIGH";
    } else if (...) { ... }
}

// AFTER — each condition has a name that explains its meaning
public String categorizeRisk(Loan loan) {
    if (isHighRisk(loan))   return "HIGH";
    if (isMediumRisk(loan)) return "MEDIUM";
    return "LOW";
}

private boolean isHighRisk(Loan loan) {
    return isLargeLongTermLoan(loan)
        && hasWeakCreditProfile(loan.getApplicant())
        && !loan.hasCollateral();
}

private boolean isLargeLongTermLoan(Loan loan) {
    return loan.getAmount() > 500_000 && loan.getDurationYears() > 10;
}

private boolean hasWeakCreditProfile(Applicant a) {
    return a.getCreditScore() < MINIMUM_CREDIT_SCORE
        || a.getDebtToIncomeRatio() > MAX_DTI_RATIO;
}
```

---

## 17. Remove Over-Engineering

**When**: Interface with a single implementation, abstract classes for one subclass, factory for one product, layers of delegation with no behavior.

### Remove Single-Implementation Interface

```java
// BEFORE — interface exists "just in case"
public interface IOrderValidator {
    ValidationResult validate(Order order);
}
@Component
public class OrderValidatorImpl implements IOrderValidator {
    public ValidationResult validate(Order order) { ... }
}
// Usage: IOrderValidator validator (always only one impl)

// AFTER — direct class (add interface when second impl is actually needed)
@Component
public class OrderValidator {
    public ValidationResult validate(Order order) { ... }
}
```

### Collapse Pointless Delegation Chain

```java
// BEFORE — A calls B calls C, B adds nothing
public class OrderFacade {
    private final OrderManagerDelegate delegate;
    public Order createOrder(OrderRequest req) {
        return delegate.handleOrderCreation(req);  // Just passes through
    }
}
public class OrderManagerDelegate {
    private final OrderService service;
    public Order handleOrderCreation(OrderRequest req) {
        return service.create(req);  // Also just passes through
    }
}

// AFTER — call the service directly
public class OrderController {
    private final OrderService orderService;
    // Controllers call services. No middlemen.
}
```

### Replace Abstract Factory with Direct Construction

```java
// BEFORE — factory for one product, always returns same type
public abstract class NotificationFactory {
    public abstract Notification create(String message);
}
public class EmailNotificationFactory extends NotificationFactory {
    public Notification create(String message) { return new EmailNotification(message); }
}
// Usage: new EmailNotificationFactory().create("Hello") — why not just new EmailNotification()?

// AFTER — remove factory if there's only one product and no runtime variation
EmailNotification notification = new EmailNotification(message);
// Or if Spring-managed:
notificationService.send(message); // Service handles the type internally
```



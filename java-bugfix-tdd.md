---
name: java-bugfix-tdd
description: >
  Use this skill whenever the user wants to analyze, audit, or fix a Java codebase for bugs, 
  especially critical runtime errors. Triggers include: NPE (NullPointerException), Hibernate errors, 
  orphan removal issues, LazyInitializationException, ConcurrentModificationException, 
  ClassCastException, StackOverflow, memory leaks, transaction boundary bugs, equals/hashCode 
  contract violations, or any request to "find bugs", "fix errors", "audit Java code", "review 
  Spring/Hibernate/JPA entities", or "add tests to prevent regressions". Always applies TDD: 
  every fix is accompanied by a failing test written first, then the fix, with the test suite 
  acting as a regression guard. Use even when the user just pastes Java code and asks what's wrong.
---

# Java Codebase Bug Analysis & TDD Fix Skill

You are an expert Java engineer applying Test-Driven Development (TDD) to every fix. Your workflow is always:
1. **Identify** the bug category and root cause  
2. **Write a failing test** that reproduces the bug  
3. **Apply the minimal fix** to make the test pass  
4. **Verify** no existing behavior breaks  

---

## Environment Detection & Setup

### Detect OS first (critical for commands)
```bash
# Run this to detect platform
python3 -c "import platform; print(platform.system())"
# OR
uname -s 2>/dev/null || echo "Windows"
```

Use the correct path separator and commands per OS:

| Task | Linux/Mac | Windows (CMD) | Windows (PowerShell) |
|------|-----------|---------------|----------------------|
| List files | `ls -la` | `dir` | `Get-ChildItem` |
| Find files | `find . -name "*.java"` | `dir /s /b *.java` | `Get-ChildItem -Recurse -Filter *.java` |
| Run Maven | `./mvnw` | `mvnw.cmd` | `.\mvnw.cmd` |
| Run Gradle | `./gradlew` | `gradlew.bat` | `.\gradlew.bat` |
| Copy file | `cp src dst` | `copy src dst` | `Copy-Item src dst` |

### Detect build tool
```bash
# Check for Maven
ls pom.xml 2>/dev/null && echo "MAVEN"
# Check for Gradle
ls build.gradle build.gradle.kts 2>/dev/null && echo "GRADLE"
```

### Run tests
```bash
# Maven
./mvnw test -Dtest=TargetTestClass          # Linux/Mac
mvnw.cmd test -Dtest=TargetTestClass        # Windows CMD
.\mvnw.cmd test -Dtest=TargetTestClass      # Windows PowerShell

# Gradle
./gradlew test --tests "*.TargetTestClass"  # Linux/Mac
gradlew.bat test --tests "*.TargetTestClass" # Windows CMD
```

---

## Step 1 — Codebase Scan

Before fixing anything, scan the codebase:

```bash
# Collect all Java files for analysis
find . -name "*.java" -not -path "*/target/*" -not -path "*/.gradle/*" -not -path "*/build/*"
# Windows PowerShell equivalent:
# Get-ChildItem -Recurse -Filter *.java | Where-Object { $_.FullName -notmatch 'target|\.gradle|build' }
```

Read and categorize files:
- **Entities** (`@Entity`, `@Table`) → Hibernate bug surface
- **Repositories** (`@Repository`, extends `JpaRepository`) → query bugs
- **Services** (`@Service`, `@Transactional`) → transaction/session scope bugs
- **Controllers** (`@RestController`, `@Controller`) → NPE hotspots
- **Utilities / Helpers** → null handling, collections misuse

---

## Step 2 — Bug Detection Checklist

Work through each category systematically. For each bug found, note:
- **File + Line** (or method)
- **Bug Type** (from table below)
- **Severity**: CRITICAL / HIGH / MEDIUM
- **Fix Strategy**

### 2A — NullPointerException (NPE) Patterns

| Pattern | Detection signal | Fix |
|---------|-----------------|-----|
| Unchecked return from method | `obj.method()` without null-check when method can return null | `Optional<>`, null-guard, or `Objects.requireNonNull` |
| Uninitialized collection field | `List<X> items;` then `items.add(...)` | Initialize: `= new ArrayList<>()` |
| Chained calls | `a.getB().getC().getValue()` | Break chain, null-check each step, or use `Optional` |
| `@Autowired` field in non-Spring context | Unit test creates `new Service()` without injecting deps | Use constructor injection; inject mocks in test |
| Returning `null` from `@NonNull` method | Method contract violated | Return `Optional<>` or throw specific exception |
| Map.get() result used directly | `map.get(key).doSomething()` | `map.getOrDefault(key, fallback)` or null-check |

### 2B — Hibernate / JPA Errors

| Bug | Root Cause | Fix |
|-----|-----------|-----|
| `LazyInitializationException` | Accessing lazy collection outside open Session/transaction | Add `@Transactional` on calling service; use `JOIN FETCH`; or change to `EAGER` where appropriate |
| `org.hibernate.StaleObjectStateException` | Optimistic lock conflict — entity modified between read and write | Retry logic; add `@Version` field; review transaction scope |
| `DetachedObjectException` | Passing a detached entity to merge/persist in another session | Use `entityManager.merge()`; re-load entity in new tx |
| Orphan removal error | Removing child from parent collection without `orphanRemoval=true`, or `orphanRemoval=true` + re-attaching detached child | See section 2C |
| `NonUniqueResultException` | `getSingleResult()` when query returns >1 row | Use `getResultList()` + validate, or fix JPQL |
| `PersistentObjectException: detached entity passed to persist` | Calling `persist()` on an entity with an existing ID | Use `save()` (Spring Data) or `merge()` |
| `ConstraintViolationException` | FK violation, unique violation at DB level | Validate before persist; handle and translate exception |
| `MultipleBagFetchException` | Two `EAGER` `@OneToMany` collections on same entity | Change one to `Set`; or fetch in separate queries |
| Equals/hashCode using mutable DB ID | Entities in `Set` misbehave after persist (ID assigned) | Implement equals/hashCode on **business key**, not `id` |
| Missing `@Transactional` on write operations | Changes not flushed, or `TransactionRequiredException` | Add `@Transactional` to service methods that write |
| N+1 query problem | Lazy collections triggered in a loop | Use `JOIN FETCH`, `@BatchSize`, or DTO projection |

### 2C — Orphan Removal Deep Dive

This is one of the most misunderstood Hibernate features. Check for ALL of these:

**Scenario 1 — Missing `orphanRemoval=true`**
```java
// BUG: child rows become orphans in DB (no FK parent)
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
private List<Child> children;

// FIX:
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Child> children;
```

**Scenario 2 — Replacing the collection reference**
```java
// BUG: Hibernate tracks the original collection proxy; replacing it loses orphan tracking
parent.setChildren(new ArrayList<>(newChildren)); // ← BREAKS orphanRemoval

// FIX: always mutate the existing collection
parent.getChildren().clear();
parent.getChildren().addAll(newChildren);
```

**Scenario 3 — Detached child re-added to different parent**
```java
// BUG: child has orphanRemoval but is being moved between parents
// Hibernate sees child "removed" from parent A and tries to delete it,
// but it's now on parent B → ConstraintViolationException or data loss

// FIX: load child in current session before re-parenting
Child managed = entityManager.find(Child.class, child.getId());
managed.setParent(newParent);
newParent.getChildren().add(managed);
```

**Scenario 4 — `orphanRemoval` + DTO update pattern**
```java
// BUG: service receives DTO, creates new Child objects, replaces collection
// Result: all existing children deleted and re-inserted (breaks FKs, audit trails)
public void update(ParentDTO dto) {
    Parent p = repo.findById(dto.getId()).orElseThrow();
    p.setChildren(dto.getChildren().stream()
        .map(Child::new).collect(toList())); // ← DESTROYS DB rows
}

// FIX: merge by ID — update existing, add new, remove missing
public void update(ParentDTO dto) {
    Parent p = repo.findById(dto.getId()).orElseThrow();
    Set<Long> incomingIds = dto.getChildren().stream()
        .map(ChildDTO::getId).filter(Objects::nonNull).collect(toSet());
    // remove children not in DTO
    p.getChildren().removeIf(c -> !incomingIds.contains(c.getId()));
    // update existing + add new
    for (ChildDTO cdto : dto.getChildren()) {
        if (cdto.getId() == null) {
            Child c = new Child(cdto);
            c.setParent(p);
            p.getChildren().add(c);
        } else {
            p.getChildren().stream()
                .filter(c -> c.getId().equals(cdto.getId()))
                .findFirst().ifPresent(c -> c.updateFrom(cdto));
        }
    }
}
```

### 2D — Concurrency & Collections

| Bug | Signal | Fix |
|-----|--------|-----|
| `ConcurrentModificationException` | Iterating a list while adding/removing inside loop | Use `Iterator.remove()`, copy list first, or use `removeIf()` |
| Race condition on shared mutable state | Non-final static field mutated, or `@Service` field written at runtime | Make stateless; use `ThreadLocal`; synchronize properly |
| `HashMap` in concurrent context | `HashMap` used in multi-threaded code | Replace with `ConcurrentHashMap` |
| Non-atomic check-then-act | `if (!map.containsKey(k)) map.put(k, v)` in concurrent code | Use `map.computeIfAbsent(k, ...)` |

### 2E — equals / hashCode Contract

```java
// BUG: Using Lombok @Data or @EqualsAndHashCode on @Entity
// Lombok generates equals/hashCode using all fields including mutable id
// This breaks Sets, Maps, and Hibernate's identity tracking

@Entity
@Data // ← DANGER on entities
public class Order { ... }

// FIX: Explicit business-key equals/hashCode
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String orderNumber; // business key
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order that = (Order) o;
        return Objects.equals(orderNumber, that.orderNumber);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(orderNumber);
    }
}
```

### 2F — Transaction Boundary Bugs

```java
// BUG: @Transactional on private method — Spring AOP proxy bypasses it
@Transactional
private void saveData() { ... } // ← NO PROXY, no transaction

// BUG: self-invocation — calling @Transactional method from same bean
public void doWork() {
    this.transactionalMethod(); // ← calls real method, not proxy
}

// FIX: move transactional method to separate bean, or inject self via ApplicationContext
// FIX for private: make method package-private or public

// BUG: Exception swallowed → transaction not rolled back
@Transactional
public void save() {
    try { repo.save(entity); }
    catch (Exception e) { log.error("...", e); } // ← swallowed! tx commits
}

// FIX: rethrow or use TransactionAspectSupport
catch (Exception e) {
    TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    throw e;
}
```

---

## Step 3 — TDD Fix Protocol

For **every bug found**, follow this exact order. Do not skip steps.

### 3A — Write the Failing Test First

Place tests in the correct directory:
```
src/test/java/<same-package-as-class-under-test>/
```

Test naming convention: `<ClassUnderTest>Test.java` or `<ClassUnderTest>BugFixTest.java`

#### Test Template — Unit Test (plain Java, no Spring context)
```java
package com.example.service;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService;

    @Test
    @DisplayName("BUG-001: findById returns empty Optional — must not throw NPE when result is absent")
    void findById_whenNotFound_shouldReturnEmptyOptional() {
        // Arrange
        when(orderRepository.findById(99L)).thenReturn(Optional.empty());

        // Act & Assert — must NOT throw NullPointerException
        assertThatCode(() -> orderService.findById(99L))
            .doesNotThrowAnyException();

        Optional<Order> result = orderService.findById(99L);
        assertThat(result).isEmpty();
    }
}
```

#### Test Template — Hibernate/JPA Integration Test (Spring Boot)
```java
package com.example.repository;

import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.*;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@ActiveProfiles("test")
@DisplayName("Hibernate orphan removal regression tests")
class ParentRepositoryTest {

    @Autowired
    private ParentRepository parentRepository;

    @Autowired
    private TestEntityManager em;

    @Test
    @DisplayName("BUG-002: Removing child from collection must delete the child row (orphanRemoval)")
    void removeChildFromCollection_shouldDeleteChildRow() {
        // Arrange
        Parent parent = new Parent("P1");
        Child child = new Child("C1");
        parent.addChild(child);
        parentRepository.save(parent);
        em.flush(); em.clear();

        // Act
        Parent managed = parentRepository.findById(parent.getId()).orElseThrow();
        managed.getChildren().clear();
        parentRepository.save(managed);
        em.flush(); em.clear();

        // Assert — child must be gone from DB (not just from collection)
        Parent reloaded = parentRepository.findById(parent.getId()).orElseThrow();
        assertThat(reloaded.getChildren()).isEmpty();
        // Also verify at raw DB level
        Long childCount = em.getEntityManager()
            .createQuery("SELECT COUNT(c) FROM Child c WHERE c.parent.id = :id", Long.class)
            .setParameter("id", parent.getId())
            .getSingleResult();
        assertThat(childCount).isZero();
    }

    @Test
    @DisplayName("BUG-003: Replacing collection reference must not break orphan removal")
    void replacingCollectionReference_mustNotBypassOrphanRemoval() {
        // Arrange
        Parent parent = new Parent("P2");
        parent.addChild(new Child("OldChild"));
        parentRepository.save(parent);
        em.flush(); em.clear();

        // Act — simulate the broken pattern then verify correct pattern
        Parent managed = parentRepository.findById(parent.getId()).orElseThrow();
        // Correct fix: mutate, don't replace
        managed.getChildren().clear();
        managed.getChildren().add(new Child("NewChild"));
        parentRepository.save(managed);
        em.flush(); em.clear();

        // Assert
        Parent reloaded = parentRepository.findById(parent.getId()).orElseThrow();
        assertThat(reloaded.getChildren())
            .hasSize(1)
            .extracting(Child::getName)
            .containsExactly("NewChild");
    }
}
```

#### Test Template — LazyInitializationException
```java
@Test
@DisplayName("BUG-004: Accessing lazy collection outside transaction must not throw LazyInitializationException")
void accessLazyCollection_withinTransaction_shouldNotThrow() {
    // This test must pass — if it throws LazyInitializationException,
    // the @Transactional on the service method is missing or misconfigured
    assertThatCode(() -> {
        List<Child> children = parentService.getChildrenForParent(savedParentId);
        // Force collection initialization
        assertThat(children).isNotNull();
    }).doesNotThrowAnyException();
}
```

#### Test Template — ConcurrentModificationException
```java
@Test
@DisplayName("BUG-005: Modifying collection during iteration must not throw ConcurrentModificationException")
void removeWhileIterating_shouldNotThrowConcurrentModificationException() {
    List<String> items = new ArrayList<>(List.of("a", "b", "c", "b", "d"));
    
    assertThatCode(() -> items.removeIf(s -> s.equals("b")))
        .doesNotThrowAnyException();
    
    assertThat(items).containsExactly("a", "c", "d");
}
```

#### Test Template — equals/hashCode in Set
```java
@Test
@DisplayName("BUG-006: Entity equals/hashCode must be stable across persist (business key, not surrogate ID)")
void entity_addedToSetBeforeAndAfterPersist_mustBeFoundByBusinessKey() {
    Set<Order> orders = new HashSet<>();
    Order order = new Order("ORD-001");
    orders.add(order); // added before persist (id=null)
    
    // simulate persist assigning an ID
    order = orderRepository.save(order);
    
    // must still be findable in the Set after ID is assigned
    Order lookup = new Order("ORD-001");
    assertThat(orders).contains(lookup);
    assertThat(orders).hasSize(1); // no duplicates
}
```

### 3B — Confirm Test Fails Before Fix

```bash
# Must see FAILED before applying the fix — this proves the test catches the bug
./mvnw test -Dtest=OrderServiceTest#findById_whenNotFound_shouldReturnEmptyOptional
```

If the test passes before the fix → the test is wrong; revise it.

### 3C — Apply the Fix

Make the **minimal** code change. Do not refactor unrelated code in the same commit.

### 3D — Confirm Test Passes After Fix

```bash
./mvnw test -Dtest=OrderServiceTest
```

### 3E — Run Full Test Suite

```bash
./mvnw test
# Windows: mvnw.cmd test
```

Ensure zero regressions. If existing tests break, investigate before proceeding.

---

## Step 4 — Test Coverage for Critical Paths

After all fixes, verify coverage on repaired classes:

```bash
# Maven with JaCoCo
./mvnw test jacoco:report
# Report at: target/site/jacoco/index.html

# Windows:
mvnw.cmd test jacoco:report
```

Minimum targets for fixed classes:
- **Entities**: 100% equals/hashCode, all relationship methods
- **Services**: 80%+ line coverage including error paths
- **Repositories**: all custom queries have at least one passing + one edge-case test

---

## Step 5 — Output Format

Present findings as:

```
### Bug Report

| # | File | Method/Line | Type | Severity | Status |
|---|------|-------------|------|----------|--------|
| 1 | OrderService.java | findById():47 | NPE | CRITICAL | FIXED ✅ |
| 2 | Parent.java | @OneToMany | OrphanRemoval | HIGH | FIXED ✅ |
| 3 | ItemService.java | updateItems():82 | Collection replace | HIGH | FIXED ✅ |

### Tests Added

| Test Class | Test Method | Covers |
|------------|-------------|--------|
| OrderServiceTest | findById_whenNotFound_shouldReturnEmptyOptional | NPE #1 |
| ParentRepositoryTest | removeChildFromCollection_shouldDeleteChildRow | Orphan #2 |
| ParentRepositoryTest | replacingCollectionReference_mustNotBypassOrphanRemoval | Collection #3 |

### Test Run Results
All 3 new tests: PASS ✅  
Full suite: X passed, 0 failed, 0 errors
```

---

## Step 6 — Regression Guard Checklist

Before declaring done, confirm each item:

- [ ] Every bug has at least one test that **would have caught it** before the fix
- [ ] Each test has a `@DisplayName` starting with `BUG-NNN:` describing the original error
- [ ] No test uses `Mockito.any()` in a way that would pass even with the broken code
- [ ] All Hibernate tests use `em.flush(); em.clear()` after writes to force DB round-trip
- [ ] Transaction tests verify behavior at the **service layer**, not only at repository layer
- [ ] Full suite passes on clean build: `./mvnw clean test` (or `mvnw.cmd clean test` on Windows)

---

## Common Dependencies Reference

If tests are missing dependencies, add to `pom.xml` / `build.gradle`:

### Maven (pom.xml)
```xml
<!-- JUnit 5 + AssertJ + Mockito — usually brought in by spring-boot-starter-test -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- For @DataJpaTest with H2 in-memory -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

### Gradle (build.gradle)
```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
testRuntimeOnly 'com.h2database:h2'
```

### Gradle Kotlin DSL (build.gradle.kts)
```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-test")
testRuntimeOnly("com.h2database:h2")
```

---

## Anti-Patterns to Flag (Do Not Fix Without Tests)

These patterns are bugs waiting to happen. Flag them even if not currently failing:

1. `@Entity` class with `@Data` (Lombok) — equals/hashCode disaster
2. `Optional.get()` without `isPresent()` or `orElseThrow()`
3. `@OneToMany` with `cascade = CascadeType.ALL` but no `orphanRemoval`
4. `@Transactional` on `private` methods
5. Returning `null` from a method that callers chain on
6. `new ArrayList<>()` inside a loop that's a field, resetting state each iteration
7. Catching `Exception` and not rethrowing or marking tx for rollback
8. `entityManager.find()` result used without null check
9. Bidirectional relationship without a helper method to keep both sides in sync:
   ```java
   // Correct helper:
   public void addChild(Child c) {
       children.add(c);
       c.setParent(this);
   }
   public void removeChild(Child c) {
       children.remove(c);
       c.setParent(null);
   }
   ```
10. `@Version` missing on entities used in concurrent update scenarios

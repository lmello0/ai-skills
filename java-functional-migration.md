---
name: java-functional-migration
description: >
  Use this skill when the user wants to refactor or migrate a Java codebase toward a more functional style —
  without rewriting everything at once. Triggers include: "make my Java code more functional", "apply immutability
  to my Java classes", "remove setters from entities", "move side effects to the edges", "refactor Java to use
  Optional instead of null", "make Java domain objects immutable", "functional Java refactoring", "clean up Java
  OOP code", or any request to improve a Java codebase using functional programming principles. Also use when
  the user shares Java code and asks for a review or rewrite with cleaner architecture, reduced mutation, or
  better separation of concerns. Trigger even if the user only mentions one principle (e.g. "reduce setters in
  my Java code") — the full skill context will help apply it correctly alongside related principles.
---

# Java OOP → Functional Migration Skill

A pragmatic, incremental guide to moving a Java codebase toward functional style.
**Goal: reduce accidental complexity, not add it.** Apply principles progressively. Never rewrite everything at once.

---

## Step 0: Clarify Scope Before Anything Else

**Before writing a single line of code or test, ask the user which principles they want applied.**

Even if the user says "make this more functional", that's not enough to act on — applying all principles at once to an unfamiliar codebase creates noise and risk. Instead, present the available principles and let them choose.

Ask exactly this (adapt wording to match conversation tone, but cover all options):

> Before I start, which of these functional principles would you like me to apply?
> You can pick one, several, or all of them:
>
> 1. **Immutability** — make fields `final`, construct objects fully via constructor or change the class to record (if possible)
> 2. **Non-mutating functions** — methods return new objects instead of modifying their input
> 3. **Side effects at the edges** — push I/O, DB calls, and `Instant.now()` out of domain logic
> 4. **Intention-revealing methods** — replace raw setter clusters with named methods that say *why* (e.g. `user.approve(instant)` instead of two setters)
> 5. **Eliminate nulls** — replace `null` returns with `Optional<T>` and empty collections
> 6. **Streams for transformations only** — remove side effects from stream pipelines

Then wait for the answer. Only apply the principles the user selected.

**If the user has already been explicit** (e.g. "just remove the nulls" or "I only want to apply immutability"), skip this step and proceed with what they asked.

---

## Step 1: The Golden Rule — Tests First, Always

**A refactor is not complete until all tests pass. No exceptions.**

Before touching any production code:

1. **If tests already exist** — run them. Confirm they pass on the original code. Only then refactor, and only consider the work done when they pass again.
2. **If tests do not exist** — write them first, covering the current behavior, including edge cases and expected exceptions. Make them pass on the original code. Then refactor.

This is non-negotiable. A refactor that changes behavior is a bug, not an improvement.

### Writing the Characterization Tests

When the user's code has no tests, write JUnit 5 tests that pin the existing behavior before changing anything. These are called *characterization tests* — they describe what the code *does*, not what it *should* do.

```java
// Example: before refactoring User.approve(), write this first
class UserApprovalTest {

    @Test
    void approving_a_pending_user_sets_status_to_approved() {
        var user = new User(1L, "Alice", UserStatus.PENDING, null);
        user.setStatus(UserStatus.APPROVED);          // original behavior
        user.setApprovedAt(Instant.parse("2024-01-15T10:00:00Z"));

        assertThat(user.getStatus()).isEqualTo(UserStatus.APPROVED);
        assertThat(user.getApprovedAt()).isEqualTo(Instant.parse("2024-01-15T10:00:00Z"));
    }

    @Test
    void approving_a_blocked_user_throws() {
        // If this invariant exists in the original code, capture it.
        // If it doesn't, do NOT add it — that's a behavior change.
        var user = new User(1L, "Alice", UserStatus.BLOCKED, null);

        assertThatThrownBy(() -> user.approve(Instant.now()))
            .isInstanceOf(IllegalStateException.class);
    }
}
```

**Write tests against the original API.** After refactoring, update only what the API surface forces you to update (e.g. if a setter becomes a named method, update the test call — but assert the same outcome).

### What to Cover in Tests Before Refactoring

For each class you plan to change, write tests for:

- **Happy path**: normal state transitions and return values
- **Guard conditions**: any `if`/`throw` inside the method
- **Null/empty inputs**: what happens with null fields or empty collections
- **State after mutation**: if a method mutates, assert the state after
- **Unchanged fields**: assert that fields you didn't intend to touch are still the same

---

## Core Philosophy

- **Immutability by default** — objects should not change after construction unless there's a clear reason.
- **Functions transform, not mutate** — prefer returning new values over modifying the input.
- **Side effects at the edges** — I/O, DB calls, clocks, randomness belong at the entry points (controllers, use cases), not inside domain logic.
- **Intention-revealing mutations** — when state must change, wrap it in a named method that says *why*, not just *what*.
- **No nulls** — use `Optional`, or return empty collections, to make absence explicit.
- **Streams for transformations only** — not for control flow or iteration with side effects.

---

## Principles & How to Apply Them

### 1. Immutability

Make fields `final` wherever possible. Provide all values via constructor.

```java
// Before
public class Order {
    private String id;
    private List<OrderItem> items;
    public void setId(String id) { this.id = id; }
    public void setItems(List<OrderItem> items) { this.items = items; }
}

// Test first — pin existing construction behavior:
@Test
void order_stores_id_and_items() {
    var items = List.of(new OrderItem("SKU-1", 2));
    var order = new Order();
    order.setId("ORD-1");
    order.setItems(items);

    assertThat(order.getId()).isEqualTo("ORD-1");
    assertThat(order.getItems()).hasSize(1);
}

// After (refactor — test must still pass with updated construction call)
public class Order {
    private final String id;
    private final List<OrderItem> items;

    public Order(String id, List<OrderItem> items) {
        this.id = id;
        this.items = List.copyOf(items); // defensive copy
    }
}
```

**When you can't make everything final:** identify the fields that change and protect the ones that don't. Partial immutability is still progress.

---

### 2. Functions That Don't Mutate Input

When a method needs to "change" an object, return a new one instead.

```java
// Before
public void applyDiscount(Order order, BigDecimal percent) {
    order.setTotalPrice(order.getTotalPrice().multiply(percent));
}

// Test first:
@Test
void applying_discount_reduces_total_price() {
    var order = orderWithTotal(new BigDecimal("100.00"));
    applyDiscount(order, new BigDecimal("0.9"));

    assertThat(order.getTotalPrice()).isEqualByComparingTo("90.00");
}

// After
public Order withDiscount(Order order, BigDecimal percent) {
    var newTotal = order.getTotalPrice().multiply(percent);
    return new Order(order.getId(), order.getItems(), newTotal);
}

// Updated test — same assertion, updated call:
@Test
void applying_discount_returns_order_with_reduced_price() {
    var order = orderWithTotal(new BigDecimal("100.00"));
    var discounted = withDiscount(order, new BigDecimal("0.9"));

    assertThat(discounted.getTotalPrice()).isEqualByComparingTo("90.00");
    assertThat(order.getTotalPrice()).isEqualByComparingTo("100.00"); // original unchanged
}
```

Use `with*` naming for copy-and-modify methods. The extra assertion on the original is now possible and valuable — it proves no mutation happened.

---

### 3. Side Effects at the Edges

Domain/business logic should be **pure**: same input → same output, no I/O, no DB, no `Instant.now()`.

```java
// Before (business logic + side effect mixed)
public class UserService {
    public void approveUser(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        user.setStatus(UserStatus.APPROVED);
        user.setApprovedAt(Instant.now()); // side effect inside domain
        userRepository.save(user);
    }
}

// Test first (requires mocking the repository and clock):
@Test
void approving_user_saves_approved_status() {
    var user = new User(1L, "Alice", UserStatus.PENDING, null);
    when(userRepository.findById(1L)).thenReturn(Optional.of(user));

    userService.approveUser(1L);

    var captor = ArgumentCaptor.forClass(User.class);
    verify(userRepository).save(captor.capture());
    assertThat(captor.getValue().getStatus()).isEqualTo(UserStatus.APPROVED);
    assertThat(captor.getValue().getApprovedAt()).isNotNull();
}

// After (side effects at the edges, domain logic pure)
public class UserService {
    public void approveUser(Long userId) {
        Instant now = clock.instant();              // side effect: get time (edge)
        User user = userRepository.findById(userId) // side effect: DB read (edge)
            .orElseThrow();
        User approved = user.approve(now);          // pure domain logic
        userRepository.save(approved);              // side effect: DB write (edge)
    }
}

// In the domain entity (pure — easy to test without mocks):
public User approve(Instant approvedAt) {
    if (this.status == UserStatus.BLOCKED) {
        throw new IllegalStateException("Blocked user cannot be approved");
    }
    return new User(this.id, UserStatus.APPROVED, approvedAt, this.name);
}

// Domain test — no mocks needed:
@Test
void approve_returns_user_with_approved_status_and_timestamp() {
    var user = new User(1L, "Alice", UserStatus.PENDING, null);
    var approvedAt = Instant.parse("2024-01-15T10:00:00Z");

    var approved = user.approve(approvedAt);

    assertThat(approved.getStatus()).isEqualTo(UserStatus.APPROVED);
    assertThat(approved.getApprovedAt()).isEqualTo(approvedAt);
}
```

**Rule of thumb:** if a domain method calls `repository`, `clock`, `UUID.randomUUID()`, or any I/O — it's doing too much. Pure domain logic is a test-without-mocks sign: if you need a mock to unit-test it, it has a side effect.

---

### 4. Replace Random Setters with Intention-Revealing Methods

Don't remove all setters at once. Start by replacing **arbitrary field mutation** with named methods that encode *why* the change happens.

```java
// Bad — reader has to infer intent from two separate setters
user.setStatus(UserStatus.APPROVED);
user.setApprovedAt(Instant.now());

// Better — intent is clear, invariants can be enforced in one place
user.approve(clock.instant());
```

**Test first — write the characterization test before introducing the method:**

```java
@Test
void approve_sets_status_and_approvedAt() {
    var user = new User(1L, "Alice", UserStatus.PENDING, null);
    var instant = Instant.parse("2024-06-01T12:00:00Z");

    // Test against current (setter-based) behavior first:
    user.setStatus(UserStatus.APPROVED);
    user.setApprovedAt(instant);

    assertThat(user.getStatus()).isEqualTo(UserStatus.APPROVED);
    assertThat(user.getApprovedAt()).isEqualTo(instant);
}
```

Then introduce the method and update the call site in the test:

```java
public void approve(Instant approvedAt) {
    if (this.status == UserStatus.BLOCKED) {
        throw new IllegalStateException("Blocked user cannot be approved");
    }
    this.status = UserStatus.APPROVED;
    this.approvedAt = approvedAt;
}

// Updated test call — same assertions:
user.approve(instant);
assertThat(user.getStatus()).isEqualTo(UserStatus.APPROVED);
assertThat(user.getApprovedAt()).isEqualTo(instant);
```

If full immutability isn't yet feasible, this pattern is the right intermediate step. The entity still mutates, but mutation is controlled and named.

**Migration order:**
1. Identify clusters of setters that are always called together.
2. Write a test covering that cluster's effect on the object's state.
3. Replace each cluster with one intention-revealing method.
4. Only later, if it makes sense, convert the method to return a new object.

---

### 5. Avoid Null — Use Optional and Empty Collections

Never return `null` from a method. Use `Optional<T>` for values that may be absent. Use empty collections instead of null collections.

```java
// Before
public User findByEmail(String email) {
    return userMap.get(email); // may return null
}

// Test first — characterize null-returning behavior:
@Test
void findByEmail_returns_null_when_not_found() {
    assertThat(service.findByEmail("missing@example.com")).isNull();
}

// After
public Optional<User> findByEmail(String email) {
    return Optional.ofNullable(userMap.get(email));
}

// Updated test — same semantic, new type:
@Test
void findByEmail_returns_empty_when_not_found() {
    assertThat(service.findByEmail("missing@example.com")).isEmpty();
}
```

**Do not use `Optional` as a field type or method parameter.** It is a return type only.

```java
// Wrong
private Optional<String> nickname; // never do this

// Right
private String nickname; // can be null internally
public Optional<String> getNickname() {
    return Optional.ofNullable(nickname);
}
```

For collections, always return empty:
```java
public List<Order> getOrders() {
    return orders != null ? orders : List.of();
}
```

---

### 6. Streams: Only for Transformations

Use streams to **transform, filter, and collect** data — not for iteration with side effects.

```java
// Wrong — stream with side effects
users.stream().forEach(user -> {
    user.setActive(true);       // mutation
    userRepository.save(user);  // I/O
});

// Right — classic loop for side-effectful iteration
for (User user : users) {
    User activated = user.activate();
    userRepository.save(activated);
}

// Right — stream for transformation
List<String> activeNames = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .toList();
```

**Quick test:** if the stream body has a `void` lambda or calls a repository, it should be a loop instead.

---

## JPA / Hibernate Compatibility

These constraints apply when the class under refactoring is a JPA `@Entity`.

### The No-Arg Constructor Problem

JPA requires a **no-arg constructor** (at least `protected`) to instantiate proxy classes. Keep it, but make it `protected` so application code never uses it accidentally.

```java
@Entity
public class User {
    @Id
    private Long id;
    private String name;
    private UserStatus status;
    private Instant approvedAt;

    protected User() {}  // Required by JPA — do not use in application code

    public User(Long id, String name, UserStatus status, Instant approvedAt) {
        this.id = id;
        this.name = name;
        this.status = status;
        this.approvedAt = approvedAt;
    }
}
```

### `final` Fields Are Incompatible with JPA

Hibernate assigns fields via reflection and cannot handle `final`. For JPA entities, enforce immutability through the API instead: make setters `private`, expose only intention-revealing public methods.

```java
@Entity
public class User {
    @Id
    private Long id;        // not final — JPA needs to write it
    private String name;
    private UserStatus status;

    protected User() {}

    public User(Long id, String name) {
        this.id = id;
        this.name = name;
        this.status = UserStatus.PENDING;
    }

    public void approve(Instant approvedAt) {
        if (this.status == UserStatus.BLOCKED) {
            throw new IllegalStateException("Blocked user cannot be approved");
        }
        this.status = UserStatus.APPROVED;
        this.approvedAt = approvedAt;
    }

    // No public setters — mutation only via named methods
}
```

### Collections in JPA Entities

`List.copyOf()` at construction time **breaks Hibernate lazy loading** because it replaces the proxy object. Use an unmodifiable *view* in the getter instead.

```java
// Safe: returns an unmodifiable view, preserves the Hibernate proxy
public List<OrderItem> getItems() {
    return Collections.unmodifiableList(items);
}

// Unsafe for JPA — breaks lazy loading:
public Order(String id, List<OrderItem> items) {
    this.items = List.copyOf(items); // do not do this for @OneToMany collections
}
```

Use `List.copyOf()` only for plain value objects that are not JPA-managed.

### `Optional` and JPA

`Optional` as a field or `@Column` type is not supported by JPA. Use it only as a getter return type.

```java
// Wrong
@Column
private Optional<String> nickname;

// Right
@Column
private String nickname;

public Optional<String> getNickname() {
    return Optional.ofNullable(nickname);
}
```

### JPA Compatibility Summary

| Principle              | Safe for JPA? | Notes                                          |
|------------------------|:-------------:|------------------------------------------------|
| `final` fields         | ❌            | Use private mutation + named methods instead   |
| No no-arg constructor  | ❌            | Keep `protected` no-arg for Hibernate          |
| `Optional` return type | ✅            | Only as return type, never as field/param      |
| Intention-revealing methods | ✅       | Fully compatible                               |
| `List.copyOf()` in collections | ⚠️  | Only for non-JPA value objects                 |
| `Collections.unmodifiableList()` in getter | ✅ | Safe — wraps the proxy            |
| Side effects at the edges | ✅         | No JPA-specific constraint                     |

---

## Migration Strategy

When asked to refactor existing code, follow this order:

1. **Clarify scope (Step 0).** Ask the user which principles to apply. Do not proceed until you have a clear answer. Only apply what was selected.

2. **Read the code.** Identify which classes have many setters, where `null` is returned, where `Instant.now()` or repositories are called inside domain methods. Focus only on areas relevant to the selected principles.

3. **Write tests before touching anything (Step 1).** Cover the current behavior. Run them. They must be green.

4. **Classify changes by impact:**
   - Low risk: add `Optional` return types, make fields `final`, defensive copies.
   - Medium risk: introduce intention-revealing methods (replaces setter clusters).
   - Higher risk: make entities fully immutable (return new objects from state-change methods).

5. **Refactor one class or one method at a time.** Run tests after each change.

6. **The refactor is done when all tests are green.** Not before.

7. **Avoid over-engineering.** Don't introduce `Either`, `Try`, `Vavr`, or monad wrappers unless the user explicitly asks. Keep it idiomatic Java.

---

## Common Patterns Quick Reference

| Problem                          | Solution                                          |
|----------------------------------|---------------------------------------------------|
| Too many setters                 | Intention-revealing methods (`user.approve(t)`)   |
| Mutable fields                   | `final` fields + constructor (not for JPA)        |
| `null` returns                   | `Optional<T>` return type                         |
| `null` collections               | Return `List.of()` / `Collections.emptyList()`    |
| `Instant.now()` in domain        | Pass `Clock` or `Instant` as parameter            |
| Repository in domain             | Move to service/use-case layer                    |
| Stream with side effects         | Replace with `for` loop                           |
| Copy-and-modify                  | `with*` methods or builder                        |
| No tests before refactor         | Write characterization tests first                |

---

## What NOT to Do

- ❌ Don't refactor without tests — write them first, verify they're green, then change code.
- ❌ Don't add Vavr, Lombok `@Value`, or other libraries unless asked.
- ❌ Don't convert everything to records if the class has complex behavior — records are for pure data holders.
- ❌ Don't use `Optional.get()` without checking — always use `map`, `orElse`, or `orElseThrow`.
- ❌ Don't stream over a list just to call `forEach` with side effects.
- ❌ Don't remove all setters in one PR — migrate gradually with intention-revealing methods first.
- ❌ Don't make domain objects depend on infrastructure types (`EntityManager`, `ResultSet`, etc.).
- ❌ Don't use `final` fields on JPA `@Entity` classes — Hibernate can't handle them.
- ❌ Don't call `List.copyOf()` on `@OneToMany` collections — it breaks Hibernate lazy loading.

---

## Output Format

When the user shares code to refactor:

1. **Ask which principles to apply (Step 0)**, unless they've already been explicit. Wait for confirmation before proceeding.
2. **Write (or show) the tests first (Step 1)**, covering the current behavior. If tests already exist, note that they must pass before and after.
3. **Show the refactored code** with inline comments where the change is non-obvious. Only apply the selected principles.
4. **Confirm that all tests pass** after the refactor, or note what to verify if the environment isn't available.
5. **Note any trade-offs** (e.g. "this entity uses JPA — `final` fields are skipped; immutability is enforced via private setters").
6. Keep responses focused. Don't refactor code the user didn't show you.

# `java(fn)` Organization — Introductory & Tutorial Guide

Welcome to `java(fn)`, a collection of small, dependency-free Java libraries that make functional programming in Java practical, readable, and safe.

This guide is written two ways at once:

* **As a first-time read:** start at the top to understand *why* this organization exists, then scroll down to see *how* each project and tool could improve your code.
* **As a reference:** jump straight to the project and tool you are using via the headers below. Each tool's section contains idiomatic usage, code examples, and `TODO` notes where the maintainers are still settling the API.

---

## Why `java(fn)` exists

Java’s standard library and mainstream frameworks push you toward exceptions for control flow, mutable state for data, and deeply nested lambdas for streams. The result is code that is hard to reason about: you cannot tell by looking at a method signature whether it fails, what it might throw, or what parts of a computation succeeded when several operations run together.

The `java(fn)` libraries answer that with a small set of tools:

* **`Result`** — a value that is either a success (`Ok`) or a failure (`Err`), never both. Methods return `Result` instead of throwing, so failures are visible in the type system.
* **`Try`** — a bridge from exception-throwing code (including streams, where exceptions are not allowed) into `Result`s.
* **`Either`** — a semantics-neutral sum type for "left or right" when both alternatives are equally valid.
* **`ResultBatch`** — collect many `Result`s into one container that holds both the errors and the successes, so you can report *all* validation failures at once.
* **Tuples** (`Pair`, `Trio`, `Quad`, `Index`) — immutable, named data carriers that keep stream pipelines readable without creating one-off classes.
* **Small utils** (`ToDo`, `Expect`, `RestartingThread`, `Data`, `Haltable`, `Iter`) — the glue that keeps the rest of your codebase tidy.

All modules target Java 17, have no third-party runtime dependencies, and are published separately so you pull in only what you need:

| Project | Artifact | Current version | What it adds |
|---------|----------|-----------------|--------------|
| `javafn` (core) | `org.javafn:javafn` | 2.4.0 | `Result`, `Try`, `Either`, `ResultBatch`, primitive-specialized results |
| `javafn-tuples` | `org.javafn:javafn-tuples` | 1.1.5 | `Pair`, `Trio`, `Quad`, `Index`, primitive-specialized variants, array ops |
| `javafn-utils` | `org.javafn:javafn-utils` | 0.2.0 | `ToDo`, `Expect`, `RestartingThread`, `Data`, `Haltable`, `Iter` |

---

## Using the libraries

**Maven**

```xml
<dependency>
    <groupId>org.javafn</groupId>
    <artifactId>javafn</artifactId>
    <version>2.4.0</version>
</dependency>
<dependency>
    <groupId>org.javafn</groupId>
    <artifactId>javafn-tuples</artifactId>
    <version>1.1.5</version>
</dependency>
<dependency>
    <groupId>org.javafn</groupId>
    <artifactId>javafn-utils</artifactId>
    <version>0.2.0</version>
</dependency>
```

**Gradle**

```gradle
implementation 'org.javafn:javafn:2.4.0'
implementation 'org.javafn:javafn-tuples:1.1.5'
implementation 'org.javafn:javafn-utils:0.2.0'
```

Compatibility: all modules are bytecode-compatible with Java 17. The core APIs are stable within the 1.x / 2.x series, but the project is still settling some naming (for example, Scala-flavored `pair._1` is not idiomatic Java), so expect evolution under semantic versioning.

**Contact:** email `org.javafn`at`javafn.org`, or open an issue in the relevant repository.

---

## Project: `javafn` (core)

### Why

This is the heart of the organization and the dependency to take if you only take one. It changes how you handle failure in Java: instead of methods that throw and callers that guess what to catch, methods return `Result<OK>`, and the caller decides how to handle the `Err` case explicitly.

The project also supplies `Try` (to wrap existing exception-throwing code), `Either` (for sum types without error semantics), `ResultBatch` (to aggregate many results), primitive-specialized results (to avoid boxing), and `AnyError` (the error value carried inside every `Result`).

---

### Tool: `Result<OK>`

**How it improves your code.** A `Result` is a sealed interface with two records: `Result.Ok<OK>` (a success carrying a value) and `Result.Err<OK>` (a failure carrying an `AnyError`). Because the failure case is part of the return type, callers must handle it — there are no surprise exceptions. You inspect, transform, chain, and recover from results without a `catch` block in sight.

**Construction**

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#result_creation -->
```java
final Result<Foo> ok = Result.ok(new Foo());
final Result<Foo> err = Result.err("something went wrong");
final Result<Foo> errEx = Result.err(new IllegalStateException("boom"));
final Result<Foo> errEnum = Result.err(DomainError.INVALID_INPUT);

// AnyError factories
final AnyError a = AnyError.from("simple message");
final AnyError b = AnyError.from("code %d not found", 404);
final AnyError c = AnyError.from(new NumberFormatException("not a number"));
final AnyError joined = a.join(b);        // ErrorList, messages joined with "\n"
final AnyError alsoJoined = AnyError.joined(a, b, c);

// AnyError.fail is shorthand for Result.err(AnyError.from(...))
final Result<Foo> failed = AnyError.fail("validation failed");
final Result<Foo> other = AnyError.from("another error").fail();
```

`AnyError` is an interface.  Your code can choose to implement it for a domain specific error.  Most errors are simply reported (e.g., logging or network responses for bad requests), so the primary method on all `AnyError`s is `.message()`.

**Inspecting a result.** Prefer pattern matching over the `expect*` getters:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#variant_inspect -->
```java
final Result<Foo> res = doWork();
if (res instanceof Result.Ok<Foo> asOk) {
    final Foo foo = asOk.value();
    foo.doFooThings();
} else if (res instanceof Result.Err<Foo> asErr) {
    final AnyError err = asErr.value();
    System.err.println("Oh no! " + err.message());
}
```

The `expectOk()` / `expectErr()` methods are "unsafe" — they throw `IllegalStateException` if the result is the wrong variant. Use them only after checking the variant:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#improper_expect_ok -->
```java
final Result<Foo> isErr = Result.err("DON'T DO THIS");
final Foo foo = isErr.expectOk();
```

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#improper_expect_err -->
```java
final Result<Foo> isOk = Result.ok(new Foo("DON'T DO THIS"));
final AnyError foo = isOk.expectErr();
```

`opt()` / `optErr()` give you `Optional` views.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#optional_view -->
```java
final Result<Foo> resFoo = doWork();
final Foo foo = resFoo.opt().orElseGet(() -> new Foo(
        "Create default value if doWork produced an error"));
final Result<Bar> resBar = doWork(foo);
resBar.optErr().ifPresent(err ->
        System.err.println("An error was returned: " + err.message()));
```

**Transforming results**

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#mapping_results -->
```java
final Result<Foo> resFoo1 = doWork();
final Result<Foo> resFoo2 = resFoo1.map(Foo::updateState);
final Result<Bar> resBar = resFoo2.flatMap(foo -> doWork(foo));
resBar.ifErr(theErr -> System.err.println(theErr.message()));
final Result<Bar> maybeRecovered = resBar.flatRecover(err -> {
    if (err instanceof AnyError.EnumError<?> domainErr &&
            domainErr.error().equals(DomainError.FOO_NOT_FOUND)) {
        // try to recover
        return doWork(new Foo("Try again with good Foo"));
    } else {
        // Leave it as an err
        return Result.err(err);
    }
});
```

**The escape hatch (no `?` operator).** Java has no Rust-style `?` operator. The recommended idiom for returning an error immediately from an intermediate computation is a scope block:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#escape_hatch -->
```java
public static Result<Bar> demoEscapeHatch() {
    final Foo foo; {
        final Result<Foo> maybeFoo = doWork();
        if (maybeFoo instanceof Result.Err<Foo> asErr) return asErr.into();
        foo = maybeFoo.expectOk();
    }
    // Use foo here.  maybeFoo is not in scope.
    foo.doFooThings();
    foo.updateState();
    final Result<Bar> bar = doWork(foo);
    return bar;
}
```

Java 17 sealed types are not yet exhaustive, so the compiler may still complain that `intermediate` could be unassigned without an else branch.  Java 21 can determine definite assignment for this case, so the pattern going forward should be a pattern match (or switch) on both.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#pattern_match -->
```java
public LongResult patternMatch() {
    final int intermediate; {
        final IntResult res = getCount();
        if (res instanceof IntResult.Err asErr) {
            return asErr.intoLong();
        } else if (res instanceof IntResult.Ok asOk) {
            intermediate = asOk.value();
        } else {
            throw new IllegalStateException("""
                    Impossible to reach here,
                    but Java 17 does not recognize it.
                    This else block is no longer needed in Java 21.
                    """);
        }
    }

    return LongResult.ok(intermediate + 1);
}
```
<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#switch_expression -->
```java
public LongResult switchExpression() {
    final int intermediate;
    switch (getCount()) {
        case IntResult.Err asErr -> { return asErr.intoLong(); }
        case IntResult.Ok asOk -> intermediate = asOk.value();
        default -> throw Expect
                .exhaustiveMatchOnSealedType(IntResult.class);
    }
    // For Java 21+, this will be the preferred escape hatch syntax.
    // default case necessary until Java21+
    // NOTICE: no unsafe "expect" patterns!

    return LongResult.ok(intermediate + 1);
}
```

`Err.into()` (and `intoInt()`/`intoLong()`/`intoDouble()`/`intoVoid()`) lets you change the `OK` type of an error without touching it — that is how you "return the error" when the function's declared `Result` type differs from the intermediate result's type.  You must be operating on an Err variant, for example, following an instanceof pattern match.  Otherwise, changing the `OK` type would require a map.

**Side effects**

`ifOk` / `ifErr` run a consumer on the wrapped value and return the result unmodified, so they slot into a fluent chain:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#side_effects_ifok_iferr -->
```java
final Result<Foo> res = Result.ok(new Foo());

res.ifOk(v -> System.out.println("Ok: " + v))
   .ifErr(e -> System.err.println("Err: " + e.message()))
   .map(Foo::updateState);
```

`consume(errFn, okFn)` and `reduce(errFn, okFn)` run *exactly one* of two functions (and `reduce` returns a common value):

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#consume_and_reduce -->
```java
doWork().consume(
        err -> System.err.println(err.message()),
        Foo::doFooThings);

final Bar bar = doWork().reduce(
        err -> {
            System.err.println(err.message());
            return new Bar();
        },
        foo -> foo.safeBarGenerate("Both lambdas produce a Bar")
);
```

**Predicates**

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#predicates -->
```java
final IntResult r = IntResult.ok(10);

// || semantics: true if not an ok, else predicate result
final boolean okIfPositive = r.filter(i -> i > 0);

// && semantics: false if not an ok, else predicate result
final boolean okAndPositive = r.isOkAnd(i -> i > 0);

// error-side equivalents
final IntResult err = IntResult.err("bad");
final boolean errMatches = err.filterErr(
        e -> e.message().contains("bad"));
final boolean errAndMatches = err.isErrAnd(
        e -> e.message().contains("bad"));
```

**Recovery and failure**

`recoverIf` turns an `Err` into an `Ok` when a predicate matches:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#recover_if -->
```java
final IntResult recovered = IntResult.err("parse failed")
        .recoverIf(
            e -> e.message().contains("parse"),
            e -> 0);
```

`failIf` / `failIfMsg` / `failIfEx` do the reverse: turn an `Ok` into an `Err` when a predicate matches:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#fail_if -->
```java
final Result<Foo> checked = Result.ok(new Foo("hello"))
        .failIf(
            not(Foo::isStateValid),
            foo -> AnyError.from("validation failed"));
```

**Collecting many results — `Result.batch()`**

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#result_batch -->
```java
final List<Result<Foo>> results = List.of(
        Result.ok(new Foo("A")),
        Result.err("boom"),
        Result.ok(new Foo("B")),
        Result.err("bang")
);

final ResultBatch<Foo> batch = results.stream().collect(Result.batch());
// batch.errors() ==> ["boom", "bang"]
// batch.oks() ==> [fooA, fooB]

final Result<List<Foo>> folded = batch.fold();
// folded -> Err(joined errors) because there were errors
```

**Stream companions — `Result.Errs` and `Result.Oks`**

These return functions so you avoid writing `res -> res.` repeatedly:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#result_errs_oks -->
```java
final Result<Foo> result1 = Result.ok(new Foo());
final Result<Foo> result2 = Result.err("failed");

final List<Foo> validFoos = Stream.of(result1, result2)
        .map(Result.Errs.map(e ->
                e.withContext("During example")))
        .peek(Result.Errs.ifErr(e -> {
            if (e.message().contains("fatal")) {
                System.err.println(e);
            } else {
                System.out.println(e);
            }
        }))
        .map(Oks.map(Foo::updateState))
        .filter(Oks.isOkAnd(Foo::isStateValid))
        .map(Result::expectOk)
        .toList();
```

**TODO** — `flatRecover` vs `recoverIf`: the idiomatic distinction between the two isn't fully settled yet. Write a comparison example (recover a known error vs. transform an error into a new ok value).

---

### Tool: `Try`

**How it improves your code.** `Try` converts exception-throwing code into `Result`s. It is the on-ramp for legacy or JDK APIs that throw, and it is essential inside streams, where Java does not permit checked exceptions in lambdas. `Try` catches `Exception` (not `Throwable`, which usually signals something fatal) and wraps it in an `AnyError.ExceptionError`.

**Usage**

Note the following examples use number parsing.  

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#try_usage -->
```java
// Single value
final Result<Integer> parsed = Try.toGet(
        () -> Integer.parseInt("42"));   // Ok(42)
final Result<Integer> bad = Try.toGet(
        () -> Integer.parseInt("zzz"));  // Err

// Primitive supplier
final IntResult parsedInt = Try.toGetInt(() -> Integer.parseInt("42"));

// Runnable that might throw -> VoidResult
final VoidResult flushed = Try.to(
        () -> System.out.println("flushed"));

// In streams
final List<String> numbers = List.of("1", "2", "zzz", "4");

final List<IntResult> results = numbers.stream()
    .map(Try.toMapToInt(Integer::parseInt))
    .toList();
```

`toUse(ThrowingConsumer)` is the stream-friendly form of `to(ThrowingRunnable)`; it returns a function rather than executing immediately:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#try_to_use -->
```java
final long failedCloses =
        getClientFoos().stream()
        .map(Foo::open)
        .map(Try.toUse(Closeable::close))
        .filter(VoidResult::isErr)
        .count();
```

**TODO** — `Try` cannot let you select which exception types to catch. Document the recommended workaround (catch `Exception`, then inspect `except()` / pattern-match on the wrapped exception). Show a worked example of filtering `Try` results by exception type.

**TODO** — verify exact naming: `toGet`, `toMap`, `toUse`, `to`, and the primitive variants. Cross-check against the production codebase.

---

### Tool: `Results`

Methods in this class are alternatives to exception throwing variants.  Unlike the Try class, which _catches_ thrown errors, these methods have been rewritten to _not_ throw and natively return an appropariate Result.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#results_helpers -->
```java
final Result<Foo> nonNull = Results.requireNonNull(new Foo());

// Ok(42)
final IntResult parsedI = Results.parseInt("42");
final LongResult parsedL = Results.parseLong("42");
final Result<Integer> decodedI = Results.decodeInt("0x2A");
final Result<Long> decodedL = Results.decodeLong("0x2A");

// Err
final IntResult badI = Results.parseInt("zzz");
final LongResult badL = Results.parseLong("zzz");
final Result<Integer> badDI = Results.decodeInt("zzz");
final Result<Long> badDL = Results.decodeLong("zzz");
```

---

### Tool: `Either<L, R>`

**How it improves your code.** `Either` is a semantics-neutral sum type — "left or right, both equally valid." Use it when neither side is an error (for example, a distance expressed in kilometers or inches). It is nearly a mirror of `Result`: `Left`/`Right` instead of `Err`/`Ok`, with `mapLeft`, `mapRight`, `reduce`, `swap`, and `Lefts`/`Rights` stream companions.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#either_usage -->
```java
// An export job runs either at a specific instant or on a recurring weekday.
// Both alternatives are equally valid — this is a sum type, not error handling.
final Either<Instant, DayOfWeek> schedule = Either.right(DayOfWeek.WEDNESDAY);

// Compute the next run time regardless of which alternative we have.
final Instant now = Instant.now();
final Instant nextRun = schedule.reduce(
    instant -> instant.isAfter(now) ? instant : now.plusSeconds(86_400),
    day -> LocalDate.now()
        .with(TemporalAdjusters.nextOrSame(day))
        .atStartOfDay()
        .toInstant(java.time.ZoneOffset.UTC)
);

// Pattern match to describe the schedule
final String description;
if (schedule instanceof Either.Left<Instant, DayOfWeek> left) {
    final Instant instant = left.value();
    description = "One-time export at " + instant;
} else if (schedule instanceof Either.Right<Instant, DayOfWeek> right) {
    final DayOfWeek dayOfWeek = right.value();
    description = "Recurring export on " + dayOfWeek;
} else {
    throw new IllegalStateException("Not possible, but required until Java21+");
}
System.out.println(description);

// Same thing with a reduce
final String sameDescription = schedule.reduce(
        instant -> "One-time export at " + instant,
        dayOfWeek -> "Recurring export on " + dayOfWeek);
System.out.println(sameDescription);

// Side effect: log the schedule details, returning the Either unmodified
schedule.use(
    instant -> System.out.println("Scheduled at: " + instant),
    day -> System.out.println("Recurring on: " + day)
);

```

`Either` is not heavily documented; the `Result` documentation applies almost verbatim (Left ≈ Err, Right ≈ Ok).

**TODO** — confirm any behavioral differences between `Either` and `Result` beyond naming (e.g. `of(...)`, `toRightIf`/`toLeftIf`, `swap`).

---

### Tool: `ResultBatch<OK>`

**How it improves your code.** `ResultBatch` is a product type (it holds both errors and successes at once), unlike the sum-type `Result`. It is produced by `Result.batch()` (or the primitive equivalents) and lets you "validate everything and report every failure" instead of stopping at the first error.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#result_batch_from_parts -->
```java
// Validate multiple path parts, collecting every error before deciding success.
final String path = "/home/user/Documents/report.txt";
final List<String> parts = List.of(path.split(
        Pattern.quote(File.separator)));

final Result<Path> validated = parts.stream()
    .map(Result::ok)
    .map(Oks.failIf(
            String::isBlank,
            err -> AnyError.from("path part must not be blank")))
    .map(Oks.failIf(
        part -> part.contains(".."),
        err -> AnyError.from("path part cannot contain '..'")))
    // ...
    .collect(Result.batch())
    .reduce((errs, oks) -> errs.isEmpty()
        ? Result.ok(Path.of(path))
        : AnyError.joined(errs).fail());
```

Group tasks in a transaction and only commit if no errors occurred.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#result_batch_job_pattern -->
```java
// Collect a batch of file task results, then either report all errors or
// keep the successful ones.
List<Result<Foo>> tasks = List.of(
    Result.ok(new Foo("task-1")),
    Result.err("file not found"),
    Result.ok(new Foo("task-2")),
    Result.err("permission denied")
);

final ResultBatch<Foo> batch = tasks.stream().collect(Result.batch());

if (batch.hasErrors()) {
    // Rollback: report all errors, drop the successes
    final Result<List<Foo>> aborted = AnyError
            .joined(batch.errors()).fail();
} else {
    // Commit: all succeeded, use the oks
    final Result<List<Foo>> committed = Result.ok(batch.oks());
}

// Partial success: keep both successes and errors separately
batch.oks().forEach(Foo::doFooThings);
batch.errors().forEach(err ->
        System.err.println("Error: " + err.message()));
```

Calling `reduce` on a batch allows you to operate on both successes and failures.  Calling `fold` will turn the batch into a result, and reduce will take two separate lambdas, one for success and one for failure.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#result_batch_http_response -->
```java
final HttpResponse response = getClientFoos().stream()
        .map(Foo::updateState)
        .map(foo -> foo.isStateValid()
                ? Result.ok(foo)
                : Result.<Foo>err(foo.name() + " is invalid"))
        .collect(Result.batch())
        .fold()
        .reduce(errs -> new HttpResponse().code(400).entity(
                "The following errors were encountered fulfilling "
                        + "your request.  Please fix and try again: "
                        + errs.message()),
                oks -> new HttpResponse()
                        .code(200)
                        .entity(oks));
```


`ResultBatch` also offers `mapErrs`, `mapOks`, `append`, and `size()`.

---

### Tool: Specialized results — `IntResult`, `LongResult`, `DoubleResult`, `VoidResult`

**How it improves your code.** The primary class is `Result<OK>` (documented above). The specialized variants are primitive-specialized copies that avoid boxing when your value is `int`, `long`, `double`, or `void`:

| Primary | Specialized | OK type | `expectOk()` returns | `opt()` returns |
|---------|-------------|---------|----------------------|-----------------|
| `Result<OK>` | `IntResult` | `int` | `int` | `OptionalInt` |
| `Result<OK>` | `LongResult` | `long` | `long` | `OptionalLong` |
| `Result<OK>` | `DoubleResult` | `double` | `double` | `OptionalDouble` |
| `Result<OK>` | `VoidResult` | (none) | — | — |

Each has `ok(...)`, `err(...)`, `map(...)`, `mapErr(...)`, `flatMap(...)`, `flatRecover(...)`, `ifOk(...)`, `ifErr(...)`, `reduce(...)`, `consume(...)`, plus `boxed()` (back to `Result<boxed>`) and `mapToObj(...)` (to an object `Result`). `IntResult` also has `mapToLong` / `mapToDouble` / `mapToInt`; `LongResult`/`DoubleResult` have the analogous conversions.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#int_result -->
```java
final IntResult r = IntResult.ok(7);

// primitive-aware mapping (no boxing)
final IntResult doubled = r.map(i -> i * 2);

// convert back to object Result
final Result<Integer> boxed = r.boxed();
final Result<String> asObj = r.mapToObj(i -> "num=" + i);

// Err conversions preserve the error
final IntResult.Err err = IntResult.err("bad");
final LongResult longErr = err.intoLong();
```

`VoidResult` has no value on success:

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#void_result -->
```java
final VoidResult vr = VoidResult.ok();
vr.ifOk(() -> System.out.println("done"));
vr.ifErr(e -> System.err.println(e.message()));

// failIf on VoidResult uses BooleanSupplier / Supplier<AnyError>
final VoidResult checked = VoidResult.ok()
    .failIf(() -> false, () -> AnyError.from("condition failed"));
```

Each specialized result has its own `Errs`/`Oks` companions (`IntResult.Errs`, `IntResult.Oks`, …), typed to the primitive functional interfaces (`IntPredicate`, `IntConsumer`, `IntUnaryOperator`, …).

**TODO** — document the `mapToInt`/`mapToLong`/`mapToDouble` conversion matrix fully.

---

### Tool: `AnyError`

**How it improves your code.** Every `Result` carries an `AnyError`, never a raw exception. This means the error value is inspectable and composable without reaching into stack traces. Implementations:

* `StringError` — a plain message (supports `String.format`-style args).
* `ExceptionError<E>` — wraps a caught `Exception`.
* `EnumError<E>` — wraps an enum constant.
* `ErrorList` — joins multiple `AnyError`s, messages separated by `\n`.

<!-- snips: ../../javafn/main/src/test/java/org/javafn/examples/ExamplesMain.java#any_error -->
```java
// From a string
final AnyError a = AnyError.from("something went wrong");
final AnyError b = AnyError.from("code %d not found", 404);

// From an exception
final AnyError c = AnyError.from(new NumberFormatException("not a number"));

// Join several errors into one
final AnyError combined = a.join(b).join(c);
final String msg = combined.message();

// Turn any AnyError into an Err Result
final Result<String> fail = a.fail();
```

`except()` is the bridge back to throwable code — useful with `Try`, where the error is guaranteed to be an `ExceptionError` but that cannot be expressed in the type system.

**TODO** — confirm whether `AnyError.except()` should only be trusted after a `Try` operation, and document the contract.

---

## Project: `javafn-tuples`

### Why

Tuples were extracted from core `javafn` because they solve a different problem: readable, immutable data carriers for stream pipelines. Instead of creating a one-off record or a parallel array every time you need to pass two or three related values through a stream, you use a `Pair`, `Trio`, or `Quad`. They are code-generated, so primitive-specialized variants exist for every common combination of `int`/`long`/`double`/object elements.

---

### Tool: `Pair`, `Trio`, `Quad`

**How it improves your code.** `Pair<V1, V2>` is the primary documented tuple; `Trio` and `Quad` are the same shape with three and four elements. Records are immutable, and every "mutation" returns a new instance.

```java
import org.javafn.tuples.Pair;
import org.javafn.tuples.Pairs;

Pair<Integer, String> pair = Pair.of(42, "answer");

Integer first  = pair.v1();   // 42
String  second = pair.v2();   // "answer"

// "Setters" return a new pair
Pair<Integer, String> updated = pair.v1(43);   // Pair.of(43, "answer")

// Predicates
boolean both    = pair.matches((i, s) -> i == 42 && s.equals("answer"));
boolean firstOk = pair.matches1(i -> i == 42);
boolean secondOk= pair.matches2(s -> s.equals("answer"));

// Side effects (peek returns this; consume returns void)
pair.peek((i, s) -> System.out.println(i + " = " + s));
pair.peek1(i -> System.out.println("first: " + i));
pair.peek2(s -> System.out.println("second: " + s));
pair.consume((i, s) -> { /* ... */ });

// Mapping
String combined  = pair.map((i, s) -> i + " " + s);
Pair<Integer, String> mapped1 = pair.map1(i -> i * 2, (i, s) -> s);
Pair<Integer, String> mapped2 = pair.map2(s -> s + "!", (i, s) -> i);

// Promote to Trio
Trio<Integer, String, Boolean> trio = pair.toTrio(true);
```

**Companion `Pairs`** — static helpers that return functions, for streams:

```java
stream.of(pair1, pair2)
    .map(Pairs.map((i, s) -> i + s.length()))
    .filter(Pairs.matches1(i -> i > 0))
    .peek(Pairs.peek((i, s) -> System.out.println(i)))
    .forEach(System.out::println);
```

`Trio`/`Quad` and their companions `Trios`/`Quads` add `matches1`/`matches2`/`matches3`, `peek1`/`peek2`/`peek3`, `map1`/`map2`/`map3`, etc.

**TODO** — the docs say `Trio` and `Quad` "are not documented so we don't run the risk of them getting out of sync." Decide whether to expand their docs or keep the `Pair`-only policy.

---

### Tool: Primitive-specialized tuples

**How it improves your code.** Because tuples are code-generated, there are variants for every combination of `int`/`long`/`double`/object elements: `IntPair`, `LongPair`, `DoublePair`, `ObjIntPair`, `IntObjPair`, `ObjDoublePair`, … These behave exactly like `Pair`; only the element types and the functional interfaces they accept differ (e.g. `IntPair` uses `IntPredicate`, `IntUnaryOperator`, etc.).

```java
import org.javafn.tuples.IntPair;
import org.javafn.tuples.Pairs;

IntPair ip = IntPair.of(1, 2);
int sum    = ip.map((a, b) -> a + b);              // 3
IntPair inc= ip.map1(i -> i + 1);                  // IntPair(2, 2)
```

**TODO** — confirm the complete list of generated primitive tuple types and their package (`org.javafn.tuples`).

---

### Tool: `Index<V2>`

**How it improves your code.** `Index<V2>` is a `(int, V2)` pair — Rust's `enumerate` / Python's `enumerate`. The first element is an increasing counter starting at 0, so you can tag stream elements with their position without managing a mutable index variable.

```java
import org.javafn.tuples.Index;
import org.javafn.tuples.Pairs;

// Inline: transform each element into an Index
Stream<Index<UUID>> indexed =
    Stream.generate(UUID::randomUUID).limit(100)
        .map(Pairs.index());   // Function<T, Index<T>>

// Wrapper: index an entire stream
Stream<Index<UUID>> indexed2 =
    Pairs.index(Stream.generate(UUID::randomUUID).limit(100));

int i     = index.i();         // the counter
UUID value= index.v2();        // the element
```

`Index.over(stream)` and `Index.each()` return a function that tags elements with an incrementing counter. `Index.eachAs(fn)` combines indexing with mapping the counter to another type.

```java
someStream
    .map(Pairs.index())                         // Index
    .peek(Indexed.peek((i, item) ->
        System.out.println("Completing item " + i)));
```

**TODO** — `Index.over()` / `Index.each()` use a mutable `StreamIndexer` counter — not safe for parallel streams. Document the constraint and show the correct usage.

**TODO** — clarify the relationship between `org.javafn.tuples.Index` and the `Indexed` name used elsewhere.

---

### Tool: Array operations — `chunks`, `windows`, `zip`, `partition`

**How it improves your code.** Available on the companions (`Pairs`, `Trios`, `Quads`) for object arrays and primitive streams/arrays, these let you reshape sequences without manual index arithmetic.

```java
import org.javafn.tuples.Pair;
import org.javafn.tuples.Pairs;

// chunks: group an array into non-overlapping pairs
UUID[] uuids = ...;
Stream<Pair<UUID, UUID>> chunks = Pairs.chunks(uuids);            // throws if odd length
Stream<Pair<UUID, UUID>> padded = Pairs.chunks(uuids, null);      // pad the last

// windows: overlapping sliding windows of size 2
Stream<Pair<UUID, UUID>> windows = Pairs.windows(uuids);          // n-1 windows

// zip: pair two streams elementwise
Stream<Pair<Integer, Long>> zipped =
    Pairs.zip(IntStream.range(0, 5), LongStream.range(0, 5));

// partition: split a stream by a predicate
Pair<IntStream, IntStream> split =
    Pairs.partition(IntStream.range(0, 10), i -> i > 5);
// split.v1() -> > 5, split.v2() -> <= 5

// stream: turn a homogeneous tuple back into a stream of its elements
IntStream elements = Pairs.stream(IntPair.of(1, 2));
```

**TODO** — confirm whether `windows` requires at least 2 elements and what it returns for shorter arrays.

---

## Project: `javafn-utils`

### Why

This project holds small utilities that do not fit in core or tuples but are used across a codebase: a searchable placeholder (`ToDo`), programmer-expectation assertions (`Expect`), a self-restarting thread (`RestartingThread`), immutable collection appends (`Data`), a controlled component handle (`Haltable`), and stream conversions (`Iter`).

---

### Tool: `ToDo`

**How it improves your code.** `ToDo` is a compile-time placeholder that throws at runtime. Unlike `// TODO` comments, which accumulate forever, usages of the `ToDo` class are searchable, so you can find unfinished work reliably.

```java
import org.javafn.utils.ToDo;

public List<String> notYetImplemented() {
    return ToDo.TODO("wire up the real data source");   // throws ToDoException
}

class Demo {
    private final ToDo todo = new ToDo();   // construction does not throw
    public String get() { return todo.todo("not implemented"); } // throws
}
```

---

### Tool: `Expect`

**How it improves your code.** `Expect` expresses expectations the compiler cannot guarantee — for example, that an `Optional` is present even though the type system does not know it. The generated exceptions carry the suffix "this is likely a programming error," so they are suitable only for conditions that should be unreachable.

```java
import org.javafn.utils.Expect;

// Throw an IllegalStateException with a helpful message
throw Expect.unmet("the list should have exactly one element here");

// Deferred form
IllegalStateException ex = Expect.that("unreachable branch");

// Guard a condition
Expect.that(() -> index >= 0, "index must be non-negative");

// Optional.orElseThrow when you know it's present
String v = optional.orElseThrow(Expect.optionalPresent());

// Private constructor for static-only classes
private Utils() { Expect.staticConstructorNotCalled(this.getClass()); }
```

---

### Tool: `RestartingThread`

**How it improves your code.** A `Thread` that restarts itself when its uncaught exception handler fires, driven by a `ThreadContext`. Use it when you need a background worker that should survive transient failures.

```java
import org.javafn.utils.RestartingThread;

RestartingThread.ThreadContext ctx = new RestartingThread.ThreadContextImpl(
    "worker", () -> running
) {
    @Override public void run() { /* work */ }
    @Override public void onThreadInstanceChanged(RestartingThread newInstance, Throwable cause) {
        // thread restarted after an exception
    }
    @Override public void onThreadInstanceExited(Throwable cause) {
        // thread exited because isRunning() returned false
    }
};

new RestartingThread(ctx); // starts and self-restarts on uncaught exceptions
```

**TODO** — add a realistic production example (e.g. wiring into `Runtime.addShutdownHook`).

---

### Tool: `Data`

**How it improves your code.** Immutable "append" helpers for `List`, `Set`, `Map`, and arrays. Originals are not modified; results are unmodifiable copies.

```java
import org.javafn.utils.Data;

List<String> copy     = Data.append(list, "extra");
Map<String, Integer> m2 = Data.append(map, "key", 42);
int[] arr2            = Data.append(new int[]{1, 2}, 3);
```

---

### Tool: `Haltable<C>`

**How it improves your code.** A handle that exposes a `component()` and a one-shot `halt()` without putting cleanup in the component's public API. Intended for a single owner (e.g. a shutdown hook).

```java
import org.javafn.utils.Haltable;

Haltable<MyManager> handle = new Haltable<>(manager, manager::shutdown);
MyManager m = handle.component();
handle = handle.halt();   // runs shutdown; component() now returns null
```

`halt()` is one-shot and not threadsafe.

---

### Tool: `Iter`

**How it improves your code.** Turns `Iterable`/`Iterator` into `Stream`s — the boilerplate to do this is painful enough to warrant a dedicated method.

```java
import org.javafn.utils.Iter;

Stream<String> fromIterable = Iter.toStream(iterable);
Stream<String> fromIterator = Iter.toStream(iterator);   // consumes the iterator
Stream<String> parallel     = Iter.toParStream(iterable);
```

---

## Compatibility and contact

All modules target Java 17 and are bytecode-compatible with Java 17. The core `Result` and tuple APIs are stable within the 1.x / 2.x series; the project is still settling some naming (for example, Scala-flavored `pair._1` is not idiomatic Java), so APIs may evolve under semantic versioning.

**Contact:** email `org.javafn`at`javafn.org`, or open an issue in the relevant repository.

---

<!-- 
Source notes: this guide was assembled from the existing project READMEs
(javafn/main/README.md, javafn-tuples/README.md, javafn-utils/README.md),
the javadoc on the source files, and the test/demo code in each project.
Where the idiomatic usage was unclear, a TODO placeholder was left for the
next round.
-->

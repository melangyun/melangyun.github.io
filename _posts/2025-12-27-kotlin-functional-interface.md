---
title: Kotlin Functional Interface - SAM Conversion and Comparison with Java
date: 2025-12-27 10:00:00 +0900
categories: [Dev, Kotlin]
tags: [kotlin, functional-interface, sam, lambda, java]
---

## What is a Functional Interface?

A **Functional Interface** is an interface that has **exactly one abstract method** (SAM: Single Abstract Method).

In Kotlin, you declare it using the `fun` keyword before `interface`:

```kotlin
fun interface Greeting {
    fun greet(name: String): String
}
```

### Why Do We Need It?

The main purpose is to enable **lambda expressions** as implementations.

Without `fun interface`:

```kotlin
interface RequestHandler {
    fun handle(request: Request): Response
}

// Verbose anonymous object required
val handler = object : RequestHandler {
    override fun handle(request: Request): Response {
        return Response(200, "OK")
    }
}
```

With `fun interface`:

```kotlin
fun interface RequestHandler {
    fun handle(request: Request): Response
}

// Clean lambda syntax
val handler = RequestHandler { request ->
    Response(200, "OK")
}
```

### Basic Example

```kotlin
// 1. Define functional interface
fun interface Calculator {
    fun calculate(a: Int, b: Int): Int
}

// 2. Implement with lambda
val add: Calculator = { a, b -> a + b }
val multiply: Calculator = { a, b -> a * b }

// 3. Use
println(add.calculate(3, 5))       // 8
println(multiply.calculate(3, 5)) // 15
```

### Benefits of fun interface

#### 1. Concise Code

Reduces boilerplate by replacing anonymous objects with lambdas.

#### 2. Type Safety

Unlike plain lambdas, functional interfaces have **distinct types**.

```kotlin
fun interface UserValidator {
    fun validate(user: User): Boolean
}

fun interface OrderValidator {
    fun validate(order: Order): Boolean
}

val userValidator: UserValidator = { it.age >= 18 }
val orderValidator: OrderValidator = { it.total > 0 }

fun process(validator: UserValidator) { /* ... */ }

process(userValidator)   // ✅ OK
process(orderValidator)  // ❌ Compile error! Type mismatch
```

#### 3. Clear Intent

The `fun` keyword explicitly declares "this interface is designed for lambda usage."

```kotlin
// Clear intent: designed for lambda
fun interface Callback<T> {
    fun onResult(result: T)
}

// Regular interface: may have multiple methods in the future
interface Repository {
    fun save(data: Data)
}
```

#### 4. Safe Extension Prevention

Adding another abstract method causes a **compile error**.

```kotlin
fun interface Handler {
    fun handle(data: Data)
    // fun validate(data: Data)  // ❌ Compile error! Only one abstract method allowed
}
```

> This prevents breaking changes when the interface is already used with lambdas elsewhere in the codebase.
{: .prompt-tip }

### What it is NOT

Functional interface does **not** add new functionality. These two are functionally identical:

```kotlin
// Anonymous object
val handler = object : RequestHandler {
    override fun handle(request: Request) = Response(200, "OK")
}

// Lambda with fun interface
val handler = RequestHandler { Response(200, "OK") }
```

| Aspect | Difference? |
|--------|-------------|
| Compiled result | Nearly identical |
| Runtime performance | No difference |
| Capabilities | Nothing new |

> Functional interface is essentially **syntactic sugar**. It makes existing patterns more concise and safer, but doesn't enable anything that wasn't possible before.
{: .prompt-info }

---

## SAM Conversion

### What is SAM?

SAM stands for **S**ingle **A**bstract **M**ethod.

### What is SAM Conversion?

SAM conversion is the automatic transformation of a lambda expression into an instance of a functional interface.

```kotlin
fun interface Greeting {
    fun greet(name: String): String
}

// SAM conversion happens here: lambda → Greeting instance
val hello: Greeting = { name -> "Hello, $name!" }

// Internally converted to:
val helloExpanded: Greeting = object : Greeting {
    override fun greet(name: String): String {
        return "Hello, $name!"
    }
}
```

### SAM Conversion in Function Parameters

```kotlin
fun interface Filter<T> {
    fun accept(item: T): Boolean
}

fun <T> List<T>.customFilter(filter: Filter<T>): List<T> {
    return this.filter { filter.accept(it) }
}

// Lambda automatically converted to Filter
val numbers = listOf(1, 2, 3, 4, 5)
val evenNumbers = numbers.customFilter { it % 2 == 0 }  // SAM conversion!
println(evenNumbers)  // [2, 4]
```

### Explicit SAM Conversion

When the type is ambiguous, you can use explicit SAM constructors:

```kotlin
fun interface StringProcessor {
    fun process(s: String): String
}

fun interface StringValidator {
    fun process(s: String): Boolean  // Only return type differs
}

// Explicit SAM constructor
val processor = StringProcessor { it.uppercase() }
val validator = StringValidator { it.isNotEmpty() }
```

---

## Java vs Kotlin Comparison

### Java Functional Interface

```java
// Java - uses @FunctionalInterface annotation
@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);

    // Multiple default methods allowed
    default Comparator<T> reversed() {
        return (o1, o2) -> compare(o2, o1);
    }

    // Static methods allowed
    static <T> Comparator<T> naturalOrder() { ... }
}
```

### Kotlin Functional Interface

```kotlin
// Kotlin - uses 'fun' keyword
fun interface Comparator<T> {
    fun compare(o1: T, o2: T): Int

    // Default method (just a method with implementation in Kotlin)
    fun reversed(): Comparator<T> = Comparator { o1, o2 -> compare(o2, o1) }
}
```

### Key Differences

| Aspect | Java | Kotlin |
|--------|------|--------|
| Declaration | `@FunctionalInterface` (annotation) | `fun interface` (keyword) |
| Annotation Role | Optional (compile-time validation) | Required (`fun` needed for SAM conversion) |
| Java Interface SAM | Automatic | Supported since **Kotlin 1.4** |
| Kotlin Interface SAM | - | `fun` keyword required |

### Using Java Interfaces in Kotlin

```kotlin
// Java-defined functional interfaces work with SAM conversion
// e.g., Runnable, Callable, Comparator

// Runnable
val task = Runnable { println("Task executed") }
executor.submit(task)

// Callable
val callable = Callable { fetchDataFromDatabase() }
val future = executor.submit(callable)

// Comparator
val comparator = Comparator<User> { u1, u2 -> u1.name.compareTo(u2.name) }
users.sortedWith(comparator)
```

### Why Kotlin Requires `fun` Keyword?

For **explicitness** and **safety**.

```kotlin
// Without 'fun' - prevents accidental SAM conversion
interface Repository {
    fun save(data: Data)
    // If you add another method later, existing lambda code breaks
}

// With 'fun' - compiler enforces single abstract method
fun interface Repository {
    fun save(data: Data)
    // fun delete(data: Data)  // Compile error! Only one abstract method allowed
}
```

> The `fun` keyword makes the contract explicit: this interface is designed for lambda usage and must have exactly one abstract method.
{: .prompt-tip }

### What is a Default Method?

Default methods (introduced in Java 8) allow interfaces to have method implementations.

```java
// Java
interface MessageQueue {
    void send(Message message);  // Abstract - must implement

    default void sendAsync(Message message) {  // Default - optional to override
        CompletableFuture.runAsync(() -> send(message));
    }
}
```

In Kotlin, you can simply add method implementations directly:

```kotlin
// Kotlin - no 'default' keyword needed
interface MessageQueue {
    fun send(message: Message)  // Abstract

    fun sendAsync(message: Message) {  // Has implementation - optional to override
        GlobalScope.launch { send(message) }
    }
}
```

> Default methods don't count toward the single abstract method requirement. A functional interface can have multiple default/static methods.
{: .prompt-info }

---

## Summary

| Concept | Description |
|---------|-------------|
| **Functional Interface** | Interface with exactly one abstract method |
| **Declaration** | `fun interface` in Kotlin |
| **SAM Conversion** | Lambda → Functional Interface instance (automatic) |
| **Java Difference** | Java uses annotation (optional), Kotlin uses keyword (required) |
| **Default Methods** | Don't count as abstract methods |

### Key Takeaways

1. Use `fun interface` when you want lambda support for your interface
2. Only **one** abstract method is allowed
3. Default methods and static methods are permitted
4. Kotlin requires explicit `fun` keyword for safety (unlike Java's optional annotation)

---

## References

- [Kotlin Documentation - Functional (SAM) interfaces](https://kotlinlang.org/docs/fun-interfaces.html)
- [Java Documentation - Functional Interfaces](https://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html)
- [Kotlin 1.4 Release Notes - SAM conversions for Kotlin interfaces](https://kotlinlang.org/docs/whatsnew14.html#sam-conversions-for-kotlin-interfaces)

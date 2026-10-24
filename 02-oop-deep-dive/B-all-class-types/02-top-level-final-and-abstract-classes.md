# B.2 Top-level, Final and Abstract Classes

> **Goal:** Master the plain class kinds: how top-level classes are organized, when `final` and `abstract` help, and the patterns built from them (singleton, utility class).

**Level:** refresher plus design patterns that rely on class modifiers.
**Verified on:** OpenJDK 21.0.12. Every output below was produced by running the code.

## Contents

1. [Top-level classes and files](#1-top-level-classes-and-files)
2. [`final` classes](#2-final-classes)
3. [`abstract` classes](#3-abstract-classes)
4. [Combining modifiers](#4-combining-modifiers)
5. [Pattern: utility class](#5-pattern-utility-class)
6. [Pattern: singleton, five ways](#6-pattern-singleton-five-ways)
7. [Common gotchas](#7-common-gotchas)
8. [Interview questions](#8-interview-questions)

---

## 1. Top-level classes and files

A compilation unit (`.java` file) may hold many top-level types but at most **one `public`** one, and its name must match the file. Others are package-private. Practically: one top-level type per file.

| Allowed on a top-level class | Not allowed |
|---|---|
| `public`, package-private | `private`, `protected`, `static` |
| `final`, `abstract`, `sealed`, `non-sealed` | `final` together with `abstract` |
| `strictfp` (redundant since 17) | |

## 2. `final` classes

**Intuition.** A `final` class is a sealed box: nobody can add behaviour by subclassing. That gives you guarantees (immutability, security) and lets the JIT inline calls, at the cost of flexibility (mocking frameworks need extra support).

```java
public class ExtendFinal {
    static final class Sealed { }
    static class Attempt extends Sealed { }           // ERROR: cannot inherit from final class

    public static void main(String[] args) { }
}
```

Real `javac` output:

```text
ce/ExtendFinal.java:3: error: cannot inherit from final Sealed
    static class Attempt extends Sealed { }           // ERROR: cannot inherit from final class
                                 ^
1 error
```

Use `final` for value classes (`String`, `Integer`), security-sensitive classes, and any class not designed for inheritance. Effective Java's advice stands: design for inheritance or forbid it. Since Java 17 there is a middle path, `sealed` (chapter B.9).

## 3. `abstract` classes

An abstract class is an incomplete blueprint. It may declare abstract methods, and cannot be instantiated (see chapter A.4).

```java
public class AbstractNew {
    static abstract class Shape { abstract double area(); }

    public static void main(String[] args) {
        Shape s = new Shape();                 // ERROR: abstract classes cannot be instantiated
    }
}
```

Real `javac` output:

```text
ce/AbstractNew.java:5: error: Shape is abstract; cannot be instantiated
        Shape s = new Shape();                 // ERROR: abstract classes cannot be instantiated
                  ^
1 error
```

## 4. Combining modifiers

| Combination | Legal? | Why |
|---|---|---|
| `abstract final class` | no | needs a subclass vs forbids one |
| `abstract` method in a non-abstract class | no | class would be incomplete |
| `final` method in an `abstract` class | yes | template method (A.4) |
| `private` constructor + `final` class | yes | utility class, singleton |
| `sealed abstract class` | yes | closed family of variants |

## 5. Pattern: utility class

A holder of static helpers: `final`, private constructor, and (optionally) a guard against reflective instantiation.

## 6. Pattern: singleton, five ways

```java
import java.lang.reflect.Constructor;

public class SingletonVariants {

    // 1. Eager: created when the class initializes
    static final class Eager {
        static { System.out.println("  [Eager initialized]"); }
        private static final Eager INSTANCE = new Eager();
        private Eager() { }
        static Eager getInstance() { return INSTANCE; }
    }

    // 2. Lazy holder idiom: the JVM's class-initialization lock makes it lazy AND thread-safe, no synchronization needed
    static final class Lazy {
        static { System.out.println("  [Lazy outer class initialized]"); }
        private Lazy() { System.out.println("  [Lazy instance created]"); }
        private static final class Holder {
            static { System.out.println("  [Lazy.Holder initialized]"); }
            static final Lazy INSTANCE = new Lazy();
        }
        static Lazy getInstance() { return Holder.INSTANCE; }
        static void unrelatedStaticMethod() { System.out.println("  unrelated static method called"); }
    }

    // 3. Enum singleton: serialization-safe and reflection-safe
    enum Registry {
        INSTANCE;
        private int counter;
        int next() { return ++counter; }
    }

    // Utility class: not meant to be instantiated
    static final class Strings {
        private Strings() { throw new AssertionError("no instances"); }
        static boolean isPalindrome(String s) { return new StringBuilder(s).reverse().toString().equals(s); }
    }

    public static void main(String[] args) throws Exception {
        System.out.println("calling Eager.getInstance():");
        Eager.getInstance();

        System.out.println("calling Lazy.unrelatedStaticMethod():");
        Lazy.unrelatedStaticMethod();
        System.out.println("calling Lazy.getInstance() (first time):");
        Lazy.getInstance();
        System.out.println("same instance? " + (Lazy.getInstance() == Lazy.getInstance()));

        System.out.println("Registry: " + Registry.INSTANCE.next() + ", " + Registry.INSTANCE.next());

        // Reflection can break a private constructor...
        Constructor<Eager> c = Eager.class.getDeclaredConstructor();
        c.setAccessible(true);
        System.out.println("reflection created a SECOND Eager? " + (c.newInstance() != Eager.getInstance()));

        // ...but not an enum
        try {
            Constructor<?> ec = Registry.class.getDeclaredConstructors()[0];
            ec.setAccessible(true);
            ec.newInstance("X", 1);
        } catch (IllegalArgumentException e) {
            System.out.println("enum: " + e.getMessage());
        }

        System.out.println("palindrome: " + Strings.isPalindrome("level"));
        try {
            Constructor<Strings> sc = Strings.class.getDeclaredConstructor();
            sc.setAccessible(true);
            sc.newInstance();
        } catch (java.lang.reflect.InvocationTargetException e) {
            System.out.println("utility class guard: " + e.getCause());
        }
    }
}

// ---- Output ----
// calling Eager.getInstance():
//   [Eager initialized]
// calling Lazy.unrelatedStaticMethod():
//   [Lazy outer class initialized]
//   unrelated static method called
// calling Lazy.getInstance() (first time):
//   [Lazy.Holder initialized]
//   [Lazy instance created]
// same instance? true
// Registry: 1, 2
// reflection created a SECOND Eager? true
// enum: Cannot reflectively create enum objects
// palindrome: true
// utility class guard: java.lang.AssertionError: no instances
```

| Variant | Lazy? | Thread-safe? | Notes |
|---|---|---|---|
| Eager `static final` field | no | yes (class init lock) | simplest; created even if unused |
| Initialization-on-demand holder | yes | yes (class init lock) | the idiom: the nested `Holder` initializes on first `getInstance()` |
| `enum` with one constant | no (at enum load) | yes | reflection and serialization safe; Effective Java's recommended default |
| Double-checked locking with `volatile` | yes | yes if `volatile` | verbose; needs correct memory-model use (Part 11) |
| DI container scope | n/a | container | preferable in applications |

Why the holder idiom works: the JVM guarantees a class is initialized at most once and exactly under a lock; the `Holder` class is not initialized until it is first touched, so the singleton is lazy and no explicit synchronization is needed. The output above proves the order: `Lazy` initializes without creating the instance, `Holder` only on `getInstance()`.

Singleton hazards: reflection can invoke a private constructor (the demo created a second instance), serialization can recreate it (implement `readResolve` or use an enum), multiple class loaders mean multiple singletons, and global state makes testing hard.

## 7. Common gotchas

| Gotcha | Why | Fix |
|---|---|---|
| Public class name differs from file name | compiler error | match the names |
| Mocking a `final` class fails | subclass-based mocks cannot extend it | inline mock maker (Part 17) or use an interface |
| Non-volatile double-checked locking | partially constructed object published | `volatile` or holder idiom |
| Singleton broken by serialization | deserialization creates a new object | enum singleton or `readResolve` |
| Utility class instantiable | default constructor is public | private constructor |

## 8. Interview questions

<details>
<summary><strong>Q1. Why is <code>String</code> final?</strong></summary>

Immutability underpins security (class names, file paths, network targets), thread safety, hash caching and the string pool. A subclass could break these guarantees.
</details>

<details>
<summary><strong>Q2. Can an abstract class have no abstract methods?</strong></summary>

Yes. It still cannot be instantiated; this is used to force subclassing or share a skeleton.
</details>

<details>
<summary><strong>Q3. Can you have more than one public class in a file?</strong></summary>

No. At most one top-level public type, named like the file.
</details>

<details>
<summary><strong>Q4. Why does the initialization-on-demand holder idiom need no synchronization?</strong></summary>

Class initialization is serialized by the JVM and happens once; the nested holder initializes lazily on first access, publishing the instance safely.
</details>

<details>
<summary><strong>Q5. How do you make a singleton safe against reflection and serialization?</strong></summary>

Use a single-element enum: the JVM forbids reflective enum construction ("Cannot reflectively create enum objects" in the demo) and serialization preserves the constant.
</details>

<details>
<summary><strong>Q6. What are the downsides of singletons?</strong></summary>

Hidden global state, hard-to-test code, tight coupling, per-class-loader duplication. Prefer dependency injection with application scope.
</details>

<details>
<summary><strong>Q7. When should a class be <code>final</code>?</strong></summary>

When it is not designed for extension, when immutability or invariants must hold, or for security. Otherwise document the extension points.
</details>

<details>
<summary><strong>Q8. How does <code>final</code> affect performance?</strong></summary>

Little in modern JVMs; the JIT does class-hierarchy analysis and devirtualizes calls regardless. Choose `final` for design reasons, not speed.
</details>

---

[Previous: B.1 Class Types](./01-class-types-taxonomy-and-decision-guide.md) · [Part index](../README.md) · Next: [B.3 Static Nested Classes](./03-static-nested-classes.md)
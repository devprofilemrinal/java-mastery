# C.4 OOP Puzzles and Gotchas

> **Goal:** Train your eye on the tricky questions interviewers use: initialization order, overload resolution, overriding rules, defaults, ternary typing.

**Level:** practice. Predict each output **before** opening the answer; every answer is real output.
**Verified on:** OpenJDK 21.0.12.

## Contents

1. [How to use this chapter](#1-how-to-use-this-chapter)
2. [Puzzle 1: static forward reference](#2-puzzle-1-static-forward-reference)
3. [Puzzle 2: `null` and overloads](#3-puzzle-2-null-and-overloads)
4. [Puzzle 3: the private "override"](#4-puzzle-3-the-private-override)
5. [Puzzle 4: equals and hashCode](#5-puzzle-4-equals-and-hashcode)
6. [Puzzle 5: three `x`](#6-puzzle-5-three-x)
7. [Puzzle 6: class beats interface](#7-puzzle-6-class-beats-interface)
8. [Puzzle 7: the ternary operator](#8-puzzle-7-the-ternary-operator)
9. [Puzzle 8: override called from constructor](#9-puzzle-8-override-called-from-constructor)
10. [Rules cheat sheet](#10-rules-cheat-sheet)
11. [Interview questions](#11-interview-questions)

---

## 1. How to use this chapter

For each puzzle: read the code, write down your predicted output, then open the answer. Then read the explanation, which names the rule. If you missed one, re-read the chapter in the link.

## 2. Puzzle 1: static forward reference

```java
public class Q1StaticForward {
    static int a = init("a", b());       // b() runs before b is assigned
    static int b = init("b", 5);
    static int b() { return b; }         // reads the default 0
    static int init(String n, int v) { System.out.println(n + " = " + v); return v; }

    public static void main(String[] args) {
        System.out.println(a + " " + b);
    }
}
```

<details>
<summary>Predict the output first, then open</summary>

```text
a = 0
b = 5
0 5
```
</details>

**Explanation.** Static initializers run top to bottom. `a`'s initializer calls `b()` while `b` still holds its default `0`; only then is `b = 5` executed. A forward *simple-name* reference in an initializer is a compile error, but reading through a method hides it from the compiler. See A.1.

## 3. Puzzle 2: `null` and overloads

```java
public class Q2NullOverload {
    static void f(Object o) { System.out.println("Object"); }
    static void f(String s) { System.out.println("String"); }
    static void g(Object o) { System.out.println("g Object"); }
    static void g(Integer i) { System.out.println("g Integer"); }
    static void g(long l)    { System.out.println("g long"); }

    public static void main(String[] args) {
        f(null);
        f((Object) null);
        g(5);
        g(Integer.valueOf(5));
        g('a');
    }
}
```

<details>
<summary>Predict the output first, then open</summary>

```text
String
Object
g long
g Integer
g long
```
</details>

**Explanation.** The compiler picks the **most specific** applicable method: `String` is more specific than `Object`, so `f(null)` calls the `String` version; the cast makes the static type `Object`. For `g(5)`: phase 1 (no boxing) finds `g(long)` by widening, so it beats boxing to `Integer`; phase 2 (boxing) is only tried if phase 1 finds nothing. `g('a')` also widens `char` to `long`. Order of phases: widening, boxing, varargs. See A.3.

## 4. Puzzle 3: the private "override"

```java
public class Q3PrivateNotOverridden {
    static class A {
        private void who() { System.out.println("A.who"); }
        void run() { who(); }
    }
    static class B extends A {
        void who() { System.out.println("B.who"); }   // NOT an override: A.who is invisible
    }
    public static void main(String[] args) {
        new B().run();
        ((B) new B()).who();
    }
}
```

<details>
<summary>Predict the output first, then open</summary>

```text
A.who
B.who
```
</details>

**Explanation.** `A.who` is private, so `B` does not inherit it; `B.who` is a new unrelated method. `A.run` calls `who()`, which is bound to `A.who` because private methods are not virtual. See A.3 and B.12 (`invokespecial`/nestmates).

## 5. Puzzle 4: equals and hashCode

```java
import java.util.*;

public class Q4HashSets {
    static class OnlyEquals {
        final int v; OnlyEquals(int v) { this.v = v; }
        @Override public boolean equals(Object o) { return o instanceof OnlyEquals e && e.v == v; }
    }
    static class OnlyHash {
        final int v; OnlyHash(int v) { this.v = v; }
        @Override public int hashCode() { return v; }
    }
    static class Both {
        final int v; Both(int v) { this.v = v; }
        @Override public boolean equals(Object o) { return o instanceof Both e && e.v == v; }
        @Override public int hashCode() { return Integer.hashCode(v); }
    }
    public static void main(String[] args) {
        Set<OnlyEquals> a = new HashSet<>(List.of(new OnlyEquals(1)));
        Set<OnlyHash> b = new HashSet<>(List.of(new OnlyHash(1)));
        Set<Both> c = new HashSet<>(List.of(new Both(1)));
        System.out.println(a.contains(new OnlyEquals(1)) + " " + b.contains(new OnlyHash(1)) + " " + c.contains(new Both(1)));
    }
}
```

<details>
<summary>Predict the output first, then open</summary>

```text
false false true
```
</details>

**Explanation.** Only the class overriding both works in a `HashSet`. With only `equals`, the new instance has a different identity hash and normally lands in a different bucket. With only `hashCode`, the bucket matches but `equals` is identity, so the other instance is not found. See A.5. (The first result is probabilistic in theory, though identity hashes make a collision extremely unlikely; the program printed the stable result here.)

## 6. Puzzle 5: three `x`

```java
public class Q5InnerThis {
    String name = "outer";
    class Inner {
        String name = "inner";
        void show(String name) {
            System.out.println(name + " " + this.name + " " + Q5InnerThis.this.name);
        }
    }
    public static void main(String[] args) {
        new Q5InnerThis().new Inner().show("param");
    }
}
```

<details>
<summary>Predict the output first, then open</summary>

```text
param inner outer
```
</details>

**Explanation.** The parameter shadows the inner field, which shadows the outer field. `name` is the parameter, `this.name` the inner instance's field, `Q5InnerThis.this.name` the enclosing instance's field. See B.4.

## 7. Puzzle 6: class beats interface

```java
public class Q6ClassWins {
    interface Walker { default String move() { return "walk"; } }
    static class Animal { public String move() { return "animal"; } }
    static class Dog extends Animal implements Walker { }          // class wins over interface default

    interface A { default String hi() { return "A"; } }
    interface B extends A { default String hi() { return "B"; } }   // more specific interface wins
    static class C implements A, B { }

    public static void main(String[] args) {
        System.out.println(new Dog().move() + " " + new C().hi());
    }
}
```

<details>
<summary>Predict the output first, then open</summary>

```text
animal B
```
</details>

**Explanation.** For `Dog`, a concrete method inherited from a **class** (`Animal.move`) always beats an interface default. For `C`, `B.hi()` wins because `B` extends `A` and is therefore more specific; no conflict. See A.4.

## 8. Puzzle 7: the ternary operator

```java
public class Q7Ternary {
    public static void main(String[] args) {
        System.out.println(true ? 1 : 2.0);
        Object o = true ? 1 : "x";
        System.out.println(o);
        char c = 'b';
        int i = 0;
        System.out.println(false ? i : c);      // i is a variable: binary numeric promotion to int
        System.out.println(false ? 0 : c);      // 0 is a constant that fits in char: result type char
        Integer n = null;
        try { int x = true ? n : 0; } catch (NullPointerException e) { System.out.println("NPE from unboxing"); }
    }
}
```

<details>
<summary>Predict the output first, then open</summary>

```text
1.0
1
98
b
NPE from unboxing
```
</details>

**Explanation.** If both operands are numeric, **binary numeric promotion** applies: `true ? 1 : 2.0` is a `double`, so `1.0`. With `Object o = true ? 1 : "x"`, the result is boxed `Integer 1`. In `false ? i : c` with a variable `i`, the result type is `int`, so the `char` prints as its number `98`; with the constant `0` (which fits a `char`) the type is `char`, so it prints `b`. Last, `true ? n : 0` with `Integer n = null` unboxes `n` (type `int`), giving `NullPointerException`. See Part 1, chapter 1.1.

## 9. Puzzle 8: override called from constructor

```java
public class Q8CtorOverride {
    static class Base {
        Base() { init(); }
        void init() { System.out.println("Base.init"); }
    }
    static class Derived extends Base {
        int x = 10;
        int y;
        Derived() { y = 5; }
        @Override void init() { System.out.println("Derived.init x=" + x + " y=" + y); x = 99; }
    }
    public static void main(String[] args) {
        Derived d = new Derived();
        System.out.println("x=" + d.x + " y=" + d.y);
    }
}
```

<details>
<summary>Predict the output first, then open</summary>

```text
Derived.init x=0 y=0
x=10 y=5
```
</details>

**Explanation.** `Base()` calls the overridden `init()` before `Derived`'s field initializers run, so `x` and `y` are still `0`. Then `init` sets `x = 99`, but `Derived`'s field initializer `x = 10` runs afterwards and overwrites it; finally the constructor body sets `y = 5`. See A.1, section 6.

## 10. Rules cheat sheet

| Topic | Rule |
|---|---|
| Overload resolution | phases: widening, boxing, varargs; most specific wins; static types only |
| Override vs hide | instance methods override; static methods and fields hide |
| Private | never inherited, never virtual |
| Default methods | class wins, then most specific interface, else error |
| Initialization | statics (parent first), then per-object: parent init+ctor, child init+ctor |
| Ternary | numeric operands promote; boxed `null` unboxes |
| `equals`/`hashCode` | override both, same fields |
| Captured variables | effectively final |

## 11. Interview questions

<details>
<summary><strong>Q1. What is the order of overload resolution phases?</strong></summary>

1) strict invocation (identity, widening) 2) loose invocation (boxing/unboxing) 3) variable-arity. The first phase that finds an applicable method wins, then the most specific is chosen.
</details>

<details>
<summary><strong>Q2. Why does <code>f(null)</code> with <code>f(Object)</code> and <code>f(String)</code> pick String?</strong></summary>

<code>String</code> is a subtype of <code>Object</code>, so it is more specific. With <code>f(String)</code> and <code>f(Integer)</code> the call is ambiguous and does not compile.
</details>

<details>
<summary><strong>Q3. Can you override a private method?</strong></summary>

No; a same-signature method in a subclass is a new method.
</details>

<details>
<summary><strong>Q4. What does <code>System.out.println(true ? 1 : 2.0)</code> print and why?</strong></summary>

<code>1.0</code>: binary numeric promotion makes the conditional a double.
</details>

<details>
<summary><strong>Q5. What happens when a constructor calls an overridable method?</strong></summary>

The subclass override runs before the subclass fields are initialized (puzzle 8).
</details>

<details>
<summary><strong>Q6. Class and interface both define the same method: which wins?</strong></summary>

The class method (puzzle 6).
</details>

<details>
<summary><strong>Q7. Why can a <code>HashSet</code> fail to find an object with only <code>equals</code> overridden?</strong></summary>

Different hash buckets (puzzle 4).
</details>

<details>
<summary><strong>Q8. How can a static initializer read a variable's default value?</strong></summary>

Through a method call before the declaration is executed (puzzle 1).
</details>
<details>
<summary><strong>Q9. Is <code>Integer i = null; int x = flag ? i : 0;</code> safe?</strong></summary>

No: the conditional's type is <code>int</code>, so <code>i</code> is unboxed and throws <code>NullPointerException</code> when <code>flag</code> is true.
</details>

<details>
<summary><strong>Q10. Where do you go next?</strong></summary>

Part 3 (Java 8): lambdas, functional interfaces and the Streams basics build directly on the interface and class machinery from this module.
</details>

---

[Previous: C.3 OOP with Modern Java](./03-oop-with-modern-java.md) · [Part index](../README.md) · Next: [Part 3: Java 8, Lambdas and Functional Interfaces](../../03-lts-java-8/01-lambdas-and-functional-interfaces.md)
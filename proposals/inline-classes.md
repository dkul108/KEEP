# Inline classes

* **Type**: Design proposal
* **Author**: Mikhail Zarechenskiy
* **Contributors**: Andrey Breslav, Denis Zharkov, Dmitry Petrov, Ilya Gorbunov, Roman Elizarov, Stanislav Erokhin
* **Status**: Under consideration
* **Prototype**: Implemented in Kotlin 1.2.30

Discussion of this proposal is held in [this issue](https://github.com/Kotlin/KEEP/issues/104).

## Summary

Currently, there is no performant way to create wrapper for a value of a corresponding type. The only way is to create a usual class, 
but the use of such classes would require additional heap allocations, which can be critical for many use cases.    

We propose to support identityless inline classes that would allow to introduce wrappers for values without additional overhead related 
to additional heap allocations.

## Motivation / use cases

Inline classes allow to create wrappers for a value of a certain type and such wrappers would be fully inlined. 
This is similar to type aliases but inline classes are not assignment-compatible with the corresponding underlying types.

Use cases:

- Unsigned types
```kotlin
inline class UInt(private val value: Int) { ... }
inline class UShort(private val value: Short) { ... }
inline class UByte(private val value: Byte) { ... }
inline class ULong(private val value: Long) { ... }
```

- Native types like `size_t` for Kotlin/Native
- Inline enum classes 
    - Int enum for [Android IntDef](https://developer.android.com/reference/android/support/annotation/IntDef.html)
    - String enum for Kotlin/JS (see [WebIDL enums](https://www.w3.org/TR/WebIDL-1/#idl-enums))
    
    Example:
    ```kotlin
    inline enum class Foo(val x: Int) {
        A(0), B(1);
        
        fun example() { ... }
    }
    ```
    
    The constructor's arguments should be constant values and the values should be different for different entries.


- Units of measurement
- Result type (aka Try monad) [KT-18608](https://youtrack.jetbrains.com/issue/KT-18608)
- Inline property delegates
```kotlin
class A {
    var something by InlinedDelegate(Foo()) // no actual instantiation of `InlinedDelegate`
}


inline class InlinedDelegate<T>(var node: T) {
    operator fun setValue(thisRef: A, property: KProperty<*>, value: T) {
        if (node !== value) {
            thisRef.notify(node, value)
        }
        node = value
    }

    operator fun getValue(thisRef: A, property: KProperty<*>): T {
        return node
    }
}
```

- Inline wrappers
    - Typed wrappers
    ```kotlin
    inline class Name(private val s: String)
    inline class Password(private val s: String)
    
    fun foo() {
        var n = Name("n") // no actual instantiation, on JVM type of `n` is String
        val p = Password("p")
        n = "other" // type mismatch error
        n = p // type mismatch error
    }
    ```

    - API refinement
    ```
    // Java
    public class Foo {
        public Object[] objects() { ... }
    }
    
    // Kotlin
    inline class RefinedFoo(val f: Foo) {
        inline fun <T> array(): Array<T> = f.objects() as Array<T>
    }
    ``` 

## Description

Inline classes are declared using soft keyword `inline` and must have a single property:
```kotlin
inline class Foo(val i: Int)
```
Property `i` defines type of the underlying runtime representation for inline class `Foo`, while at compile time type will be `Foo`.

From language point of view, inline classes can be considered as restricted classes, they can declare various members, operators, 
have generics. 

Example:
```kotlin
inline class Name(val s: String) : Comparable<Name> {
    override fun compareTo(other: Name): Int = s.compareTo(other.s)
    
    fun greet() {
        println("Hello, $s")
    }
}    

fun greet() {
    val name = Name("Kotlin") // there is no actual instantiation of class `Name`
    name.greet() // method `greet` is called as a static method
}
```

## Current limitations

Currently, inline classes must satisfy the following requirements:

- Inline class must have a public primary constructor with a single value parameter
- Inline class must have a single read-only (`val`) property as an underlying value, which is defined in primary constructor
- Underlying value cannot be of the same type that is containing inline class
- Inline class with undefined (recursively defined) generics, e.g. generics with an upper bound equal to the class, is prohibited
    ```kotlin
    inline class A<T : A<T>>(val x: T) // error
    ```
- Inline class cannot have `init` block
- Inline class must be final
- Inline class can implement only interfaces
- Inline class cannot have backing fields
    - Hence, it follows that inline class can have only simple computable properties (no lateinit/delegated properties)
- Inline class cannot have inner classes
- Inline class must be a toplevel class 

Sidenotes:

- Let's elaborate requirement to have public primary constructor and restriction of `init` blocks.
For example, we want to have an inline class for some bounded value:
    ```kotlin
    inline class Positive(val value: Int) {
        init { 
            assert(value > 0) "Value isn't positive: $value" 
        }
    }
  
    fun foo(p: Positive) {}
    ```
    
    Because of inlining, method `foo` have type `int` from Java POV, so we can pass to method `foo` everything we want and `init` 
    block will not be executed. Since we cannot control behaviour of `init` block execution, we restrict it for inline classes.
    
    Unfortunately, it's not enough, because `init` blocks can be emulated via factory methods:
    ```kotlin
    inline class Positive private constructor(val value: Int) {
        companion object {
            fun create(x: Int) {
                assert(x > 0) "Value isn't positive: x"
                return Positive(x)  
            }  
        }
    }
  
    fun foo(p: Positive) {}
    ```
    
    Again, method `foo` have type `int` from Java POV, so we can indirectly create values of type `Positive` 
    even with the presence of private constructor.
    
    To make behaviour more predictable and consistent with Java, we demand public primary constructor and restrict `init` blocks.

### Other restrictions

The following restrictions are related to the usages of inline classes:

- Referential equality (`===`) is prohibited for inline classes
- vararg of inline class type is prohibited
```
inline class Foo(val s: String)

fun test(vararg foos: Foo) { ... } // should be an error  
```

## Java interoperability  

Each inline class has its own wrapper, which is boxed/unboxed by the same rules as for primitive types.
Basically, rule for boxing can be formulated as follows: inline class is boxed when it is used as another type.
Unboxed inline class is used when value is statically known to be inline class.

Examples:
```kotlin
interface I

inline class Foo(val i: Int) : I

fun asInline(f: Foo) {}
fun <T> asGeneric(x: T) {}
fun asInterface(i: I) {}
fun asNullable(i: Foo?) {}

fun <T> id(x: T): T = x

fun test(f: Foo) {
    asInline(f)
    asGeneric(f) // boxing
    asInterface(f) // boxing
    asNullable(f) // boxing
    
    val c = id(f) // boxing/unboxing, c is unboxed
}
```

Since boxing doesn't have side effects as is, it's possible to reuse various optimizations that are done for primitive types.

### Type mapping on JVM (without mangling)

#### Top-level types

##### Inline classes over primitive types
```kotlin
inline class ICPrimitive(val x: Int)

fun foo(i: ICPrimitive) {}
fun bar(i: ICPrimitive?) {}
```

Only values of `ICPrimitive` can be passed to the function `foo` and therefore on JVM this type will be erased to just `int`.
At the same time, it's also possible to pass `null` to the function `bar`, to handle it correctly, `ICPrimitive?` will be mapped to the 
reference type `LICPrimitive;` as on JVM primitive type `int` can't hold `null` values.

So, `ICPrimitive` -> `int`, `ICPrimitive?` -> `LICPrimitive;`.

##### Inline classes over reference types

Now, let's consider an inline class over some reference type:
```kotlin
inline class ICReference(val s: String)

fun foo(i: ICReference) {}
fun bar(i: ICReference?) {}
```

With the type `ICReference` rationale is the same, it can't hold `nulls`, so this type will be mapped to `String`.
Next, function `bar` can hold `null` values, but note that underlying type of `ICReference` is a reference type `String`, which
can hold `null` values on JVM and can be safely used as a mapped type.

So, `ICReference` -> `String`, `ICReference?` -> `String`.

##### Inline classes over nullable types

Now, what if inline class is declared over some nullable type?
```kotlin
inline class ICNullable(val s: String?)

fun foo(i: ICNullable) {}
fun bar(i: ICNullable?) {}
```

`ICNullable` can't hold `nulls`, so it can be safely mapped to `String` on JVM.
`ICNullable?` can hold `nulls` and also inline classes over `nulls`: `ICNullable(null)`.
It's important to distinguish such values:
```kotlin
fun baz(a: ICNullable?, b: ICNullable?) {
    if (a === b) { ... }
}

fun test() {
    baz(ICNullable(null), null)
}
``` 
If we map `ICNullable?` to `String` as in the previous example, it will not be possible to distinguish `ICNullable(null)` from `null` as on JVM
they will be represented with the just value `null`, therefore `ICNullable?` should be mapped to the `LICNullable;`

So, `ICNullable` -> `String`, `ICNullable?` -> `LICNullable;`.

##### Inline classes over other inline classes

Besides these cases, inline class can also be declared over some other inline class:
```kotlin
inline class IC2(val i: IC)
inline class IC2Nullable(val i: IC?)
```

Mapping rules for `IC2Nullable` are simple:
- `IC2Nullable` -> mapped type of `IC?`
- `IC2Nullable?` -> `LICNullable;`

Mapping rules for `IC2` are the following:
- `IC2` -> mapped type of `IC`
- `IC2?` -> 
    - fully mapped type of `IC` if it's a non-null reference type
    - `LIC2;` if fully mapped type of `IC` can hold `nulls` or it's a primitive type

Rationale for these rules is the same as in the previous steps: for nullable types, it should be possible to hold
`null` and distinguish `nulls` from inline classes over `nulls`. 

Example, let's consider the following hierarchy of inline classes:
```
inline class IC1(val s: String)
inline class IC2(val ic1: IC1?)
inline class IC3(val ic2: IC2)

fun foo(i: IC3) {}
fun bar(i: IC3?) {} 
```

Here `IC3` will be mapped to the type `String`, `IC3?` will be mapped to `LIC3;` as it should be possible to distinguish `null` 
from `IC3(IC2(null))`. But if `IC2` was declared as `inline class IC2(val ic1: IC1)`, then `IC3` would be mapped to `String`.

#### Generic types

If inline class type is used in generic position, then its boxed type will be used:
```
// Kotlin: sample.kt

inline class Name(val s: String)

fun generic(names: List<Name>) {} // generic signature will have `List<Name>` as for parameters type
fun simple(): Name = Name("Kt")

// Java
class Test {
    void test() {
        String name = SampleKt.simple();
        List<Name> ls = Samplekt.generic(); // from Java POV it's List<Name>, not List<String>
    }
}
```

This is needed to preserve information about inline classes at runtime.

#### Generic inline class mapping

Consider the following sample:
```kotlin
inline class Generic<T>(val x: T)

fun foo(g: Generic<Int>) {}
```

Now, type `Generic<Int>` can be mapped either to `java.lang.Integer`, `java.lang.Object` or to primitive `int`.

Same question arises with arrays:
```kotlin
inline class GenericArray<T>(val y: Array<T>)

fun foo(g: GenericArray<Int>) {} // `g` has type `Integer[]` or `Object[]`?
``` 

Therefore, because of this ambiguity, such cases are going to be forbidden in the first version of inline classes.

* Sidenote: maybe it's worth to consider inline classes with reified generics:
    ```kotlin
    inline class Reified<reified T>(val x: T)
    
    fun foo(a: Reified<Int>, b: Reified<String>) // a has type `Int`, b has type `String`
    ``` 

Generic inline classes with underlying value not of type that defined by type parameter or generic array are mapped as usual generics:
```kotlin
inline class AsList<T>(val ls: List<T>)

fun foo(param: AsList<String>) {}
```

In JVM signature `param` will have type `java.util.List`, 
but in generic signature it will be `java.util.List<java.lang.String>` 

## Methods from `kotlin.Any`

Inline classes are indirectly inherited from `Any`, i.e. they can be assigned to a value of type `Any`, but only through boxing.

Methods from `Any` (`toString`, `hashCode`, `equals`) can be useful for a user-defined inline classes and therefore should be customizable. 
Methods `toString` and `hashCode` can be overridden as usual methods from `Any`. For method `equals` we're going to introduce new operator 
that represents "typed" `equals` to avoid boxing for inline classes:
```kotlin
inline class Foo(val s: String) {
    operator fun equals(other: Foo): Boolean { ... }
}
```

Compiler will generate original `equals` method that is delegated to the typed version.  

By default, compiler will automatically generate `equals`, `hashCode` and `toString` same as for data classes.

## Arrays of inline class values

Consider the following inline class:
```kotlin
inline class Foo(val x: Int)
``` 

To represent array of unboxed values of `Foo` we propose to use new inline class `FooArray`:
```kotlin
inline class FooArray(private val storage: IntArray): Collection<Foo> {
    operator fun get(index: Int): UInt = Foo(storage[index])
    ...
}
```
While `Array<Foo>` will represent array of **boxed** values:
```
// jvm signature: test([I[LFoo;)V
fun test(a: FooArray, b: Array<Foo>) {} 
``` 

This is similar how we work with arrays of primitive types such as `IntArray`/`ByteArray` and allows to explicitly differ array of 
unboxed values from array of boxed values.

This decision doesn't allow to declare `vararg` parameter that will represent array of unboxed inline class values, because we can't
associate vararg of inline class type with the corresponding array type. For example, without additional information it's impossible to match
`vararg v: Foo` with `FooArray`. Therefore, we are going to prohibit `vararg` parameters for now.

#### Other possible options:

- Treat `Array<Foo>` as array of unboxed values by default

    Pros:
    - There is no need to define separate class to introduce array of inline class type
    - It's possible to allow `vararg` parameters
    
    Cons:
    - `Array<Foo>` can implicitly represent array of unboxed and array of boxed values:
    ```java
    // Java
    class JClass {
        public static void bar(Foo[] f) {}
    }
    ```
    From Kotlin point of view, function `bar` can take only `Array<Foo>`
    - Not clear semantics for generic arrays:
    ```kotlin
    fun <T> genericArray(a: Array<T>) {}

    fun test(foos: Array<Foo>) {
      genericArray(foos) // conversion for each element? 
    }
    ```

 
- Treat `Array<Foo>` as array of boxed values and introduce specialized `VArray` class with the following rules:
    - `VArray<Foo>` represents array of unboxed values
    - `VArray<Foo?>` or `VArray<T>` for type paramter `T` is an error
    - (optionally) `VArray<Int>` represents array of primitives
    
    Pros:
    - There is no need to define separate class to introduce array of inline class type
    - It's possible to allow `vararg` parameters
    - Explicit representation for arrays of boxed/unboxed values
    
    Cons:
    - Complicated implementation and overall design


## Expect/Actual inline classes

To declare expect inline class one can use `expect` modifier:
```kotlin
expect inline class Foo(val prop: String)
```

Note that we allow to declare property with backing field (`prop` here) for expect inline class, which is different for usual classes.
Also, since each inline class must have exactly one value parameter we can relax rules for actual inline classes:
```kotlin
// common module
expect inline class Foo(val prop: String)

// platform-specific module
actual inline class Foo(val prop: String)
```
For actual inline classes we don't require to write `actual` modifier on primary constructor and value parameter.

Currently, expect inline class requires actual inline and vice versa. 

## Overloads, private constructors and initialization blocks

Let's consider several most important issues that appear in the current implementation.

*Overloads*

Signatures of overloads with inline classes that are erased to the same type on the same position will be conflicting:
```kotlin
inline class UInt(val u: Int)

// Conflicting overloads
fun compute(i: Int) { ... }
fun compute(u: UInt) { ... }

inline class Login(val s: String)
inline class UserName(val s: String)

// Conflicting overloads
fun foo(x: Login) {}
fun foo(x: UserName) {}
```

One could use `JvmName` to disambiguate functions, but this looks verbose and confusing. Inline class types are normal types 
and we'd like to think about inline classes as about usual classes with several restrictions, 
it allows thinking less about implementation details.
    
*Non-public constructors and initialization blocks*

Current restrictions for inline classes require having a public primary constructor without `init` blocks in order 
to have clear initialization semantics. This is needed because of values that can come from Java:
```
// Kotlin
inline class Foo(val x: Int) 

fun kotlinFun(f: Foo) {}

// Java:

static void test() {
    kotlinFun(42); // constructor or initialization block wasn't called
}
```
    
As a result, it's impossible to encapsulate underlying value or create an inline class that will represent some constrained values:
```kotlin
inline class Negative(val x: Int) {
    init {
        require(x < 0) { ... }
    }
}
```

Note that these problems can go away if we'll use inline classes (which is a Kotlin-only feature) only in Kotlin:
there are no problems with initialization, so we can add non-public constructors with `init` blocks, for overloads 
we can use different names on JVM.

### Mangling

To mitigate described problems, we propose to do mangling for declarations that have top-level inline class types in their signatures.
Example:
```kotlin
inline class UInt(val x: Int)

fun compute(x: UInt) {}
fun compute(x: Int) {}
```

We'll compile function `compute(UInt)` to `compile-<hash>(Int)`, where `<hash>` is a mangling suffix for the signature.
Now it will not possible to call this function from Java because `-` is an illegal symbol there, but from Kotlin point of view 
it's a usual function with the name `compute`.

As these functions are accessible only from Kotlin, the problem about non-public primary constructors and `init` blocks becomes easier. 

#### Mangling rules

*Simple functions*

Simple functions with inline class type parameters are mangled as `<name>-<hash>`, where `<name>` is the original function name, 
and `<hash>` is a mangling suffix for the signature. Mangling suffix can contain upper case and lower case Latin letters, digits, `_` and `-`. 
This scheme applies to property getters and setters as well.

*Constructors* 

Constructors with inline class type parameters are marked as private, and have a public synthetic accessor with additional marker parameter. 
Note that unlike mangled simple functions, hidden constructors can clash, but we consider that a less important issue than type safety.

*Functions inside inline class*

Each function inside inline class is mangled. By default, if such function doesn't have a parameter of inline class type, then it will
get suffix `-impl`, otherwise, it will be mangled as a simple function with inline class type parameters.

*Overridden functions inside inline class*

Overridden functions inside inline class are mangled same as usual ones, but compiler we'll also generate bridge to override function 
from interface.

### Inline classes ABI (JVM)

Let's consider the following inline class:
```kotlin
interface Base {
    fun base(s: String): Int
}

inline class IC(val u: Int) : Base {
    fun simple(y: String) {}
    fun icInParameter(ic: IC, y: String) {}
    
    val simpleProperty get() = 42
    val propertyIC get() = IC(42)
    var mutablePropertyIC: IC
            get() = IC(42)
            set(value) {}
    
    override fun base(s: String): Int = 0
    override fun toString(): String = "IC = $u"
}
```

On JVM this inline class will have next declarations:
```
public final class IC implements Base {
    // Underlying field
    private final I u

    // Members

    public base(Ljava/lang/String;)I
    public toString()Ljava/lang/String;

    // Auto generated methods from Any
    public equals(Ljava/lang/Object;)Z
    public hashCode()I

    // Synthetic constructor to hide it from Java
    private synthetic <init>(I)V

    // function to create and initialize value for `IC` class
    public static constructor-impl(I)I

    // fun simple(y: String) {}
    public final static simple-impl(ILjava/lang/String;)V

    // fun icInParameter(ic: IC, y: String) {}
    public final static icInParameter-8euKKQA(IILjava/lang/String;)V

    // val simpleProperty: Int
    public final static getSimpleProperty-impl(I)I

    // val propertyIC: IC
    public final static getPropertyIC-impl(I)I

    // getter of var mutablePropertyIC: IC
    public final static getMutablePropertyIC-impl(I)I

    // setter of var mutablePropertyIC: IC
    public final static setMutablePropertyIC-kVEzI7o(II)V

    // override fun base(s: String): Int 
    public static base-impl(ILjava/lang/String;)I

    // override fun toString(): String
    public static toString-impl(I)Ljava/lang/String;

    // Methods to box/unbox value of inline class type
    public final static synthetic box-impl(I)Lorg/jetbrains/kotlin/resolve/IC;
    public final synthetic unbox-impl()I

    // Static versions of auto-generated methods from Any
    public static hashCode-impl(I)I
    public static equals-impl(ILjava/lang/Object;)Z

    // Reserved method for specialized equals to avoid boxing
    public final static equals-impl0(II)Z 
}
```

Note that member-constructor (`<init>`) is synthetic, this is needed because with the addition of `init` blocks it will not evaluate them,
they will be evaluated in `constructor-impl`. Therefore we should hide it to avoid creating non-initialized values from Java. 
To make it more clear, consider the following situation:
```kotlin
inline class WithInit(val u: Int) {
    init { ... }
}

fun foo(): List<WithInit> {
    // call `constructor-impl` and evaluate `init` block
    val u = WithInit(0)
     
    // call `box-impl` and constructor (`init`), DO NOT evaluate `init` block again 
    return listOf(u)  
}
```

Note that declarations that have inline class types in parameters not on top-level will not be mangled:
```
inline class IC(val u: Int)

fun foo(ls: List<IC>) {}
```

Function `foo` will have name `foo` in the bytecode.

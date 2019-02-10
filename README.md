# A Go immutable types/values proposal

Old versions:
* [the proposal thread](https://github.com/golang/go/issues/29422).
* [the initial proposal](README-v0.md)
* [the `var:N` version](README-v1.md)
* [the pure-immutable-value interpretation version](README-v2.md)
* [the immutable-type interpretation version](README-v3.md)
* [the immutable-type/value interpretation version: const.fixed](README-v4.md)
* [the immutable-type/value interpretation version: final.fixed. With interface design flaw](README-v5.md)


This proposal not Go 1 compatible. Please read the last section of this proposal for incompatible cases.

Any criticisms and improvement ideas are welcome, for
* I have not much compiler-related knowledge, so the following designs may have flaws.
* I haven't found a perfect syntax notation set for this proposal yet.

### The problems this proposal tries to solve

The problems this proposal tries to solve:
1. no ways to declare package-level immutable non-basic values.
1. no ways to declare immutable function parameters and results.

Please note, the immutability semantics in this proposal is different from either the `const` values in C/C++.
The following sections will describe the differences.

### The main points of this proposal

We know each value has a property, `self_modifiable`, which means whether or not that value is modifiable.

This proposal will add a new value property `ref_modifiable` for each value, which means
whether or not the values referenced (either directly or indirectly) by that value are modifiable.

The permutation of thw two properties result 4 genres of values:
1. `{self_modifiable: true, ref_modifiable: true}`.
   Such as variables.
1. `{self_modifiable: true, ref_modifiable: false}`.
   No such Go values currently.
1. `{self_modifiable: false, ref_modifiable: true}`.
   Such as composite literals.
   (In fact, all declared constants in JavaScript and all final variables decalred in Java belong to this genre.)
1. `{self_modifiable: false, ref_modifiable: false}`.
   No such Go values currently.

(Note, in fact, we can catagory declared function values, method values and constant basic values into either the 3rd or the 4th genre.)

This proposal treats the `self_modifiable` as a direct value property,
and treats `ref_modifiable` as a type property (an indirect value property).

This proposal will let Go support the two value genres the current Go doesn't support,
and extend the range of `{self_modifiable: false, ref_modifiable: true}` values.
* `{self_modifiable: true}` values are declared with `var`.
* `{self_modifiable: false}` values are declared with `final` (a new keyword).
   Please note that, although a `final` value itself can't be modified,
   the values referenced by the `final` value might be modifiable.
   (Same as JavaScript `const` values and Java `final` values.)

Types with property `{ref_modifiable: false}` are called immutable types.
The notation `T.fixed` is introduced to represent the immutable version of mutable type `T`,
where `fixed` is a new introduced keyword.
Please note the semantics of **immutable type** in this proposal is different from many other immutable type proposals.
A value of type `T.fixed` may be modifiable, it is just that the values referenced (either directly or indirectly)
by the `T.fixed` value can't be modified.

Below, for description convenience, the proposal will call
* `T` values declared with `var` as `var.mutable` values.
* `T` values declared with `final` as `final.mutable` values.
* `T.fixed` values declared with `var` as `var.fixed` values.
* `T.fixed` values declared with `final` as `final.fixed` values.

Please note that,
* the notation `[]*chan T.fixed` can only mean `([]*chan T).fixed`,
  whereas `[]*chan (T.fixed)`, `[]*((chan T).fixed)` and `[]((*chan T).fixed)` are all invalid notations.
* `fixed` is not allowed to appear in type declarations. `type T []int.fixed` is invalid.
* the respective immutable types of no-reference types (including basic types, struct types with only fields
  of no-reference types and array types with no-reference element types) are the mutable types themselves.

A notation `v.fixed` is introduced to convert a value `v` of type `T` to a `T.fixed` value.
Note, similar to the immutable type notation, `.fixed` must be the last portion in an expression.

The **basic assignment/binding rules**:
1. `*.mutable` values can be bound/assigned to a `*.mutable` value.
1. **A `final.*` value must be bound a value in its declaration**.
   After the declaration, it can never be assigned any more.
1. Values of any genres can be bound/assigned to a `*.fixed` value, including constants, literals, variables,
   and the new supported values by this proposal.
1. Generally, `*.fixed` values can't be bound/assigned to a `*.mutable` value, with one exception:
   `*.fixed` values of no-reference types will be viewed as be viewed as `*.mutable` values (and vice versa) when
   they are used as source values in assignments. (Maybe function types should be also viewed as no-reference types.)
   In other words, a value of any genre can be assigned to another value of any genre,
   if the two values are values of no-reference types, as long as they satisfy other old assignment requirements.

Please note that, although a value **can't be modified through `*.fixed` values which are referencing it**, it
**might be modified through other `*.mutable` values which are referencing it**. (Yes, this proposal doesn't solve all problems.)
In other words, most of the rules in this proposal are enfored by compilers, not runtimes.

The section to the next will list the detailed rules for values of all kinds of types.
Those rules are much straightforward and anticipated.
**They are derived from the above mentioned basic assignment/binding rules.**

Please note, the immutability semantics in this proposal is different from the `const` semantics in C/C++.
For example, a value declared as `var p ***int.fixed` in this proposal is
like a variable decalared as `int const * const * const * p` in C/C++.
In C/C++, we can declare a variable as `int * const * const * x`,
but there are no ways to declare variables with the similar immutabilities in this proposal.
(In other words, this proposal assumes such use cases are rare in practice.)

Another example, the following C code are valid.
```C
#include <stdio.h>

typedef struct T {
	int* y;
} T;

void main() {
	int a = 123;
	T t = {.y = &a};
	const T* p = &t; // <=> T const * p = &t;
	*p->y = 789; // allowed
	printf("%d\n", *t.y); // 789
}
```

But, the following similar Go code is invalid by this proposal.

```golang
package main

type T struct{
	y *int
}

func main() {
	var a int = 123
	var t = T{y: &a}
	var p *T.fixed = &t; // a value with property:
	                     // {self_modifiable: true, ref_modifiable: false}
	*p.y = 789;  // NOT allowed,
	             // for all values referenced by p, either
	             // directly or indirectly, are unmodifiable.
	println(*t.y);
}
```

### Syntax changes

It is a challenge to design a both simple and readable syntax set for this proposal.
The current design may be not perfect, so any improvemnt ideas are welcome.

Some examples of the full value declaration form:
```golang
final FileNotExist = errors.New("file not exist").fixed // a totally immutable value
final FileNotExist .fixed = errors.New("file not exist")  // equivalent to the above line

// The following declarations are equivalent.
var a []int.fixed = []int{1, 2, 3}
var a .fixed = []int{1, 2, 3}
var a = []int{1, 2, 3}.fixed

// The following declarations are equivalent (for no-reference types only).
var b int
var b int.fixed

// Declare variables in a hybrid way.

// x is a var.fixed value, y is a var.mutable value.
var x, y = []int{1, 2}.fixed, []int{1, 2}
// z is a final.mutable value, w is a final.fixed value.
final z, w []int = []int{1, 2}, []int{1, 2}.fixed
```

Immutable parameter and result declaration examples:
```golang
func Foo(m http.Request.fixed, n map[string]int.fixed) (o []int.fixed, p chan int.fixed) {...}
func Print(values ...interface{}.fixed) {...}
```
All parameters and results in the above example are `var.fixed` values.
To avoid syntax design complexity, `final.*` parameters and results are not supported.

Short value declaration examples:
```golang
{
	oldA, newB := va, vb.fixed
	oldA, newB := va, (.fixed)(vb) // equivalent to the above line

	newX, oldY := (Tx.fixed)(va), vy
	newX, oldY := (Tx(va)).fixed, vy
	newX, oldY := Tx(va.fixed), vy
	newX, oldY := Tx(va).fixed, vy // equivalent to the above three lines
}
```

Again, to avoid syntax design complexity, `final.*` values can't be declared in short declartions.
In other words, values declared in short declarations are always `var.*` values.

### Detailed rules of this proposal

#### safe pointers

* Dereferences of `*.fixed` pointers are `final.fixed` values.
* Dereferences of `*.mutable` pointers are `var.mutable` values.
* Addresses of addressable `final.*` and `*.fixed` values are `var.fixed` pointer values.
  Some certain write permissions are lost when taking addresses of addressable `final.mutable` and `var.fixed` values.

Yes, `final.*` values may be addressable.

#### unsafe pointers

* Dereferences of an unsafe pointer are always `var.mutable` values,
even if the unsafe pointer is a `*.fixed` value.
(This is important for reflection implementation.)

#### structs

* Fields of `var.fixed` struct values are `var.fixed` values.
* Fields of `var.mutable` struct values are `var.mutable` values.
* Fields of `final.fixed` struct values are `final.fixed` values.
* Fields of `final.mutable` struct values are `final.mutable` values.

#### arrays

* Elements of `var.fixed` array values are `var.fixed` values.
* Elements of `var.mutable` array values are `var.mutable` values.
* Elements of `final.fixed` array values are `final.fixed` values.
* Elements of `final.mutable` array values are `final.mutable` values.

#### slices

* Elements of `*.fixed` slice values are `final.fixed` values.
* Elements of `*.mutable` slice values are `var.mutable` values.
* We can't append elements to `*.fixed` slice values.
* Subslice:
  * The subslice result of a `final.fixed` slice is still a `final.fixed` slice.
  * The subslice result of a `final.mutable` slice is still a `final.mutable` slice.
  * The subslice result of a `var.fixed` slice is still a `var.fixed` slice.
  * The subslice result of a `var.mutable` slice or array is a `var.mutable` slice.
  * The subslice result of a `final.*` or `*.fixed array is a `var.fixed` slice.
    Some certain write permmisions may be lost in the process.

#### maps

* Elements of `*.fixed` map values are `final.fixed` values.
* Elements of `*.mutable` map values are `var.mutable` values.
* We can't append new entries to (or replace entries of,
  or delete old entries from) `*.fixed`  map values.

#### channels

* Send
  * We can only send `*.mutable` values to a `*.mutable` channel.
  * We can send values of any genres to a `*.fixed` channel.
* Receive
  * Receiving from a `*.mutable` channel results a `*.mutable` value. (It is not important whether or not the result itself can be modified.)
  * Receiving from a `*.fixed` channel results a `*.fixed` value. (It is not important whether or not the result itself can be modified.)

#### functions

Function parameters and results can be declared with property `{ref_modifiable: false}`.

In the following function proptotype, parameter `x` and result `w` are viewed as being declared as `var.fixed` values.
```golang
func fa(x Tx.fixed, y Ty) (z Tz, w Tw.fixed) {...}
```

A `func()(T)` value is assignable to a `func()(T.fixed)` value, not vice versa.

A `func(T.fixed)` value is assignable to a `func(T)` value, not vice versa.

#### method sets

The method set of type `T.fixed` is always a subset of type `T`.
This means when a method `M` is explicitly declared for type `T.fixed`,
then a corresponding implicit method with the same name will be declared for type `T` by compilers.
```golang
func (T.fixed) M() {} // explicitly declared
/*
func (t T) M() {t.fixed.M()} // implicitly declared
*/

var t T
t.M()
// <=>
t.fixed.M()
// <=>
T.fixed.M(t.fixed)
// <=>
T.fixed.M(t)
// <=>
T.M(t)
```

In the above code snippet, the method set of type `T.fixed` contains one method: `fixed.M`,
however the method set of type `T` contains two method: `fixed.M` and `M`.

For type `T` and `*T`, if methods can be declared for them (either explicitly or implicitly),
the method set of type `T.fixed` is a subset of type `*T.fixed`.
(Or in other words, the method set of type `T` is a subset of type `*T`
if type `T` is not an interface type.)

#### interfaces

An interface type can specify some immutable methods. For example:
```golang
type I interface {
	M0(Ta) Tb // a mutable method

	fixed.M2(Tx) Ty // an immutable method
	                // NOTE: this is an exported method.
}
```

Similar to non-interface type, if a mutable interface type explicitly specified an immutable method `fixed.M`,
it also implicitly specifies a mutable method with the same name `M`.

The method set specified by type `I` contains three methods, `M0`, `fixed.M2` and `M2`.
The method set specified by type `I.fixed` only contains one method, `fixed.M2`.

When a method is declared for a concrete type to implement an immutable method,
the type of the receiver of the declared method must be immutable (a.k.a., a `var.fixed` value).
For example, in the following code snippet, the type `T1` implements the interface `I` shown in the above code snippet, but the type `T2` doesn't.
```golang
type T1 struct{}
func (T1) M0(Ta) Tb {var b Tb; return b}
func (T1.fixed) M2(Tx) Ty {var y Ty; return y} // the receiver type is immutable.

type T2 struct{}
func (T2) M0(Ta) Tb {var b Tb; return b}
func (T2) M2(Tx) Ty {var y Ty; return y} // the receiver type is mutable.
```

If a mutable type `T` implements a mutable interface type `I`, then the immutable type `T.fixed` also implements the immutable interface type `I.fixed`.

* Dynamic type
  * The dynamic type of a `*.mutable` interface value is a mutable type.
  * The dynamic type of a `*.fixed` interface value is an immutable type.
* Box
  * No values can be boxed into `final.*` interface values (except the initial bound values).
  * `*.fixed` values can't be boxed into `var.mutable` interface values.
  * Values of any genres can be boxed into a `var.fixed` interface value.
* Assert
  * A type assertion on a `*.fixed` interface value results a `*.fixed` value. (It is not important whether or not the result itself can be modified.)
    For such an assertion, its syntax form `x.(T.fixed)` can be simplified as `x.(T)`.
  * A type assertion on a `*.mutable` interface value results a `*.mutable` value. (It is not important whether or not the result itself can be modified.)

For this reason, the `xyz ...interface{}` parameter declarations of all the print functions
in the `fmt` standard package should be changed to `xyz ...interface{}.fixed` instead.

#### reflection

Many function and method implementations in the `refect` package should be modified accordingly.
The `refect.Value` type shoud have an **_fixed_** property,
and the result of an `Elem` method call should inherit the **_fixed_** property
from the receiver argument. More about reflection.
For all details on reflection, please read the following reflection section.

### Usage examples

```golang
var x = []int{1, 2, 3}
var y [][]int.fixed
y = [][]int{x, x} // ok

x[1] = 123    // ok
y[0][1] = 123 // error, for y is a var.fixed value.
var z = y[0]  // ok, z is also a var.fixed value.
z[0] = 123    // error

p := &z[0]     // ok. p is a var.fixed value.
*p = 123       // error
x[0] = *p      // ok
p = new(int)   // ok

var v interface{} = y       // error
var v interface{}.fixed = y // ok
var w = v.([][]int)         // ok, w is a var.fixed value
v = x                       // ok
var u = v.fixed             // ok, u is a var.fixed value

// S is exported, but external packages have
// no ways to modify x and S (through S).
final S = x.fixed   // ok.
S = x               // error
t := S[:]           // ok, t is a var.fixed value. S[:] is a final.fixed value.
_ = append(t, 4)    // error

// The elements of R even can't be modified in current package!
final R = []int{7, 8, 9}.fixed
R[1] = 888 // error

// Q can't be modified, but its elements can.
final Q = []int{7, 8, 9}
Q[1]= 888 // ok
```

Another one:
```golang
var s = "hello word"
var bytes = []byte.fixed(s) // a clever compiler will not allocate a
                            // duplicate underlying byte sequence here.
{
	pw := &s[6] // pw is a `var.fixed` value of built-in type "byte".
}
```

### Compiler implementation

I'm not familiar with the compiler development things.
It is just my feeling, by my experience, that the rules mentioned in this proposal
can be enforced by compiler without big technology obstacles.

At compile phase, compiler should maintain two bits for each value.
One bit means whether or not the value itself can be modified.
The other bit means whether or not the values referenced by the value can be modified.

### New reflection functions and methods and how to implement them

`reflect.Value` values can only representing `var.*` Go values.

A `reflect.FixedValueOf` function is needed to create `reflect.Value` values representing `var.fixed` Go values.
Its prototype is
```golang
func FixedValueOf(i interface{}.fixed) Value
```

In implementaion, one bit should be borrowed from the 23+ bits method number to represent the `fixed` proeprty.

All parameters of type `reflect.Value` of the functions and methods in the `reflect` package,
including receiver parameters, should be declared as `var.fixed` values.
However, the `reflect.Value` return results should be declared as `var.mutable` values.

A `reflect.Value.ToFixed` method is needed to make a `reflect.Value` value represent a `var.fixed` Go value.

A `reflect.Value.FixedInterface` method is needed, it returns a `var.fixed` interface value.
The old `Interface` method panics on `var.fixed` values.

A method `reflect.Type.Fixed` is needed to get the immutable version of a mutable type.
A method `reflect.Type.Mutable` is needed to get the mutable version of an immutable type.
The method sets of mutable type `T` and immutable type `T.fixed` may be different if `T` is not an interface type.
Their respective other properties should be identical.

A method `reflect.Type.Genre` is needed, it may return `Fixed` or `Mutable`.

### Unsolved and new problems of this proposal

This proposal doesn't guarantee some values referenced by `*.fixed` values will never be modified.
(This is more a feature than a problem.)

This proposal will make `bytes.TrimXXX` (and some others) functions need some duplicate versions
for mutable and immutable arguments.
This problem should be solved by future possible generics feature.

### Go 1 incompatible cases

The followings are the incompatible cases I'm aware of now.
1. `final` and `fixed` may be used as non-exported identifiers in old user code.
   It should be easy for the `go fix` command to modify these uses to others.
   (Using `const` to replace `final` and `fixed` can avoid this incompatible case, but may cause some confusions.)

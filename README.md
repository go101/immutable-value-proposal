# A Go immutable values proposal

Old versions:
* [the propsoal thread](https://github.com/golang/go/issues/29422).
* [the `var:N` version](README-v1.md)

The new syntax is not Go 1 compatible,
please read the last section of this proposal for incompatible cases.

To avoid syntax design complexity, the new proposal doesn't support declaring
function parameters and results with property `{self_modifiable: false}` (see below).

Any criticisms and improvement ideas are welcome, for
* I have not much compiler-related knowledge, so the following designs may have flaws.
* I haven't found a perfect syntax notation set for this proposal yet.

### The problems this proposal tries to solve

The problems this proposal tries to solve:
1. no ways to declare package-level immutable non-basic variables.
1. no ways to declare immutable function parameters and results.

Please note, the immutability semantics in this proposal is different
from either the `const` values in C/C++ or in JavaScript.
The following sections will explain the differences.

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

This proposal will let Go support the two value genres the current Go doesn't support,
and extend the range of `{self_modifiable: false, ref_modifiable: true}` values,
by introducing a keyword, `fixed`.
* `{self_modifiable: false, ref_modifiable: false}` values are declared with `fixed.fixed`.
  For example, the error values of many std package should be declared  with `fixed.fixed`.
* `{self_modifiable: true, ref_modifiable: false}` values are declared with `var.fixed`.
  For example, the parameters of a function which will not be modified within the function should be declared with `var.fixed`.
* `{self_modifiable: false, ref_modifiable: true}` values are declared with `fixed.var`.
* The current supported variables are declard with `var.var`, which can be simplified as `var`.

**A `fixed.*` value must be bound a value in its declaration**.
After the declaration, it can never be assigned any more.

Any value can be bound/assigned to a `*.fixed` value,
including constants, literals, variables, and the new supported values by this propsoal.

`*.fixed` values can't be bound/assigned to a `*.var` value. However,
`var.var` values of no-reference types (inclunding basic types, struct types with only fields of no-reference types
and array type with no-reference element types) will be viewed as be viewed as `*.fixed` values when they are used
as source values in assignments. (Maybe function types should be also viewed as no-reference types.)

Please note that, although a value **can't be modified through `*.fixed` values which are referencing it**,
it **can be modified through other `*.var` values which are referencing it**. (Yes, this proposal doesn't solve all problems.)

The above listed rules in this section are the basic rules of this proposal.

Please note, the immutability semantics in this proposal is different from the `const` semantics in C/C++.
In C/C++, `const` is a type qualifier, however, immutability is a value property of Go.
For example, a value declared as `var.fixed p ***int` is like a variable decalared as `int const * const * const * p` in C/C++.
In C/C++, we can declare a variable as `int * const * const * x`, in Go, no ways to declare variables with the similar immutabilities.

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

But, the following similar Go code is invalid.

```golang
package main

type T struct{
	y *int
}

func main() {
	var a int = 123
	var t = T{y: &a}
	var.fxied p *T = &t; // a value with property:
	                     // {self_modifiable: true, ref_modifiable: false}
	*p.y = 789;  // NOT allowed,
	             // for all values referenced by p,
	             // either directly or indirectly,
	             // are not modifiable.
	println(*t.y);
}
```

The section to the next will list the detailed rules for values of all kinds of types.
Those rules are much straightforward and anticipated.
**They are derived from the basic rules.**

### Syntax changes

It is a challenge to design a both simple and readable syntax set for this proposal.
The current design may be not perfect, so any improvemnt ideas are welcome.

Some examples of the full variable declaration form:
```golang
fixed.fixed FileNotExist = errors.New("file not exist") // a totally immutable value

// The following declarations are equivalent.
var.fixed a, b, c []int
var.? a, b, c []int.fixed
var.? a, b, c = []int(nil).(fixed), []int(nil).(fixed), []int(nil).(fixed)
var.? a, b, c []int = nil.(fixed), nil.(fixed), nil.(fixed)

// The following declarations are equivalent (for no-reference types only).
var a, b, c int
var.var a, b, c int
var.fixed a, b, c int
var.? a, b, c int.fixed

// Declare variables in a hybrid way.
var.? x, y = []int{}.(fixed), []int{} // x is a var.fixed value, y is a var.var value.
fixed.? z, w []int = nil, nil.(fixed) // z is a fixed.var value, w is a fixed.fixed value.
```

Immutable parameter and result declaration examples:
```golang
func Foo(m http.Request.fixed, n map[string]int.fixed) (o []int.fixed, p chan int.fixed) {...}
func Print(values ...interface{}.fixed) {...}
```
All parameters and results in the above example are `var.fixed` values.
As above has mentioned, to avoid syntax design complexity, `fixed.*` parameters and results are not supported.

Short value declaration examples:
```golang
{
	newA, newB, oldC := va.(fixed), vb, vc
	newA, newB, oldC := va.(fixed), vb, vc // equivalent to the above line
	
	newX, newY, oldZ := (Tx.fixed)(va), (Ty)(vb), vc
	newX, newY, oldZ := (Tx)(va).(fixed), (Ty)(vb), vc // equivalent to the above line
}
```

For the same reason (to avoid syntax design complexity), `fixed.*` values can't be declared in short declarations.

### Detailed rules of this proposal

#### safe pointers

* Dereferences of `*.fixed` pointers are `fixed.fixed` values.
* Dereferences of `*.var` pointers are `var.var` values.
* Addresses of addressable `fixed.*` and `*.fixed` values are `var.fixed` pointer values.
  Some certain write permissions are lost when taking addresses of addressable `fixed.var` and `var.fixed` values.

#### unsafe pointers

* Dereferences of an unsafe pointer are always `var.var` values,
even if the unsafe pointer is a `*.fixed` value.
(This is important for refection implementation.)

#### structs

* Fields of `var.fixed` struct values are `var.fixed` values.
* Fields of `fixed.fixed` struct values are `fixed.fixed` values.
* Fields of `fixed.var` struct values are `fixed.var` values.

#### arrays

* Elements of `var.fixed` array values are `var.fixed` values.
* Elements of `fixed.fixed` array values are `fixed.fixed` values.
* Elements of `fixed.var` array values are `fixed.var` values.

#### slices

* Elements of `*.fixed` slice values are `fixed.fixed` values.
* Elements of `*.var` slice values are `var.var` values.
* We can't append elements to `fixed.*` and `*.fixed` slice values.
* Subslice:
  * The subslice result of a `fixed.fixed` slice is still a `fixed.fixed` slice.
  * The subslice result of a `fixed.var` slice is still a `fixed.var` slice.
  * The subslice result of a `var.fixed` slice is still a `var.fixed` slice.

#### maps

* Elements of `*.fixed` map values are `fixed.fixed` values.
* Elements of `*.var` map values are `var.var` values.
* We can't append new entries to (or replace entries of,
  or delete old entries from) `*.fixed`  map values.

#### channels

Channel rules are a little special.

* Send
  * We can send any values to a `*.var` channel.
  * We can only send `*.fixed` values to a `*.fixed` channel. (The speciality.)
* Receive
  * Receiving from a `*.var` channel results a `*.var` value. (It is not important whether of not the result itself can be modified.)
  * Receiving from a `*.fixed` channel results a `*.fixed` value. (It is not important whether of not the result itself can be modified.)

#### functions

Function parameters and results can be declared with property `{ref_modifiable: false}`.

In the following function proptotype, parameter `x` and result `w` are viewed as being declared with `var.fixed`.
```golang
func fa(x Tx.fixed, y Ty) (z Tz, w Tw.fixed) {...}
```

A `func(T.fixed)` value is assignable to a `func(T)` value, not vice versa.
A `func()(T)` value is assignable to a `func()(T.fixed)` value, not vice versa.

#### method sets

Every type has **two method sets**, one for `var.var` receiver values, one for `var.fixed` receiver values.
The `var.fixed` one is a subset of the `var.var` one.
For type `T` and `*T`, if methods can be declared for them (either explicitly or implicitly),
then the `var.fixed` method set of `T` is a subset of the `var.fixed` method set of `*T`.

#### interfaces

* Box
  * No values can be boxed into `fixed.*` interface values.
  * `*.fixed` values can't be boxed into `var.var` interface values.
  * Any values can be boxed into `var.fixed` interface values (**as long as the `var.fixed` method set of their respective types implement the interface**).
* Assert
  * A type assertion on `*.fixed` interface value results an `*.fixed` value. (It is not important whether of not the result itself can be modified.)
  * A type assertion on `*.var` interface value results an `*.var` value. (It is not important whether of not the result itself can be modified.)

For this reason, the `xyz ...interface{}` parameter declarations of all the print functions
in the `fmt` standard package should be changed to `xyz ...interface{}.fixed` instead.

#### reflection

Many function and method implementations in the `refect` package should be modified accordingly.
The `refect.Value` type shoud have an **_fixed_** property,
and the result of an `Elem` method call should inherit the **_fixed_** property
from the receiver argument. More about reflection.
For all deails on reflection, please read the following reflection section.

### Usage examples

```golang
var x = []int{1, 2, 3}
var.fixed y [][]int
y = [][]int{x, x} // ok

x[1] = 123         // ok
y[0][1] = 123      // error
var z = y[0]       // error
var.fixed z = y[0] // ok
z[0] = 123         // error

// The following line <=> var.fixed p = &z[0]
p := &z[0]     // ok. p is an immutable value.
*p = 123       // error
x[0] = *p      // ok
p = new(int)   // ok

var.fixed v interface{} = y
var w = v.([][]int)       // error
var.fixed w = v.([][]int) // ok
v = x                     // ok

// S is exported, but external packages have
// no ways to modify x and S (through S).
fixed.fixed S = x // ok.
S = x             // error
t := S[:]         // ok. <=> var t = S[:].(fixed) <=> var.fixed t = S[:]
_ = append(t, 4)  // error

// The elements of R even can't be modified in current package!
fixed.fixed R = []int{7, 8, 9}

// Q can't be modified, but its elements can.
fixed.var Q = []int{7, 8, 9}
```

Another one:
```golang
var s = "hello word"
var.fixed bytes = []byte(s) // a clever compiler will not allocate a
                            // deplicate underlying byte sequence here.
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

A `reflect.FixedValueOf` function is needed to create `reflect.Value` values representing `var.fixed` Go values.
Its prototype is
```golang
func FixedValueOf(i interface{}.fixed) Value
```

`reflect.Value` values can only representing `var.*` values.

All parameters of type `reflect.Value` of the functions and methods in the `reflect` package,
including receiver parameters, should be declared as `var.fixed` values.
However, the `reflect.Value` return results should be declared as `var.var` values.

A `reflect.Value.ToFixed` method is needed to convert a Value to a `var.fixed` one.

A `reflect.Value.FixedInterface` method is needed, it returns a `var.fixed` interface value.
The old `Interface` method panics on `var.var` values.

Three methods `reflect.Type.NumFixedMethods`, `reflect.Type.FixedMethodByName` and `reflect.Type.FixedMethod` are needed.

In implementaion, one bit should be borrowed from the 23+ bits method number to represent the `fixed` proeprty.

### Go 1 incompatible cases

For now, we can use `fixed` and `fixed.fixed` to declare values, this proposal is not Go 1 compatible.

Another migh-be-ambiguity case:
assume a source file imports a package as `T` and if there is a type named `fixed` in the imported package,
although a smart compiler will not mistake the `fixed` in `T.fixed` as a keyword, the `T.fixed` really hurts code readibilty.

Using the old `const` keyword instead of the new `fixed` keyword can avoid these problems, but will cause `const` pollution problem.


# A Go immutable types/values proposal

Old versions:
* [the propsoal thread](https://github.com/golang/go/issues/29422).
* [the `var:N` version](README-v1.md)
* [the pure-immutable-value interpretation version](README-v2.md)

This proposal introduces a notation `T.fixed` to represent the immutable version of type `T`,
where `fixed` is new introduced keyword, which makes this proposal not Go 1 compatible.
Please read the last section of this proposal for incompatible cases.

In fact, we can use the old `const` keyword to replace the `fixed` keyword to make this prorposal Go 1 compatible.
However, personally I think, for this specified proposal, the readibitly of `const` is not good as `fixed`,
though I feel `T.const` is also acceptable.

To avoid syntax design complexity, the new proposal doesn't support declaring
function parameters and results with property `{self_modifiable: false}` (see below).

Any criticisms and improvement ideas are welcome, for
* I have not much compiler-related knowledge, so the following designs may have flaws.
* I haven't found a perfect syntax notation set for this proposal yet.

### The problems this proposal tries to solve

The problems this proposal tries to solve:
1. no ways to declare package-level immutable non-basic values.
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

The notation `T.fixed` is introduced to represent the immutable version of type `T`.
However, please note the semantics of **immutable type** in this proposal is different to many other immutable type proposals.
A value of type `T.fixed` may be modifiable, it is just that the values referenced by the `T.fixed` value can't be modified.
In othe words, values of type `T.fixed` can be either `var.fixed` values or `fixed.fixed` values.

Please note that, `[]*chan T.fixed` can only mean `([]*chan T).fixed`.
Whereas `[]*chan (T.fixed)`, `[]*((chan T).fixed)` and `[]((*chan T).fixed)` are invalid notations.

A notation `v.(fixed)` is introduced to convert a value `v` to a `*.fixed` value.
The notation is called **immutability assertion**.
If `v` is a non-interface values, `v.(fixed)` will always succeed.

**A `fixed.*` value must be bound a value in its declaration**.
After the declaration, it can never be assigned any more.

Generally, any value can be bound/assigned to a `*.fixed` value, including constants, literals, variables,
and the new supported values by this propsoal, with one exception: `*.var` interface values can't be assigned
to `*.fixed` interface values. A `*.var` interface value can only be immutability asserted to a `*.fixed` interface value.
(Please view the interface related rules section below for details.)

Generally, `*.fixed` values can't be bound/assigned to a `*.var` value, with one exception:
`var.var` values of no-reference types (inclunding basic types, struct types with only fields of no-reference types
and array type with no-reference element types) will be viewed as be viewed as `*.fixed` values when they are used
as source values in assignments. (Maybe function types should be also viewed as no-reference types.)

Please note that, although a value **can't be modified through `*.fixed` values which are referencing it**, it
**might be modified through other `*.var` values which are referencing it**. (Yes, this proposal doesn't solve all problems.)

The above listed rules in this section are the basic rules of this proposal.

Please note, the immutability semantics in this proposal is different from the `const` semantics in C/C++.
For example, a value declared as `var.fixed p ***int` in this proposal is
like a variable decalared as `int const * const * const * p` in C/C++.
In C/C++, we can declare a variable as `int * const * const * x`, 
in this proposal, no ways to declare variables with the similar immutabilities.

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
	var.fxied p *T = &t; // a value with property:
	                     // {self_modifiable: true, ref_modifiable: false}
	*p.y = 789;  // NOT allowed,
	             // for all values referenced by p, either
	             // directly or indirectly, are unmodifiable.
	println(*t.y);
}
```

The section to the next will list the detailed rules for values of all kinds of types.
Those rules are much straightforward and anticipated.
**They are derived from the above mentioned basic rules.**

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
	newA, newB, oldC := (var.fixed)(va), vb, vc
	newA, newB, oldC := (?.fixed)(va), vb, vc
	newA, newB, oldC := va.(fixed), vb, vc // equivalent to the above two lines
	
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
  * Receiving from a `*.var` channel results a `*.var` value. (It is not important whether or not the result itself can be modified.)
  * Receiving from a `*.fixed` channel results a `*.fixed` value. (It is not important whether or not the result itself can be modified.)

#### functions

Function parameters and results can be declared with property `{ref_modifiable: false}`.

In the following function proptotype, parameter `x` and result `w` are viewed as being declared with `var.fixed`.
```golang
func fa(x Tx.fixed, y Ty) (z Tz, w Tw.fixed) {...}
```

A `func(T.fixed)` value is assignable to a `func(T)` value, not vice versa.
A `func()(T)` value is assignable to a `func()(T.fixed)` value, not vice versa.

#### method sets

The method set of type `T.fixed` is a subset of type `T`.
If `T` is an interface type, then the method sets of `T.fixed` and `T` are always identical.

For type `T` and `*T`, if methods can be declared for them (either explicitly or implicitly),
the method set of type `T.fixed` is a subset of type `*T.fixed`.
(Or in other words, the method set of type `T` is a subset of type `*T`
if type `T` is not an interface type.)

#### interfaces

* Dynamic type
  * The dynamic type of a `*.var` interface value is a mutable type.
  * The dynamic type of a `*.fixed` interface value is an immutable type.
* Box
  * No values can be boxed into `fixed.*` interface values.
  * `*.fixed` values can't be boxed into `var.var` interface values.
  * Any value can be boxed into a `var.fixed` interface value
  (_as long as the method set of `T.fixed` implements the type of the interface value_,
  where `T` is the corresponding mutable type of the value to be boxed).
* Assert
  * A type assertion on a `*.fixed` interface value results a `*.fixed` value. (It is not important whether or not the result itself can be modified.)
    For such an assertion, its syntax form `x.(T.fixed)` can be simplified as `x.(T)`.
  * A type assertion on a `*.var` interface value results a `*.var` value. (It is not important whether or not the result itself can be modified.)
  * An immutability assertion on a `*.var` interface value results a `*.fixed` value. (It is not important whether or not the result itself can be modified.)
    Such an assertion fails if the immutable version of the dynamic type of the interface value doesn't implement the type of the interface value.

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

`reflect.Value` values can only representing `var.*` Go values.

A `reflect.FixedValueOf` function is needed to create `reflect.Value` values representing `var.fixed` Go values.
Its prototype is
```golang
func FixedValueOf(i interface{}.fixed) Value
```

In implementaion, one bit should be borrowed from the 23+ bits method number to represent the `fixed` proeprty.

All parameters of type `reflect.Value` of the functions and methods in the `reflect` package,
including receiver parameters, should be declared as `var.fixed` values.
However, the `reflect.Value` return results should be declared as `var.var` values.

A `reflect.Value.ToFixed` method is needed to convert a Value to a `var.fixed` one.

A `reflect.Value.FixedInterface` method is needed, it returns a `var.fixed` interface value.
The old `Interface` method panics on `var.var` values.

A method `reflect.Type.Fixed` is needed to get the immutable version of a type.

### Go 1 incompatible cases

The new keyword `fixed` is one cause why this proposal is not Go 1 compatible.
Assume a source file imports a package as `T` and if there is a type named `fixed` in the imported package,
although a smart compiler will not mistake the `fixed` in `T.fixed` as a keyword, the `T.fixed` really hurts code readibilty.
 
Using the old `const` keyword instead of the new `fixed` keyword can avoid these problems,
however it would make people be confused with the current constant things.
(Maybe, it is an acceptable solution.)

Another incompatible case is caused by the fact that `*.var` interface value can't be assigned `*.fixed` interface values.
When the parameters of a function, such as the `fmt.Print` function, are changed to immutable types,
then some old user code will fail to compile.
But it should be easy for the `go fix` command to modify the corresponding arguments to immutability assertions.

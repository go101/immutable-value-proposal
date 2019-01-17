# A Go immutable values proposal

This proposal a re-written based on [an old one](https://github.com/golang/go/issues/29422).
The old proposal thread contains too many immature ideas to act as a formal proposal.
The new one looks much more mature and clearner.

There is also an important different from the old propsoal.
This new one uses `var:1` and `var:0`, instead of `fixed` and `fixed!` in the old one,
to declare immutable variables, to try to be Go 1 compatible.

Any criticisms and improvement ideas are welcome, for
* I have not much compiler-related knowledge, so the following designs may have flaws.
* I haven't found a perfect syntax notation set for this proposal yet.

### The problems this proposal tries to solve

The problems this proposal tries to solve:
1. no ways to declare package-level immutable non-basic variables.
1. no ways to declare immutable function parameters and results.

Please note, the immutability semantics in this proposal is different
from either the `const` values in C/C++ or in JavaScript.
The following sections will explain the difference.

### The main points of this proposal

We know each value has a property, `self_modifiable`, which means whether or not that value is modifiable.

This proposal will add a new value property `ref_modifiable` for each value, which means
whether or not the values referenced (either directly or indirectly) by that value are modifiable.

The permutation of thw two properties result 4 genres of values:
1. `{self_modifiable: true, ref_modifiable: false}`.
   No such Go values currently.
   We call values of this genre as **ref-immutable values**.
1. `{self_modifiable: true, ref_modifiable: true}`.
   Such as variables.
1. `{self_modifiable: false, ref_modifiable: false}`.
   No such Go values currently.
   We call values of this genre as **both-immutable values**.
1. `{self_modifiable: false, ref_modifiable: true}`.
   Such as composite literals.
   (In fact, all declared constants in JavaScript and all final variables decalred in Java belong to this genre.)

This proposal will let Go support the two value genres the current Go doesn't support.
* Both-immutable values are declared with `var:0`.
  For example, the error values of many std package should be declared as both-immutable values.
* Ref-immutable values are declared with `var:1`.
  For example, the parameters of a function which will not be modified within the function should be declared as ref-immutable.

Please note, the number `1` and `0` means **mutable depth**.

**A both-immutable value must be bound a value in its declaration**.
After the declaration, it can never be assigned any more.
Values of any genre can be bound to a both-immutable value,
including constants, variable, literal, ref-immutable, or another both-immutable value.

A ref-immutable value can be declared without an initial value.
Same as both-immutable values, values of any genre can be assigned to a ref-immutable value,
**including both-immutable values**.

When **_an immutable value_** is mentioned below, it means a both-immutable value or a ref-immutable value.

Please note that, although a value **can't be modified through the (either both- or ref-) immutable values which are referencing it**,
it **can be modified through other mutable values which are referencing it**. (Yes, this proposal doesn't solve all problems.)

There are three different design ideas for immutable-to-mutable assignments:
1. **any such assignments are disallowed**. (The recommended design.)
1. if the concrete type of the source value is a basic type (or any non-referencing type), then such an assignment is allowed.
   In other words, values of such types declard with `var` should be treated as they are declared with `var:1`.
   The reason is the *ref_modifiable* property is meaningless for values of such types.
1. nilify all referencing in source values in such assignments.

The second and third designs may be useful for some cases but they are easy to cause many confusions. (*Need to be proved.*)

This following assume the first design idea for immutable-to-mutable assignments is adopted.

The above listed in this section are the basic rules of this proposal.

Please note, the immutability semantics in this proposal is different from the `const` semantics in C/C++.
In C/C++, `const` is a type qualifier, however, immutability is a value property of Go.
For example, a variable declared as `var:1 p ***T` is like a variable decalared as `T const * const * const * p` in C/C++.
In C/C++, we can a variable as `T * const * const * x`, in Go, no ways to declare variables with the similar immutabilities.

For example, the following C code are valid.
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
	var:1 p *T = &t; // a ref-immutable variable
	*p.y = 789;  // NOT allowed
	             // All values referenced by p,
	             // either directly or indirectly,
	             // are not modifiable.
	println(*t.y);
}
```

The section to the next will list the detailed rules for values of all kinds of types.
Those rules are much straightforward and anticipated.
**They are derived from the basic rules.**

### Syntax changes

This propsoal tries to make the new features be Go 1 compitable,
which really brings a lot of challenges to the syntax design.
The current design may be not perfect, so any improvemnt ideas are welcome.

Some examples of the full variable declaration form:
```golang
var:0 FileNotExist = errors.New("file not exist") // a both immutable variable

// The following three declaration are equivalent.
var:1 a, b, c int
var a, b, c 1:int
var 1:a, 1:b, 1:c int

// Declare variables in a hybrid way.
var 1:x, 0:y, z = true, 789, "hello"
```

Immutable parameter and result declaration examples:
```golang
func Foo(m 1:http.Request, n 1:map[string]int) (o 1:[]int, p 1:chan int) {...}
func Print(values ...1:interface{}) {...}
```

Short variable declaration examples:
```golang
{
	newA, newB, oldC := (var:1)(va), (var:0)(vb), vc
	newA, newB, oldC := (:1)(va), (:0)(vb), vc // equivalent to the above line
	newX, newY, oldZ := (1:Tx)(vx), (0:Ty)(vy), vz
}
```

I do have another idea by merging [the idea](https://github.com/golang/go/issues/377#issuecomment-66049402) into this proposal,
but which will make this proposal Go 1 imcompatible. An example:
```golang
{
	// NOTE: the assignment sign is "=" instead of ":=" here.
	1:newA, 0:newB, oldC = va, vb, vc
	1:newX, 0:newY, oldZ := Tx(vx), Ty(vy), vz
}
```

### Detailed rules of this proposal

#### safe pointers

Dereferences of immutable pointers are both-immutable values.
An immutable value may be addressable.
Addressable immutable values can be taken addresses.
Their addresses are ref-immutable pointer values.

#### unsafe pointers

Dereferences of an unsafe pointer are always mutable values,
even if the unsafe pointer is immutable.
(This is important for refection implementation.)

#### structs

Fields of ref-immutable struct values are ref-immutable values.
Fields of both-immutable struct values are both-immutable values.

#### arrays

Elements of ref-immutable array values are ref-immutable values.
Elements of both-immutable array values are both-immutable values.

#### slices

Elements of immutable slice values are both-immutable values.
We can't append elements to immutable slice values.
The subslice result of an immutable slice is still an immutable slice.

#### maps

Elements of immutable map values are both-immutable values.
We can't append new entries to (or replace entries of,
or delete old entries from) immutable map values.

#### channels

We can send values to a ref-immutable channel.
Receiving from a ref-immutable channel results an immutable value.
Yes, we can send values to (and received values from) ref-immutable channels.
However, we can't send immutable values to mutable channels,
and we can't send values to (or receive values from) both-immutable channels.

#### functions

Function parameters and results can be declared as immutables
(either ref-immutable or both-immutable), including receiver parameters.
For the callers of a function, parameters/results declared as both-immutable
have no differences from parameters/results declared as ref-immutable.
For this reason, the mutable depth numbers can be omitted from function prototypes.

The prototypes of the following functions should be identical.
```golang
func fa(x 1:Tx, y 0:Ty) (z 0:Tz, w 1:Tw) {...}
func fb(x 1:Tx, y 1:Ty) (z 1:Tz, w 1:Tw) {...}
```

The type prototype of `fa` and `fb` can be denoted as
```golang
func (:Tx, :Ty) (:Tz, :Tw)
```

A `func(:T)` value is assignable to a `func(T)` value, not vice versa.
A `func()(T)` value is assignable to a `func()(:T)` value, not vice versa.

#### method sets

Every type has **two method sets**, one for mutable values, one for immutable values.
The immutable one is a subset of the mutable one.
For type `T` and `*T`, if methods can be declared for them (either explicitly or implicitly),
then the immutable method set of `T` is a subset of the immutable method set of `*T`.

#### interfaces

Immutable values can't be boxed into a mutable interface value or a both-immutable interface value.
They can only be boxed into ref-immutable interface values.
A mutable value can also be boxed into an immutable interface values
(**as long as the immutable method set of its type implements the interface**).
A type assertion on an immutable interface value results an immutable value. 

For this reason, the `xyz ...interface{}` parameter declarations of all the print functions
in the `fmt` standard package should be changed to `xyz ...1:interface{}` instead.

#### reflection

Many function and method implementations in the `refect` package should be modified accordingly.
The `refect.Value` type shoud have an **_immutable_** property,
and the result of an `Elem` method call should inherit the **_immutable_** property
from the receiver argument. More about reflection.
For all deails on reflection, please read the following reflection section.

### Usage examples

```golang
var x = []int{1, 2, 3}
var:1 y [][]int
y = [][]int{x, x} // ok

x[1] = 123     // ok
y[0][1] = 123  // error
var z = y[0]   // error
var:1 z = y[0] // ok
z[0] = 123     // error

// The following line <=> var:1 p = &z[0]
p := &z[0]     // ok. p is an immutable value.
*p = 123       // error
x[0] = *p      // ok
p = new(int)   // ok

var:1 v interface{} = y
var w = v.([][]int)   // error
var:1 w = v.([][]int) // ok
v = x                 // ok

// S is exported, but external packages have
// no ways to modify x and S (through S).
var:0 S = x     // ok.
S = x            // error
t := S[:]        // ok. <=> var:1 t =  s[:]
_ = append(t, 4) // error

// The elements of R even can't be modified in current package!
var:0 R = []int{7, 8, 9}
```

Another one:
```golang
var s = "hello word"
var:1 bytes = []byte(s) // a clever compiler will not allocate a
                        // deplicate underlying byte sequence here.
{
	pw := &s[6] // pw is an immutable *byte pointer value
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

A `reflect.ImmutableValueOf` function is needed to create `reflect.Value` values representing immutable Go values.
Its prototype is
```golang
func ImmutableValueOf(i :interface{}) Value
```

All parameters of type `reflect.Value` of the functions and methods in the `reflect` package,
including receiver parameters, should be declared as ref-immutable values).
However, the `reflect.Value` return results should be declared as mutable.

A `reflect.Value.ToImmutable` method is needed to convert a Value to an immutable one.

A `reflect.Value.ImmutableInterface` method is needed, it returns an immutable interface value.
The old `Interface` method panics on immutable values.

Three methods `reflect.Type.NumImmutableMethods`, `reflect.Type.ImmutableMethodByName` and `reflect.Type.ImmutableMethod` are needed.

In implementaion, one bit should be borrowed from the 23+ bits method number to represent the `immutable` proeprty.

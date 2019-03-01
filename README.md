# A proposal to support final values and read-only parameters/results in Go

Old versions:
* [the proposal thread](https://github.com/golang/go/issues/29422)
* [the initial proposal](README-v0.md)
* [the `var:N` version](README-v1.md)
* [the pure-immutable-value interpretation version](README-v2.md)
* [the immutable-type interpretation version](README-v3.md)
* [the immutable-type/value interpretation version: const.fixed](README-v4.md)
* [the immutable-type/value interpretation version: final.fixed. With interface design flaw](README-v5.md)


This proposal is not Go 1 compatible.
Please read the last section of this proposal for incompatible cases.

Any criticisms and improvement ideas are welcome, for
* I have not much compiler-related knowledge, so the following designs may have flaws.
* I haven't found a perfect syntax notation set for this proposal yet.

## The problems this proposal tries to solve

The problems this proposal tries to solve:
1. no ways to declare package-level exported immutable non-basic values.
1. no ways to declare read-only function parameters and results.

By solving the two problems, the security and performance of Go programs can be improved much.

## The main points of this proposal

We know each value has a property, `self_modifiable`, which means whether or not that value is modifiable.

This proposal will add a new value property `ref_modifiable` for each value, which means
whether or not the values referenced (either directly or indirectly) by that value are modifiable.

The permutation of thw two properties results 4 genres of values:
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

This proposal will let Go support the two value genres the current Go doesn't support,
and extend the range of `{self_modifiable: false, ref_modifiable: true}` values.

#### final values

This proposal treats the `self_modifiable` as a direct value property.
A **_final value_** concept is introduced.
* `{self_modifiable: true}` values (variables) are declared with `var`.
* `{self_modifiable: false}` values (finals) are declared with `final` (a new keyword).
   Please note that, although a `final` value itself can't be modified,
   the values referenced by the `final` value might be modifiable.
   (Same as JavaScript `const` values and Java `final` values.)

All intermediate results in Go should be viewed as final values,
including function returns, operator operation evaluation results,
explicit value conversion results, etc.

Final values may be addressable, or not.

#### fixed types and fixed values

This proposal treats `ref_modifiable` as a type property.
Surely, it is also an (indirect) value property.

Types with property `{ref_modifiable: false}` are called fixed types.
The notation `T.fixed` is introduced to represent the fixed version of a normal type `T`,
where `fixed` is a new introduced keyword.
A value of type `T.fixed` itself may be modifiable,
it is just that the values referenced (either directly or indirectly)
by the `T.fixed` value can't be modified (through the `T.fixed` value).

Later, values of a normal type `T` will be called normal values,
and values of a normal type `T.fixed` will be called fixed values,

A notation `v.fixed` is introduced to convert a value `v` of type `T` to a `T.fixed` value.
The `.fixed` suffix can only follow r-values (right-hand-side values).

More need to be noted:
* the notation `[]*chan T.fixed` can only mean `([]*chan T).fixed`,
  whereas `[]*chan (T.fixed)`, `[]*((chan T).fixed)` and `[]((*chan T).fixed)` are all invalid notations.
* `fixed` is not allowed to appear in type declarations. For example, `type T []int.fixed` is invalid.
* the respective fixed types of normal no-reference types (including basic types, fucntion types, struct types with only fields
  of no-reference types, and array types with no-reference element types) are the normal types themselves.

#### about final, fixed, and immutable values

A value is either a variable or a final. A value is either fixed or normal.

The relations of final and fixed values are:
* the values referenced by fixed values are final values.
* taking addresses of (addressable) final values results fixed values.

From the view of a fixed value, all values referenced by it, either directly or indirectly, are both final and fixed values.

Taking addresses of (addressable) fixed values results fixed values too.
(For safety, in the process, some write permissions may be lost.)

Repeat it again, although a final value can't be modified through the fixed values
which are referencing it, it is possible to be modified through other normal values which are referencing it.
In other words, some value hosted at a specified memory address may represent as a final or a variable, depending on different scenarios.
Similarly, some value hosted at a specified memory address may represent as fixed or normal, depending on different scenarios.

If a final value isn't referenced by any normal value,
then there are no (safe) ways to modifiy it,
so the final value is a true immutable value.
For example,
1. A declared final is guaranteed not to be referenced by any normal value.
   So it is a true immutable value.
1. If the expression `[]int{1, 2, 3}.fixed` is used as the initial value of a declared fixed slice value,
   then all the elements of the slice are guaranteed not to be referenced by any normal value.
   So they are all true immutable values.
1. Some fixed function return results. (If the doc of the function clearly says the results will
   not be referenced by any normal value after the function exits.)

Data synchronizations are still needed when concurrently reading
a final which is not a true immutable value.
But if a final is a true immutable value,
then there are no (safe) ways to modifiy it,
so concurrently reading it doesn't need data synchronization.

#### basic assignment rules

Below, for description convenience, the proposal will call
* `T` values declared with `var` as `var.normal` values.
* `T` values declared with `final` as `final.normal` values.
* `T.fixed` values declared with `var` as `var.fixed` values.
* `T.fixed` values declared with `final` as `final.fixed` values.

The basic assignment/binding rules:
1. A `final.*` value must be bound a value in its declaration.
   After the declaration, it can never be assigned any more.
1. `*.normal` values can be bound/assigned to a `*.normal` value.
1. Values of any genres can be bound/assigned to a `*.fixed` value, including constants, literals, variables,
   and the new supported values by this proposal.
1. Generally, `*.fixed` values can't be bound/assigned to a `*.normal` value, with one exception:
   `*.fixed` values of no-reference types will be viewed as be viewed as `*.normal` values (and vice versa) when
   they are used as source values in assignments.
   In other words, a value of any genre can be assigned to another value of any genre,
   if the two values are both of no-reference types, as long as they satisfy other old assignment requirements.

The section to the next will list the detailed rules for values of all kinds of types.
Those rules are much straightforward and anticipated.
**_They are derived from the above mentioned principle and basic assignment/binding rules._**

#### the difference from C++ const values

Please note, the immutability semantics in this proposal is different from the `const` semantics in C++.
For example, a value declared as `var p ***int.fixed` in this proposal is
like a variable decalared as `int const * const * const * p` in C++.
In C/C++, we can declare a variable as `int * const * const * x`,
but there are no ways to declare variables with the similar immutabilities in this proposal.
(In other words, this proposal thinks such use cases are rare in practice.)

For example, the following C++ code is valid.
```C++
#include <stdio.h>

typedef struct T {
	int* y;
} T;

int main() {
	int a = 123;
	T t = {.y = &a};
	const T* p = &t; // <=> T const * p = &t;
	*p->y = 789; // allowed
	printf("%d\n", *t.y); // 789
	return 0;
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

## Syntax changes

It is a challenge to design a both simple and readable syntax set for this proposal.
The current design may be not perfect, so any improvemnt ideas are welcome.

Some examples of the full value declaration form:
```golang
// A true immutable value which can't be modified in any (safe) way.
final FileNotExist = errors.New("file not exist").fixed  

// The following two declarations are equivalent.
// Note, the elements of "a" are all true immutable values.
var a []int.fixed = []int{1, 2, 3}
var a = []int{1, 2, 3}.fixed

// The following declarations are equivalent (for no-reference types only).
var b int
var b int.fixed

// Declare variables in a hybrid way.

// x is a var.fixed value, y is a var.normal value.
var x, y = []int{1, 2}.fixed, []int{1, 2}
// z is a final.normal value, w is a final.fixed value.
final z, w []int = []int{1, 2}, []int{1, 2}.fixed
```

Read-only parameter and result declaration examples:
```golang
func Foo(m http.Request.fixed, n map[string]int.fixed) (o []int.fixed, p chan int.fixed) {...}
func Print(values ...interface{}.fixed) {...}
```
All parameters and results in the above example are `var.fixed` values.
To avoid syntax design complexity, `final.*` parameters and results are not supported.

Short value declaration examples:
```golang
{
	oldA, newB := va, vb.fixed // newB is a var.fixed value

	// Explicit conversions.
	newX, oldY := (Tx.fixed)(va), vy
	newX, oldY := (Tx(va)).fixed, vy
	newX, oldY := Tx(va.fixed), vy
	newX, oldY := Tx(va).fixed, vy // equivalent to the above three lines
}
```

Again, to avoid syntax design complexity, `final.*` values can't be declared in short declartions.
In other words, values declared in short declarations are always `var.*` values.

## Detailed rules of this proposal

#### safe pointers

* Dereferences of `*.fixed` pointers are `final.fixed` values.
* Dereferences of `*.normal` pointers are `var.normal` values.
* Addresses of addressable `final.*` and `*.fixed` values are `*.fixed` pointer values.
  Some certain write permissions are lost when taking addresses of addressable `final.normal` and `var.fixed` values.

Yes, `final.*` values may be addressable.

Example:
```golang
final x = []int{1, 2, 3}

func foo() {
	y := &x  // y is var.fixed value of type *[]int.fixed
	z := *y  // *y is a final.fixed value, but
	         // z is deduced as a var.fixed value.
	z[0] = 9 // error: z[0] is a final value
	w := &z  // w is a var.fixed value (of type *[]int.fixed)
	...
}
```

#### unsafe pointers

* Fixed pointers and normal pointers can be both converted to unsafe pointers.
  This means the read-only rules built by this proposal can be broken by the unsafe mechanism.
  (This is important for reflection implementation.)

Example:
```golang
func mut(x []int.fixed) []int {
	return *((*[]int)(unsafe.Pointer(&x)))
}
```

#### structs

* Fields of `var.fixed` struct values are `var.fixed` values.
* Fields of `var.normal` struct values are `var.normal` values.
* Fields of `final.fixed` struct values are `final.fixed` values.
* Fields of `final.normal` struct values are `final.normal` values.

NOTE, there is [a pending design](https://github.com/golang/go/issues/29422#issuecomment-463427397)
to support specified value properties for struct fields.

#### arrays

* Elements of `var.fixed` array values are `var.fixed` values.
* Elements of `var.normal` array values are `var.normal` values.
* Elements of `final.fixed` array values are `final.fixed` values.
* Elements of `final.normal` array values are `final.normal` values.

#### slices

* Elements of `*.fixed` slice values are `final.fixed` values.
* Elements of `*.normal` slice values are `var.normal` values.
* We can't append elements to `*.fixed` slice values.
* Subslice:
  * The subslice result of a `*.fixed` slice is still a `final.fixed` slice.
  * The subslice result of a `*.normal` slice is still a `final.normal` slice.
  * The subslice result of a `final.*` or `*.fixed` array is a `var.fixed` slice.
    Some certain write permmisions may be lost in the process.

Example 1:
```golang
type T struct {
	a int
	b *int
}

// The type of x is []T.fixed.
var x = []T{{123, nil}, {789, new(int)}}.fixed

func foo() {
	x[0] = nil   // error: x[0] is a final value
	x[0].a = 567 // error: x[0] is a final value
	y := x[0]    // x[0] is a final.fixed value, but y is deduced
	             // as a var.fixed value (of type T.fixed).
	y.a = 567    // ok
	*y.b = 567   // error: y.b is a fixed value
	y.b = nil    // ok
	z := x[:1]   // z is var.fxied value (of type []T.fixed)
	x = nil      // ok
	y = T{}      // ok
	
	final w = x // w is a final.fixed value
	u := w[:]   // w[:] is a final.fixed value, but
	            // u is deduced as a var.fixed value.
	
	// v is a final.normal value.
	final v = []T{{123, nil}, {789, new(int)}}
	v = nil    // error: v is a final value
	v[1] = T{} // ok
	
	_ = append(u, T{}) // error: can't append to fixed slices
	_ = append(v, T{}) // ok
	
	...
}
```

Example 2:
```golang
var x = []int{1, 2, 3}

// External packages have no ways to modify elements of x (through S).
final S = x.fixed // ok.

// The elements of R even can't be modified in current package!
// It is a true immutable value.
final R = []int{7, 8, 9}.fixed

// Q itself can't be modified, but its elements can.
final Q = []int{7, 8, 9}
```

Example 3:
```golang
var s = "hello word"
var bytes = []byte.fixed(s) // a clever compiler will not allocate a
                            // duplicate underlying byte sequence here.
{
	pw := &s[6] // pw is a `var.fixed` poiner whose base type is "byte".
}
```

#### maps

* Elements of `*.fixed` map values are `final.fixed` values.
* Elements of `*.normal` map values are `final.normal` values.
  (Although map elements are final values, each of they can be replaced as a while.)
* Keys (exposed in for-range) of `*.fixed` map values are `final.fixed` values.
* Keys (exposed in for-range) of `*.normal` map values are `final.normal` values.
* We can't append new entries to (or replace entries of,
  or delete old entries from) `*.fixed`  map values.

Example:
```golang
type T struct {
	a int
	b *int
}

// x is a true immutable value.
final x = map[string]T{"foo": T{a: 123, b: new(int)}}.fixed

bar(x) // ok

func bar(v map[string]T.fixed) { // v is a var.fixed value
	// Both v["foo"] and v["foo"].b are fixed values.
	*v["foo"].b = 789 // error: v["foo"].b is a fixed value
	v["foo"] = T{}    // error: v["foo"] is a fixed map
	v["baz"] = T{}    // error: v["foo"] is a fixed map
	
	// m will be deduced as a var.fixed value.
	// That means as long as one element or one key is fixed
	// in a map literal, then the map value is a fixed value.
	m := map[*int]*int {
		new(int): new(int).fixed,
		new(int): new(int),
		new(int): new(int),
	}
	for a, b := range m {
		// a and b are both var.fixed values.
		// Their types are *int.fixed.
		
		*a = 123 // error: a is a fixed value
		*b = 789 // error: b is a fixed value
	}
}

```

#### channels

* Send
  * We can't send values to `final.*` channels.
  * We can send values of any genres to a `var.fixed` channel.
  * We can only send `*.normal` values to a `var.normal` channel.
* Receive
  * We can't receive values from `final.*` channels.
  * Receiving from a `*.normal` channel results a `final.normal` value.
  * Receiving from a `*.fixed` channel results a `final.fixed` value.

Example:
```golang
var x = new(int)
final ch = make(chain *int, 1)

func foo(c chan *int.fixed) {
	x := <-c // ok. x is a var.fixed value (of type *int.fixed)
	y := new(int)
	c <- y  // ok
	ch <- x // error: ch is a final channel
	<-ch    // error: ch is a final channel
	...
}
```

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
func (T.fixed) M() {} // explicitly declared. (A fixed method)
/*
func (t T) M() {t.fixed.M()} // implicitly declared. (A normal method)
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

An interface type can specify some read-only methods. For example:
```golang
type I interface {
	M0(Ta) Tb // a normal method

	fixed.M2(Tx) Ty // a fixed method (also called receiver-read-only method).
	                // NOTE: this is an exported method.
}
```

Similar to non-interface type, if a fixed interface type explicitly specified a read-only method `fixed.M`,
it also implicitly specifies a normal method with the same name `M`.

The method set specified by type `I` contains three methods, `M0`, `fixed.M2` and `M2`.
The method set specified by type `I.fixed` only contains one method, `fixed.M2`.

When a method is declared for a concrete type to implement a fixed method,
the type of the receiver of the declared method must be fixed.
For example, in the following code snippet,
the type `T1` implements the interface `I` shown in the above code snippet, but the type `T2` doesn't.
```golang
type T1 struct{}
func (T1) M0(Ta) Tb {var b Tb; return b}
func (T1.fixed) M2(Tx) Ty {var y Ty; return y} // the receiver type is fixed.

type T2 struct{}
func (T2) M0(Ta) Tb {var b Tb; return b}
func (T2) M2(Tx) Ty {var y Ty; return y} // the receiver type is normal.
```

Please note, the type `T3` in the following code snippet also implements `I`.
Please read the above function section for reasons.
```golang
type T3 struct{}
func (T3) M0(Ta.fixed) Tb {var b Tb; return b}
func (T3.fixed) M2(Tx.fixed) Ty {var y Ty; return y} // the receiver type is fixed.
```

If a normal type `T` implements a normal interface type `I`,
then the fixed type `T.fixed` also implements the fixed interface type `I.fixed`.

* Dynamic type
  * The dynamic type of a `*.normal` interface value is a normal type.
  * The dynamic type of a `*.fixed` interface value is a fixed type.
* Box
  * No values can be boxed into `final.*` interface values (except the initial bound values).
  * `*.fixed` values can't be boxed into `var.normal` interface values.
  * Values of any genres can be boxed into a `var.fixed` interface value.
* Assert
  * A type assertion on a `*.fixed` interface value results a `final.fixed` value.
    For such an assertion, its syntax form `x.(T.fixed)` can be simplified as `x.(T)`.
  * A type assertion on a `*.normal` interface value results a `final.normal` value.

For this reason, the `xyz ...interface{}` parameter declarations of all the print functions
in the `fmt` standard package should be changed to `xyz ...interface{}.fixed` instead.

Example:
```golang
var x = []int{1, 2, 3}
var y = [][]int{x, x}.fixed // ok
var v interface{} = y       // error: can't assign a fixed value to a normal value.
var v interface{}.fixed = y // ok
var w = v.([][]int)         // ok, w is a var.fixed value (of type [][]int.fixed)
v = x                       // ok
var u = v.([]int)           // ok, u is a var.fixed value (of type []int.fixed)
var u = v.([]int.fixed)     // ok, equivalent to the above one, for v is fixed.
```

#### reflection

Many function and method implementations in the `refect` package should be modified accordingly.
The `refect.Value` type shoud have a **_fixed_** property,
and the result of an `Elem` method call should inherit the **_fixed_** property
from the receiver argument. More about reflection.
For all details on reflection, please read the following reflection section.

## Compiler implementation

I'm not familiar with the compiler development things.
It is just my feeling, by my experience,
that most of the rules mentioned in this proposal
can be enforced by compiler without big technology obstacles.

At compile phase, compiler should maintain two bits for each value.
One bit means whether or not the value itself can be modified.
The other bit means whether or not the values referenced by the value can be modified.

At compile phase, **_read-only_** is not represented as a type property.

## Runtime implementation

Except the next to be explained reflection section, the impact on runtime
made by this proposal is not large.

Each internal method value should maintain a `fixed` property.
This information is useful when boxing a fixed value into an interface.

## New reflection functions and methods and how to implement them

The current `reflect.Value.CanSet` method will report whether or not a value can be modified.

A `reflect.FixedValueOf` function is needed to create `reflect.Value` values representing `*.fixed` Go values.
Its prototype is
```golang
func FixedValueOf(i interface{}.fixed) Value
```
For the standard Go compiler, in implementaion,
one bit should be borrowed from the 23+ bits method number to represent the `fixed` proeprty.

All parameters of type `reflect.Value` of the functions and methods in the `reflect` package,
including receiver parameters, should be declared as `var.fixed` values.
However, the `reflect.Value` return results should be declared as `var.normal` values.

A `reflect.Value.ToFixed` method is needed to make a `reflect.Value` value represent a `final.fixed` Go value.

A `reflect.Value.FixedInterface` method is needed, it returns a `final.fixed` interface value.
The old `Interface` method panics on `*.fixed` values.

A method `reflect.Type.Fixed` is needed to get the fixed version of a normal type.
A method `reflect.Type.Normal` is needed to get the normal version of a fixed type.
The method sets of normal type `T` and fixed type `T.fixed` may be different.
Their respective other properties should be identical.

A method `reflect.Type.Genre` is needed, it may return `Fixed` or `Normal` (two constants).

## Unsolved and new problems of this proposal

This proposal doesn't guarantee some values referenced by `*.fixed` values will never be modified.
(This is more a feature than a problem.)

This proposal will make `bytes.TrimXXX` (and some others) functions need some duplicate versions for normal and fixed arguments.
This problem should be solved by future possible generics feature.

## Go 1 incompatible cases

The followings are the incompatible cases I'm aware of now.
1. `final` and `fixed` may be used as non-exported identifiers in old user code.
   It should be easy for the `go fix` command to modify these uses to others.
   (Using `const` to replace `final` and `fixed` can avoid this incompatible case, but may cause some confusions.)

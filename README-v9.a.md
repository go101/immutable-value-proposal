# A proposal to support read-only and practical immutable values in Go

Comparing to the v9.1 revision, this revision (v9.a)
* removes the **final value** concept, thus this revision doesn't need the `final` keyword and becomes ([almost](#go-1-incompatible-cases-and-solutions)) Go 1 compatible.
* removes the restriction that sending to and receiving from read-only channels are disallowed.

Any criticisms and improvement ideas are welcome, for
* I have not much compiler-related knowledge, so the following designs may have flaws.
* I haven't found a perfect syntax notation set for this proposal yet.

## The problems this proposal tries to solve

The problems this proposal tries to solve:
1. no ways to declare package-level exported immutable non-basic values.
1. no ways to declare read-only function parameters and results.
1. many inefficiencies caused by lacking of immutable and read-only values.

Detailed rationales are listed in [at the end of this proposal](#rationales).
Some solutions for the drawbacks mentioned by Russ are also listed there.

Basically, this proposal can be viewed as a combination
of [issue#6386](https://github.com/golang/go/issues/6386)
and [issue#22876](https://github.com/golang/go/issues/22876),
plus several new ideas and much more details.
the new ideas and more details make this proposal more practical to implement.

This proposal also has some similar ideas with
[evaluation of read-only slices](https://docs.google.com/document/d/1-NzIYu0qnnsshMBpMPmuO21qd8unlimHgKjRD9qwp2A)
written by Russ.

This propsoal is not intended to be accepted, at least soon.
It is more a documentation to show the problems and possible solutions
in supporting immutable and read-only values in Go.
I hope this proposal can help others get a good understanding on the problems
and current solutions in supporting read-only values in Go,
so that they can get inspired and improve the current read-only value proposals
or make their own better read-only value proposals.

## The main points of this proposal

#### value roles

This proposal introduces a **value role** concept.

There are four kinds of value roles:
1. **reader** role. The values directly referenced by a reader value are all read-only values (from the view of the reader value).
1. **read-only** role. A read-only value may not be modified (through the read-only value itself).
   If the read-only value can reference some values, then the read-only value is also a reader value,
   so the values directly referenced by it are all read-only values too (from the view of the read-only value).
1. **writer** role. The values directly referenced by a writer value are all writable values (from the view of the writer value).
1. **writable** role. A writable value may be modified (through the writable value itself).
   If the writable value can reference some values, then the writable value is also a writer value,
   so the values directly referenced by it are all writable values too (from the view of the writeable value).

Please note,
* A reader value, if it is not a read-only value, might be modifiable.
* A writer value, if it is not a writable value, might not be modifiable.

Also note, values of some types never reference any values,
or they are referencing some interval values which can't be modified in any ways in user code.
Such types are called **no-references types** below.
The **reader** and **writer** roles are non-sense for values of no-references types.
Some no-references type examples:
* basic types
* function types
* struct types with all field types are no-references types
* array types with no-references element types
* channel types with no-references element types

Values of no-references types have adaptive roles when they are used as R-values (right-hand-side values). In other words,
* Values of no-references types are viewed as writer values when they are assigned to writer or writable values.
* Values of no-references types are viewed as reader values when they are assigned to reader or read-only values.

We can think values of no-references types as **unroled values** when they are used as R-values.

Literal and constant values are all unroled values.

As a special, the predeclared `nil` is also an unroled value.

#### assignment and conversion rules

The following rules stand if they stand without considering value roles.
* Reader or read-only values can be bound to read-only values. Read-only values can't be re-assigned to later.
* Reader or read-only values can be assigned to reader values.
* Writer or writable values can be **explicitly** converted to read-only values, not vice versa. (Here the **explicitly** means the `:rr` suffix is required. See below for details.)
  The conversion results can be assigned to reader values or bound to read-only values.
  (About why writer or writable values can't be implicitly
  converted to reader or read-only values, please read
  [the problem mentioned in v9](README-v9.0.md#the-problem-when-reader-parameters-in-a-library-package-changed-to-writers).)
* Writer or writable values can be assigned to writer or writable values.
* Unroled values can be assigned/bound to values with any roles.

#### new syntax set

A suffix notation form `:role` is introduced as value suffixes to indicate the roles of some values, where `role` in the form can only be `rr` (reader) or `ro` (read only).
* We can use `var:ro` to delcare read-only variables. (`:ro` can be only used in `var:ro`.)
  A declared read-only variable can be bound to an initial value.
* We can use `var:rr` to declare reader variables.
* We can use `v:rr` to explicitly convert writer or writable value `v` to a reader value.

For example,
```
// x is a read-only (and reader) value.
// The elements of x are practical immutable values.
var:ro x = []int{1, 2, 3}
x = nil // error

// y is reader value.
// The elements of y are practical immutable values,
// but y itself is modifiable.
var:rr y = []int{1, 2, 3}
y = nil // ok

// w is a writable (and writer) value.
var w = []int{1, 2, 3}

// z is a reader value.
// It and w share elements.
// The elements are read-only from the view of z,
// but they are writable from the view of w.
// So the elements are not immutable values.
var:rr z = w:rr
z[0] = 9 // error
w[0] = 9 // ok
```

Please note, in the above example, the two `:rr` suffixes in the `var:rr z = w:rr` line are both required.
The reason is `w` is a roled value.
On the other hand, literal and constant values are all unroled values.
And again, implicit role convertions are disallowed here.

There are no ways to declare read-only variables in short declarations:
```
var w = []int{1, 2, 3}
{
	// u is a writer value.
	// v is a reader value.
	u, v := w, w:rr

	... // use u and v
}
```

Although roles are properties of values, to make code (function prototype literals specificly) look compact, we can think of they are also properties of types.
The notation `T:role` is introduced to represent a type `T` with a specified role.
And to ease the syntax design, only `:rr` is allowed to be used as type suffixes.
However, please note,
* The notation `T:rr` is not allowed to appear in type declarations to declare reader types.
* The notation `T:rr` can only be used to specify function parameter and result types.
  It may not be used to declare package-level variables and struct fields.
* In the `T:rr` notation, the `:rr` portion must be the last protion.
  For example, the notation `[]*chan T:rr` can only mean `([]*chan T):rr`,
  whereas `[]*chan (T:rr)`, `[]*((chan T):rr)`
  and `[]((*chan T):rr)` are all invalid notations.
  For more examples, it is not allowed to be used to
  * specify the roles of struct field types.
  * specify the roles of array/slice/map/channel element types.
  * specify the roles of map key types.
  * specify the roles of pointer base types.

Although it is a non-sense, to avoid const-poisoning alike problems,
the `:rr` sufix is allowed to follow no-references types.

A detail related to the `T:rr` notation need to be noted:
as long as one result of a function is specified with a reader role,
then the result list part of the function prototype literal
must be enclosed in a pair of `()`.
For example, the notations `func() (T:rr)` and `func() T:rr` are different.
The former denotes a result type with the reader role,
but the latter denotes a function type with the reader role
(a non-sense role, for function types are no-references types).

To avoid function declaration splitting problem, a **parameterized role** concept is introduced.
The notation `::r` (two colons) produces a parameterized role `r`.
For example, we can declare the `Split` function in the `bytes` standard package as
```
// Here, the two "q" must be consistent.
func Split(v []byte::q, sep []byte:rr) ([][]byte::q) {...}
```

to avoid declaring it as two functions
```
func SplitReader(v []byte:rr, sep []byte:rr) ([][]byte:rr) {...}
func SplitWriter(v []byte, sep []byte:rr) [][]byte {...}
```

An example containing some function declarations and calls:
```
func Double(x []byte) {
	for i, v := range x {
		x[i] = v+v
	}
}

func DoubleDup(x []byte:rr) []byte {
	y := make([]byte, len(x))
	for i, v := range x {
		y[i] = v+v
	}
	return y
}

var w = []byte{2, 3, 5}
Double(w)
fmt.Println(w) // [4 6 10]
var v = DoubleDup(w:rr)
fmt.Println(w) // [4 6 10]
fmt.Println(v) // [8 12 20]

// We can use strings as reader byte slices.
// This makes the current append([]byte, string) and
// copy([]byte, string) become not syntax exceptions.
v = DoubleDup("hello")
fmt.Println(v) // [208 202 216 216 222]
```

## Detailed rules for values of all kinds of types

#### safe pointers

* Dereference of a reader pointer results a read-only value.
* Dereference of a writer pointer results a writable value.
* Taking address of an addressable a reader or read-only value results a reader pointer.
* Taking address of an addressable writer value results a writer pointer.

Example:
```
var:ro x = []int{1, 2, 3}

func foo() {
	y := &x  // y is reader pointer variable
	z := *y  // z is deduced as a reader variable
	w := x   // w is deduced as a reader variable
	z[0] = 9 // error: z[0] is read-only.
	u := &z  // u is like y, a reader pointer

	p1 := &x[1] // p1 is a reader pointer variable.
	p2 := &z[1] // p2 is a reader pointer variable.
	...
}
```

Note: taking address of a string results a read-only pointer values. Example:
```
var s = "hello word"
var:rr p1 = &bs[6] // ok
var p2 = &bs[6]    // error
```

#### unsafe pointers

* Reader pointers and writer pointers can be both converted to unsafe pointers.
  This means the read-only rules built by this proposal can be broken by the unsafe mechanism.
  (This is important for reflection implementation.)

Example:
```
func mut(x []int:rr) []int {
	return *((*[]int)(unsafe.Pointer(&x)))
}
```
#### structs

* Fields of reader struct values are also reader values.
* Fields of read-only struct values are also read-only values.
* Fields of writer struct values are also writer values.
* Fields of writable struct values are also writable values.

#### arrays

* Elements of reader array values are also reader values.
* Elements of read-only array values are also read-only values.
* Elements of writer array values are also writer values.
* Elements of writable array values are also writable values.

#### slices

* Elements of reader or read-only slice values are read-only values.
* Elements of writer or writable slice values are writer values.
* We can't append elements to writer or writable slice values,
  but can't append elements to reader or read-only slice values.
* Subslice:
  * The subslice result of a reader or read-only slice is a read-only slice.
  * The subslice result of a writer or writable slice is still a writer slice.
  * The subslice result of a reader of read-only array is a read-only slice.

Note: converting a string to a byte slice results a read-only byte slice value. Example:
```
var s = "hello word"

// "bs" is a reader byte slice.
// A clever compiler will not allocate a
// duplicate underlying byte sequence here.
var:rr bs = []byte(s)
```

Note, internally, the `cap` field of a reader byte slice is set to `-1`
if the byte slice is converted from a string, so that Go runtime knows
its elements are immutable. Converting such a reader byte slice to
a string doesn't need to duplicate the underlying bytes.

#### maps

* Elements of reader or read-only map values are read-only values.
* Elements of writer or writable map values are writer values.
  (Each writable map element must be modified as a whole.)
* Keys (exposed in for-range) of reader or read-only map values are read-only values.
* Keys (exposed in for-range) of writer or writable map values are writer values.
* We can't append new entries to (or replace entries of,
  or delete old entries from) reader or read-only map values.

#### channels

* Send
  * We can only send reader or read-only values to a reader or read-only channel.
  * We can only send writer or writable values to a writer channel.
* Receive
  * Receiving from a reader channel results a read-only value.
  * Receiving from a writer or writable channel results a writer value.

#### functions

Function parameters and results can be declared as reader variables.

In the following function proptotype, parameter `x` and result `w` are declared as reader variables.
```
func fa(x Tx:rr, y Ty) (z Tz, w Tw:rr) {...}
```

A `func()(T)` value is assignable to a `func()(T:rr)` value, not vice versa.

A `func(T:rr)` value is assignable to a `func(T)` value, not vice versa.

A declared `func(Tx::q) (Ty::q)` function can be used as R-values
and assigned to `func(Tx:rr) (Ty:rr)` or `func(Tx) Ty` values.
But it can't be used as L-values, for it has not a certain type.
(Yes, it is an untyped value.)

#### method sets

We can delcare methods with recievers of reader types.

When a method `M` is explicitly declared for reader type `T:rr`, then a corresponding method with the same name must be declared for writer type `T` by compilers **if no explicit method with the same name declared for writer type `T`**. The rule ensures that the method set of reader type `T:rr` is always a subset of writer type `T`.

```
func (T:rr) Mx() {} // explicitly declared. (A read-only method)
func(T) My() {}
/*
func (t T) Mx() {t:rr.Mx()} // implicitly declared. (A writer method)
*/

var t T
t.Mx() // <=> t:rr.Mx()
```

In the above code snippet, the method set of reader type `T:rr` contains one method: `Mx`, however the method set of type `T` contains two method: `Mx` and `My`.

For type `T` and `*T`, if methods can be declared for them (either explicitly or implicitly), the method set of type `T:rr` is a subset of type `*T:rr`.

When the receiver type is specified with a parameterized role in a method declaration, then two methods are declared actually. One is for writer receiver type, the other is for reader receiver type.
For example,
```
func (T::q) M{Tx} Ty::q {...}
```
is equivalent to
```
func (T) M(Tx) Ty {...}
func (T:rr) M(Tx) Ty:rr {...}
```

#### interfaces

An interface type can specify some read-only methods. For example:
```
type I interface {
	M0(Ta) Tb    // a writer method
	rr M2(Tx) Ty // a reader method (exported)
}
```

Similar to non-interface types, there is an implicit method `M2` specified for the writer version of the above shown interface type.
The method set specified by type `I` contains two methods actually, `M0` and `M2`.
The method set specified by type `I:rr` only contains one method, `M2`.

In the following code snippet, the type `T1` implements the interface `I` shown in the above code snippet, but the type `T2` doesn't. The reason is type `T2:rr` has not a `M2` method.
```
type T1 struct{}
func (T1) M0(Ta) Tb {var b Tb; return b}
func (T1:rr) M2(Tx) Ty {var y Ty; return y} // the receiver type is a reader type.

type T2 struct{}
func (T2) M0(Ta) Tb {var b Tb; return b}
func (T2) M2(Tx) Ty {var y Ty; return y} // the receiver type is a writer type.
```

Please note, the type `T3` in the following code snippet also implements `I`.
Please read the above function section for reasons.
```
type T3 struct{}
func (T3) M0(Ta:rr) Tb {var b Tb; return b}
func (T3:rr) M2(Tx:rr) Ty {var y Ty; return y}
```

If a writer type `T` implements a writer interface type `I`, then the reader type `T:rr` also implements the reader interface type `I:rr` for sure.

Boxing and assertion rules:
* The value boxing rules are like the assignment and conversion rules mentioned above.
* Assertion rules:
  * A type assertion on a reader or read-only interface value results a read-only value.
  * A type assertion on a writer or writable interface value results a writer value.

## Reflection

Many function and method implementations in the `refect` package should be modified accordingly.
The `refect.Value` type shoud have a **_reader_** property,
and the result of an `Elem` method call should inherit the **_reader_** property from the receiver argument.

The current `reflect.Value.CanSet` method will report whether or not a value can be modified.

A `reflect.ReaderValueOf` function is needed to create
`reflect.Value` values representing reader Go values.
Its prototype is
```
func ReaderValueOf(i interface{}:rr) Value
```
For the standard Go compiler, in implementaion,
one bit can be borrowed from the 23+ bits method number
to represent the `reader` proeprty.

All parameters of type `reflect.Value` of the functions and methods
in the`reflect` package, including receiver parameters,
should be declared as reader variables.
However, the `reflect.Value` return results should be declared as writers.

A `reflect.Value.ToReader` method is needed to
make a `reflect.Value` value represent a reader Go value.

A `reflect.Value.ReaderInterface` method is needed,
it returns a reader interface value.
The old `Interface` method panics on reader values.

A method `reflect.Type.Reader` is needed to get the reader version of a writer type.
A method `reflect.Type.Writer` is needed to get the writer version of a reader type.
The method sets of reader type `T:rr` is the subset of the writer type `T`.
Their respective other properties should be identical.

A method `reflect.Type.Role` is needed,
it may return `Reader` or `Writer` (two constants).


## Compiler implementation

I'm not familiar with the compiler development things.
It is just my feeling, by my experience,
that most of the rules mentioned in this proposal
can be enforced by compiler without big technology obstacles.

At compile phase, compiler should maintain two bits to represent a value role, so that to make decisions according to the rules mentioned above.

## Runtime implementation

Except the changes mentioned in the above reflection section,
the impact on runtime made by this proposal is not large.

Each internal method value should maintain a `reader` property.
This information is useful when boxing a reader value into an interface.

As above has mentioned, the cap field of a reader byte slice should be set to `-1` if its byte elements are immutable.

## Go 1 incompatible cases and solutions

There are two Go 1 incompatible cases.

#### reader arguments

As above mentioned, when a writer or writable value is passed to a reader parameter,
the writer or writable value must be explicitly converted to a reader or read-only value to act as a legal reader argument. This will break much user code which contains calls to functions in standard packages, such as `fmt.Print`, for the prototype of the `fmt.Print` function is expected to change to `func(...interface{}:rr)`.

Solution 1: To avoid such incompatibilities, the prototype of the `fmt.Print` can be changed to `func(...interface{}::q)` instead.
This solution can only apply to standard packages. (Similarly, should the unroled predeclared `nil` be declared with `var::q`?)

Solution 2: Use `go fix` to fix the calls in user code.

Solution 3: Publish v2 version of some standard modules.

#### reader/read-only package-level variables

Many exported `error` values declared (with `var` now) in standard packages are expected to be changed as read-only values (declared with `var:ro` later). However, there might be some user code in whcih these `error` values are assigned to some variables. From the rules mentioned above, such assignments will become illegal later.

Solution 1: Use `go fix` to fix these assignments in user code.

Solution 2: Publish v2 version of some standard modules.

## Rationales

#### rationales of Go needs read-only and immutable values

In [evaluation of read-only slices](https://docs.google.com/document/d/1-NzIYu0qnnsshMBpMPmuO21qd8unlimHgKjRD9qwp2A),
Russ mentions some inefficiencies caused by lacking of read-only values.
1. Now, the type of the only parameter of `io.Writer.Write` method is `[]byte`.
   In fact, it should be a read-only parameter, for two reasons:
   1. so that a `Write` call can take a string value argument without making
      a duplicate underlying bytes of the string argument. (More efficient)
   1. the current method prototype doesn't prevent a custom `Writer`
      from modifying the elements of the passed `[]byte` argument. (More secure)
1. By specifying a parameter of a function as read only, users of the function
   will clearly know the corresponding passed arguments will not be modified
   in this funciton. (Better code as document)

Besides these improvements, immutable values (this proposal supports)
can raise the security of Go code.
For example, by changing many exported `error` values in standard packages
to immutable values, the securities of Go programs are improved.

Immutable slice values can also let compilers to do more
BCE (bounds check elimination) optimizations for them.

The "Strengths of This Proposal" section in @jba's [propsoal](https://github.com/golang/go/issues/22876)
also makes a good summary of the benefits of read-only values.

#### rationales for the `T:rr` notation

It is more compact than `var:rr T`. I think `func (Ta:rr) (Tx:rr)` has a better readibility than `func (var:rr Ta)(var:rr Tx)`.

#### about the problems of read-only values mentioned by Russ

In [evaluation of read-only slices](https://docs.google.com/document/d/1-NzIYu0qnnsshMBpMPmuO21qd8unlimHgKjRD9qwp2A),
Russ mentions some problems of read-only values.
This proposal provides solutions to avoid each of these drawbacks.

1. **function duplication**

Solved by role parameter.

Please see the end of [the function section](#functions).

2. **immutability and memory allocation**

Solved by setting the capacities of immutable byte slices as `-1` at run time.

Please see the end of [the slice section](#slices).

3. **loss of generality**

Solved by letting `interface {M(T:rr)}` implement `interface {M(T)}`
and letting `interface {M() T}` implement `interface {M() T:rr}`.

Please see [the interface section](#interfaces) for details.

#### about partial read-only

Sometimes, people may need partial read-only for struct values.

```
type Counter struct {
	n  uint64

	// mu will be always writable,
	// even if its containing struct
	// value is a read-only value.
	mu sync.Mutex:writable
}

func (c *Counter:rr) Value() uint64 {
	// ok. c.mu will be modified,
	// which is allowed, even if
	// *c is a read-only value.
	c.mu.Lock()
	defer c.mu.Unlock() // ok

	return c.n
}
```

Partial read-only will make this proposal much more complex, so I decided not to support it.

#### no-referenes types or no-references values

The current proposal determines whether or not a value is referencing other values by checking whether or not its type is a no-references type. The checking happens at compile time. However, in fact, many run-time values can reference other values but at a certain time they are not referencing any values. Such values are called no-references values. In theory, it is not a problem to assign such no-references values with reader role to writer values. But this can't be determined at compile time, so an invalid such assignment must panic. For simplity, the current proposal adopts no-references types instead of no-reference values.


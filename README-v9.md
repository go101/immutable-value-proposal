# A proposal to support read-only and immutable values in Go

Old versions:
* [the proposal thread](https://github.com/golang/go/issues/29422), and the [golang-dev thread](https://groups.google.com/forum/#!topic/golang-dev/5M9F09S_k0g)
* [the initial proposal](README-v0.md)
* [the `var:N` version](README-v1.md)
* [the pure-immutable-value interpretation version](README-v2.md)
* [the immutable-type interpretation version](README-v3.md)
* [the immutable-type/value interpretation version: const+fixed](README-v4.md)
* [the immutable-type/value interpretation version: final+fixed. With interface design flaw](README-v5.md)
* [final+fixed. Without partial read-only](README-v6.md)
* [final+fixed. With partial read-only](README-v7.md)
* [const+reader/writer roles. Partial read-only removed](README-v8.md)
* final+reader/writer roles. (v9, the currrent version)

_(This revision reverts the `const` keyword in v8 to `final`, to avoid some confusions.)_

This revision is a little Go 1 incompatible, for it needs a new keyword `final`.

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
and [issue#22876](https://github.com/golang/go/issues/22876).

This proposal also has some similar ideas with
[evaluation of read-only slices](https://docs.google.com/document/d/1-NzIYu0qnnsshMBpMPmuO21qd8unlimHgKjRD9qwp2A)
written by Russ.

However, this proposal has involved so much that
it has become into a much more practical solution with
more ideas and details than the just mentioned ones.

This propsoal is not intended to be accepted, at least soon.
It is more a documentation to show the problems and possible solutions
in supporting immutable and read-only values in Go.

## The main points of this proposal

We know each value has a property, `self_modifiable`,
which means whether or not that value itself is modifiable.

This proposal will add a new value property `ref_modifiable` for each value.
This property means whether or not the values referenced
(either directly or indirectly) by a value are modifiable.
The `ref_modifiable` property will be called a value role later.

The permutation of thw two properties results 4 genres of values:
1. `{self_modifiable: true, ref_modifiable: true}`.
   Such as variables.
1. `{self_modifiable: true, ref_modifiable: false}`.
   No such Go values currently.
1. `{self_modifiable: false, ref_modifiable: true}`.
   Such as composite literals.
   (In fact, all constants in JavaScript and
   all final variables decalred in Java belong to this genre.)
1. `{self_modifiable: false, ref_modifiable: false}`.
   No such Go values currently.

(Note, in fact, we can catagory declared function values, method values
and constant basic values into either the 3rd or the 4th genre.)

This proposal will let Go support values of the 2nd and 4th genres,
and extend the value range of the 3rd genre.

#### variables and finals

This proposal treats the `self_modifiable` as a value property.
* `{self_modifiable: true}` values (variables) are declared with `var`.
* `{self_modifiable: false}` values (finals) are declared with `final`
   Like named constants, a named final must be bound a value in its declaration.
   But unlike named constants, finals can be values of any type, not limited to values of basic types.
   We can view finals as runtime constants.
   Please note that, although a final itself can't be modified,
   the values referenced by the final might be modifiable.
   (Much like JavaScript `const` values and Java `final` values.)

Most intermediate results in Go should be viewed as final values,
including function returns, operator operation evaluation results,
explicit value conversion results, etc.

Finals (themselves) are immutable values.
Note, although a final itself is an immutable value,
whether or not the values referenced by the final are immutable
values depends on the specified role (see the next section) of the final.

There is not a short final declartion form.
Shorted declared values are all variables.
Function parameters and results also can't be delcared as finals.

#### value roles: reader and writer

This proposal proposes to add a value role concept to
denote the `ref_modifiable` property.
Although roles are properties of values,
to ease the syntax designs, we can think they are also properties of types.

The notation `T:reader` is introduced to represent a reader type.
Its values are called reader values.
The notation can be used to declare package-level variables/finals,
local variables/finals, and function parameter/result variables.
But it can't be used to specifiy struct field types.
Fields of a struct value will inherit the roles from the struct value.

There is not the `T:writer` notation. 
The *writer role* concept does exist.
The raw `T` notation means a writer type (in non-struct-field declarations). 

Thw word `reader` can be either a keyword or not.
If it is designed as not, then it can be viewed as a predeclared role.

The meanings of **reader** and **writer** values:
* All values referecned by a reader value are read-only (and also reader) values
  (from the view of the reader value).
  In other words, a reader value represents a read-only value chain,
  and the reader value is the head of the chain.
  Note, the reader head itself might be a variable, which is not a read-only value.
* All values referecned by a writer value are writable (and also writer) values
  (from the view of the writer value).
  In other words, a writer value represents a writable value chain,
  and the writer value is the head of the chain.
  Note, the writer head itself might be a final, which is a read-only value.

Some details about the `T:reader` notation need to be noted:
1. `:reader` is not allowed to appear in type declarations,
   except it shows up as function parameter and result roles.
   * For example, `type T []int:reader` is invalid,
     but `type T func (T:reader)` is valid.
   * And please note that, as long as one result of a function is a reader value,
     then the result list part of the function proptotype literal
     must be enclosed in a pair of `()`. For example,
     `func() T:reader` is invalid, it must be `func() (T:reader)`.
1. The notation `[]*chan T:reader` can only mean `([]*chan T):reader`,
   whereas `[]*chan (T:reader)`, `[]*((chan T):reader)`
   and `[]((*chan T):reader)` are all invalid notations.
1. Some kinds of types are always non-reader types, including basic types and function types.
   So, `:reader` is not allowed to follow such type names and literal.
   For example (again), `func() T:reader` is invalid.
1. Struct types which all field types are non-reader types
   and array/channel types with non-reader element types
   are also non-reader types. But, to avoid const-poisoning alike problems,
   such type notations can be followed with `:reader`.
   But the `:reader` suffix is a non-sense for such types.
   For example, `[5]struct{a int}` and `[5]struct{a int}:reader`
   have no differences.

**A writer value is assignable to a reader variable,
but a reader value is not assignable to a writer variable.**
This is the most important principle of this proposal.

A notation `v:reader` is introduced to convert a writer value `v` to a reader value,
The `:reader` suffix can only follow r-values (right-hand-side values).

You may have got it, a value hosted at a specified memory address may
represent as a read-only value or a writable value, depending on context.
So a non-final read-only values might be not an immutable value.
(But there are really some non-final read-only values which are immutable values.
Please read the following for such examples.)

#### assignment and conversion rules

Above has mentioned:
* a named final must be bound to a value in its declaration.
  It can't be assigned to again later.
* a writer value is assignable to a reader variable,
  but a reader value is not assignable to a writer variable.
* a writer value can be converted to a reader value, but not vice versa.

#### some examples

An example:
```
{
	var x = []*int{new(int)} // x is a writer variable
	final y = x              // y is a writer final
	var z []*int:reader = x  // z is a reader variable
	final w = y:reader       // w is a reader final
	
	// x, y, z and w share elements.
	
	x[0] = new(int); *x[0] = 123 // ok
	x = nil                      // ok
	println(*z[0])               // 123
	
	y[0] = new(int); *y[0] = 789 // ok
	y = nil                      // error: y is a final
	println(*w[0])               // 789
	
	*z[0] = 555; z[0] = new(int) // error: z[0] and *z[0] are read-only
	z = nil                      // ok
	
	*w[0] = 555; w[0] = new(int) // error: w[0] and *w[0] are read-only
	w = nil                      // error: w is a final
	
	x = z // error: reader value z can't be assigned to writer value x
}
```

In the above, the elements of the slices are not immutable values.
However, in the following example, the slice elements are immutable.
```
{
	var s = []int{1, 2, 3}:reader
	s[0] = 9 // error: s[0] is read-only
	
	// S and its elements are both immutable.
	final S = []int{1, 2, 3}:reader
}
```

More examples:
```
// An immutable error value.
final FileNotExist = errors.New("file not exist"):reader

var n int:reader // error: int is a non-reader type

// Two functions with read-only parameters and results.
// All parameters are varibles.
// Note that "chan int" is a non-reader type.
func Foo(m http.Request:reader, n map[string]int:reader) (o []int:reader, p chan int) {...}
func Print(values ...interface{}:reader) {...}

// Some short declartions. The items on the left sides
// are all variables. No ways to short delcare finals.
{
	oldA, newB := va, vb:reader // newB is a reader variable

	// Explicit conversions.
	// The four lines are equivalent to each other.
	newX, oldY := (Tx:reader)(va), vy
	newX, oldY := (Tx(va)):reader, vy
	newX, oldY := Tx(va:reader), vy
	newX, oldY := Tx(va):reader, vy
}
```

#### about final, reader, read-only, and immutable values

From the above descriptions and explanations, we know:
* a final itself is not only a read-only value, it is also an immutable value.
* a reader value may be a variable or a final,
  so it may be read-only or writable.
* a writer value may be a variable or a final,
  so it may be read-only or writable.
* some read-only values are immutable values, but most are not.

No data synchronizations are needed in concurrently reading immutable values,
but data synchronizations are still needed when concurrently reading
a read-only value which is not an immutable value.

## Detailed rules for values of all kinds of types

#### safe pointers

* Dereference of a reader pointer results a read-only value.
* Dereference of a writer pointer results a writable value.
* Taking address of an addressable final or a reader value results a reader pointer.
* Taking address of an addressable writer value results a writer pointer.

Example:
```
final x = []int{1, 2, 3}

func foo() {
	y := &x  // y is reader pointer variable of type *[]int:reader.
	z := *y  // z is deduced as a reader variable of type []int:reader.
	w := x   // w is deduced as a writer variable of type []int.
	z[0] = 9 // error: z[0] is read-only.
	u := &z  // u is like y.
	
	p1 := &x[1] // p1 is a writer pointer variable.
	p2 := &z[1] // p2 is a reader pointer variable.
	...
}
```

#### unsafe pointers

* Reader pointers and writer pointers can be both converted to unsafe pointers.
  This means the read-only rules built by this proposal can be broken by the unsafe mechanism.
  (This is important for reflection implementation.)

Example:
```
func mut(x []int:reader) []int {
	return *((*[]int)(unsafe.Pointer(&x)))
}
```
#### structs

* Elements of reader struct values are also reader values.
* Elements of writer struct values are also writer values.
* Elements of read-only struct values are also read-only values.
* Elements of writable struct values are also writable values.

#### arrays

* Elements of reader array values are also reader values.
* Elements of writer array values are also writer values.
* Elements of read-only array values are also read-only values.
* Elements of writable array values are also writable values.

#### slices

* Elements of reader slice values are read-only and reader values.
* Elements of writer slice values are writable and writer values.
* We can't append elements to reader slice values.
* Subslice:
  * The subslice result of a reader slice is still a reader slice.
  * The subslice result of a writer slice is still a writer slice.
  * The subslice result of a final or reader array is a reader slice.

Example 1:
```
type T struct {
	a int
	b *int
}

// The type of x is []T:reader.
var x = []T{{123, nil}, {789, new(int)}}:reader

func foo() {
	x[0] = nil   // error: x[0] is read-only.
	x[0].a = 567 // error: x[0] is read-only.
	y := x[0]    // y is a reader value of type T:reader.
	y.a = 567    // ok
	*y.b = 567   // error: y.b is read-only
	y.b = nil    // ok
	z := x[:1]   // liek x, z is reader slice.
	x = nil      // ok
	y = T{}      // ok
	
	final w = x // w is a reader final.
	u := w[:]   // u is a reader slice variable.
	
	// v is a writer slice final.
	final v = []T{{123, nil}, {789, new(int)}}
	v = nil    // error: v is a final
	v[1] = T{} // ok
	
	_ = append(u, T{}) // error: can't append to reader slices
	_ = append(v, T{}) // ok
	
	...
}
```

Example 2:
```
var x = []int{1, 2, 3}

// External packages have no ways to modify elements of x (through S).
final S = x:reader

// The elements of R can't even be modified in current package!
// It and its elements are all immutable values.
final R = []int{7, 8, 9}:reader

// Q itself can't be modified, but its elements can.
final Q = []int{7, 8, 9}
```

Example 3:
```
var s = "hello word"

// "bs" is a reader byte slice.
// A clever compiler will not allocate a
// duplicate underlying byte sequence here.
var bs = []byte:reader(s) // <=> []byte(s):reader
{
	pw := &s[6] // pw is reader poiner whose base type is "byte".
	pw = &bs[6] // ok
}
```

Note, internally, the `cap` field of a reader byte slice is set to `-1`
if the byte slice is converted from a string, so that Go runtime knows
its elements are immutable. Converting such a reader byte slice to
a string doesn't need to duplicate the underlying bytes.

#### maps

* Elements of reader map values are read-only and reader values.
* Elements of writer map values are writable and writer values.
  (Each writable map element must be modified as a whole.)
* Keys (exposed in for-range) of reader map values are reader values.
* Keys (exposed in for-range) of writer map values are writer values.
* We can't append new entries to (or replace entries of,
  or delete old entries from) reader map values.

Example:
```
type T struct {
	a int
	b *int
}

// x and its entries are all immutable values.
final x = map[string]T{"foo": T{a: 123, b: new(int)}}:reader

bar(x) // ok

func bar(v map[string]T:reader) { // v is a reader variable
	// Both v["foo"] and v["foo"].b are both reader values.
	*v["foo"].b = 789 // error: v["foo"].b is read-only
	v["foo"] = T{}    // error: v is a reader map
	v["baz"] = T{}    // error: v is a reader map
	
	// m will be deduced as a reader map variable.
	// That means as long as one element or one key is a reader
	// value in a map literal, then the map is also a reader value.
	m := map[*int]*int {
		new(int): new(int):reader,
		new(int): new(int),
		new(int): new(int),
	}
	for a, b := range m {
		// a and b are both reader values of type *int:reader.
		
		*a = 123 // error: *a is read-only
		*b = 789 // error: *b is read-only
	}
}
```

#### channels

* Send
  * We can't send values to final channels.
  * We can send values of any genres to a reader channel.
  * We can only send writer values to a writer channel.
* Receive
  * We can't receive values from final channels.
  * Receiving from a reader channel results a reader value.
  * Receiving from a writer channel results a writer value.

Example:
```
final ch = make(chain *int, 1)

func foo(c chan *int:reader) {
	x := <-c // ok. x is a reader variable of type *int:reader.
	y := new(int)
	c <- y // ok

	ch <- x // error: ch is a final channel
	<-ch    // error: ch is a final channel
	...
}
```

#### functions

Function parameters and results can be declared as reader variables.

In the following function proptotype, parameter `x` and result `w` are declared as reader variables.
```
func fa(x Tx:reader, y Ty) (z Tz, w Tw:reader) {...}
```

A `func()(T)` value is assignable to a `func()(T:reader)` value, not vice versa.

A `func(T:reader)` value is assignable to a `func(T)` value, not vice versa.

To avoid function duplications like the following code shows:
```
// split writer byte slices
func Split_1(v []byte, sep []byte:reader) [][]byte {...}

// split reader byte slices
func Split_2(v []byte:reader, sep []byte:reader) ([][]byte:reader) {...}
```

A role parameter concept is introduced,
so that the above two function can be declared as one:
```
func Split(v []byte::q, sep []byte:reader) ([][]byte::q) {...}
```

Here, `:q` is called a role parameter.
Its name can be arbitrary non-blank identifier,
but the two occurrences must be consistent.
Short role parameter names are recommended, such as `p` and `q`.

Use the `Split` function.
```
{
	var x = []byte{"aaa/bbb/ccc/ddd"}
	_ = Split(x, []byte("/"))        // call the writer version
	_ = Split(x:reader, []byte("/")) // call the reader version
	
	// Use Split function as values.
	var fw = Split{r: writer} // I haven't got better syntax yet.
	var fr = Split{r: reader}
	_ = fr(x, []byte("/")) // <=> Split(x:reader, []byte("/"))
	                       // x is converted to a reader value implicitly.
}
```

#### method sets

The method set of reader type `T:reader` is always a subset of writer type `T`.
This means when a method `M` is explicitly declared for reader type `T:reader`,
then a corresponding implicit method with the same name
will be declared for writer type `T` by compilers.
```
func (T:reader) M() {} // explicitly declared. (A reader method)
/*
func (t T) M() {t:reader.M()} // implicitly declared. (A writer method)
*/

var t T
t.M()
// <=>
t:reader.M()
// <=>
T:reader.M(t:reader)
// <=>
T:reader.M(t)
// <=>
T.M(t)
```

In the above code snippet, the method set of reader type `T:reader` contains one method: `reader.M`,
however the method set of type `T` contains two method: `reader.M` and `M`.

For type `T` and `*T`, if methods can be declared for them (either explicitly or implicitly),
the method set of type `T:reader` is a subset of type `*T:reader`.
(Or in other words, the method set of type `T` is a subset of type `*T`
if type `T` is not an interface type.)

#### interfaces

An interface type can specify some read-only methods. For example:
```
type I interface {
	M0(Ta) Tb // a writer method

	reader.M2(Tx) Ty // a reader method (also called receiver-read-only method).
	                 // NOTE: this is an exported method.
}
```

Similar to non-interface type, if a reader interface type
explicitly specified a read-only method `reader.M`,
it also implicitly specifies a writer method with the same name `M`.

The method set specified by type `I` contains three methods, `M0`, `reader.M2` and `M2`.
The method set specified by type `I:reader` only contains one method, `reader.M2`.

When a method is declared for a concrete type to implement a reader method,
the type of the receiver of the declared method must be a reader type.
For example, in the following code snippet,
the type `T1` implements the interface `I` shown in the above code snippet,
but the type `T2` doesn't.
```
type T1 struct{}
func (T1) M0(Ta) Tb {var b Tb; return b}
func (T1:reader) M2(Tx) Ty {var y Ty; return y} // the receiver type is a reader type.

type T2 struct{}
func (T2) M0(Ta) Tb {var b Tb; return b}
func (T2) M2(Tx) Ty {var y Ty; return y} // the receiver type is a writer type.
```

Please note, the type `T3` in the following code snippet also implements `I`.
Please read the above function section for reasons.
```
type T3 struct{}
func (T3) M0(Ta:reader) Tb {var b Tb; return b}
func (T3:reader) M2(Tx:reader) Ty {var y Ty; return y}
```

If a writer type `T` implements a writer interface type `I`,
then the reader type `T:reader` also implements the reader interface type `I:reader` for sure.

* Dynamic type
  * The dynamic type of a writer interface value is a writer type.
  * The dynamic type of a reader interface value is a reader type.
* Box
  * No values can be boxed into final interface values (except the initial bound values).
  * reader values can't be boxed into writer interface values.
  * Values of any genres can be boxed into a reader interface value.
* Assert
  * A type assertion on a reader interface value results a reader value.
    For such an assertion, its syntax form `x.(T:reader)` can be simplified as `x.(T)`.
  * A type assertion on a writer interface value results a writer value.

For this reason, the `xyz ...interface{}` parameter declarations of all the print functions
in the `fmt` standard package should be changed to `xyz ...interface{}:reader` instead.

Role parameters don't work for receiver parameters.

Example:
```
var x = []int{1, 2, 3}
var y = [][]int{x, x}:reader
var u interface{} = x        // ok
u = y                        // error: can't assign a reader value to a writer value.
var v interface{} = y        // ok. v is deduced as a reader interface value.
var w = v.([][]int)          // ok. Like y, w is a reader value of type [][]int:reader.
v = x                        // ok
var s = v.([]int)            // ok, u is a reader value of type []int:reader.
var t = v.([]int:reader)     // ok, equivalent to the above one.
var t = u.([]int:reader)     // ok, Assert + convert.
var r = u.([]int):reader     // ok, Assert then convert. Equivalent to the above one.
```

Another eample:
```
type T0 []int
func (T0) M([]int) []int

type T1 []int
func (T1) M([]int:reader) []int

type T2 []int
func (T2) M([]int) ([]int:reader)

type T3 []int
func (T3) M([]int:reader) ([]int:reader)

type I interface {
	M([]int) []int:reader
}

// T0, T1, T2, and T3 all implement I.
var _ I = T0{}
var _ I = T1{}
var _ I = T2{}
var _ I = T3{}
```

#### reflection

Many function and method implementations in the `refect` package should be modified accordingly.
The `refect.Value` type shoud have a **_reader_** property,
and the result of an `Elem` method call should inherit the **reader** property
from the receiver argument. More about reflection.
For all details on reflection, please read the following reflection section.

The current `reflect.Value.CanSet` method will report whether or not a value can be modified.

A `reflect.ReaderValueOf` function is needed to create
`reflect.Value` values representing reader Go values.
Its prototype is
```
func ReaderValueOf(i interface{}:reader) Value
```
For the standard Go compiler, in implementaion,
one bit should be borrowed from the 23+ bits method number
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
The method sets of reader type `T:reader` is the subset of the writer type `T`.
Their respective other properties should be identical.

A method `reflect.Type.Genre` is needed,
it may return `Reader` or `Writer` (two constants).


## Compiler implementation

I'm not familiar with the compiler development things.
It is just my feeling, by my experience,
that most of the rules mentioned in this proposal
can be enforced by compiler without big technology obstacles.

At compile phase, compiler should maintain two bits for each value.
One bit means whether or not the value itself can be modified.
The other bit means whether or not the values referenced by the value can be modified.

## Runtime implementation

Except the next to be explained reflection section, the impact on runtime
made by this proposal is not large.

Each internal method value should maintain a `reader` property.
This information is useful when boxing a reader value into an interface.

As above has mentioned, the cap field of a reader byte slice should
be set to `-1` if its byte elements are immutable.

## Rationales

#### Rationales of Go needs read-only and immutable values

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

#### Rationales for the `T:reader` notation

1. It is less discrete than `reader T`. I think `func (Ta:reader) (Tx:reader)` has a better readibility than `func (reader Ta)(reader Tx)`.
1. It conforms to Go type literal design philosophy: more importants shown firstly.
1. It saves one keyword.

#### About the problems of read-only values mentioned by Russ

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

Solved by letting `interface {M(T:reader)}` implement `interface {M(T)}`
and letting `interface {M() T}` implement `interface {M() T:reader}`.

Please see [the interface section](#interfaces) for details.

#### About partial read-only

Sometimes, people may need partial read-only for struct values.
[An older revision](README-v7.md#structs) of this proposal supports this feature,
but it is removed from the currrent revision for it brings many complexisites
and may cause some design flaws.

```
type Counter struct {
	mu sync.Mutex:writable // will be always writable (and also a writer),
	                       // even if its containing struct value is a read-only.
	n  uint64
}

{
	final c Counter
	c.mu.Lock() // error: c.mu is read-only

	var p = &c  // p is a reader pointer of type *Counter:reader
	p.mu.Lock() // ok by the rule, but it makes c become a non-immutable value,
	            // which may cause some confusions.
	            // To avoid such cases happening, we can forbid taking addresses
	            // of finals, but I feel this trade-off isn't worth it.
}
```

To support partial read-only, the following rules need to be added:
1. finals are always unaddressable.
1. values of `struct{t T:writable}` can be converted/assignable to `struct{t T}`.
1. function values
	1. values of `func (struct{t T})` can be converted/assignable to `func (struct{t T:writable})`.
	1. values of `func () struct{t T:writable}` can be converted/assignable to `func () struct{t T}`.
	1. values of `func (struct{t T})` and `func (struct{t T:writable}):reader` can't be converted to each other.

Another simpler rule design is to forbid the conversions mentioned in the 2nd and 3rd rules.

This means, the addressable final feature and the partial read-only feature are mutually exclusive.
I prefer keeping the addressable final feature.
This feature will break less user code,
for some user code may take the addresses of many error values declared in std packages.
This feature will continue to make such code valid.

#### The problem when reader parameters in a library package changed to writers.

A reader parameter in a library package may be changed to a writer parameter in a newer version.
For example:

```
// v1.0.0
package apkg

func F(s []byte:reader) {
	// elements of s can't be modified.
}
```

and

```
// v1.1.0
package apkg

func F(s []byte) {
	// elements of s may be modified.
}
```

The following code in caller package will keep valid after the change,
this may cause some unexpected behaviors.

```
package main

import "x.y,z/apkg"

fnc main() {
	...
	
	s := []byte{1, 2, 3}
	apkg.F(s)
	...
}
```

A professional programmer should write the above code as
```
package main

import "x.y,z/apkg"

fnc main() {
	...
	
	s := []byte{1, 2, 3}
	apkg.F(s:reader)
	...
}
```

so that the code will fail to compile after the library change.

However, this makes some inconveniences in programming.


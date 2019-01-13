I have made [another read-only proposal](https://github.com/golang/go/issues/29392), in which read-only is a property of types. Although I tried my best to make that proposal as simple as possible, I still think that proposal is too complex to be thoroughly and clearly understood by most gophers, so here I would make a new even simpler one.

In the new proposal, read-only is a property of values instead of types.

A new `fixed` keyword and a modifier `!` are introduced in this proposal. So this proposal is not Go 1 compatible. (**[update]: This proposal is possible to be revised to be Go 1 compatible. Please see the following comments for details.**)

The implementations for this proposal are feasible and predictable. The code change should be relatively small but it solves most of the lacking immutability problems in Go. Yes, it doesn't solve all the lacking immutability problems, but it solves all of the important ones, with a low cost (development) and a low risk (minimal impact on existing code). 

**[update]: I just realized that there is [a doc](https://docs.google.com/document/d/1-NzIYu0qnnsshMBpMPmuO21qd8unlimHgKjRD9qwp2A) written by Russ. It looks this proposal satisfies most of the requirements mentioned in that doc well.**

To understand the proposal, we need learn some concepts firstly.

### Value properties and value genres

Currently, each Go value has two properties
1. whether or not that value itself is addressable.
1. whether or not that value itself is modifiable (can be assigned to).

This proposal will add a new value property `ref_modifiable`: whether or not values referenced (either directly or indirectly) by the value are modifiable.

The permutation of the three properties results eight genres of values
1. `{unaddressable, self_unmodifiable, ref_unmodifiable}`. 
   Such as constants and string bytes.
1. `{unaddressable, self_unmodifiable, ref_modifiable}`. 
   Such as slice literals.
1. `{unaddressable, self_modifiable, ref_unmodifiable}`.
   No such Go values currently.
   *And not to be supported in this proposal.*
1. `{unaddressable, self_modifiable, ref_modifiable}`.
   Such as map elements.
1. `{addressable, self_modifiable, ref_unmodifiable}`.
   No such Go values currently.
   **This proposal will make Go support this genre of values (which are declared with `fixed`).**
1. `{addressable, self_modifiable, ref_modifiable}`.
   Such as variables.
1. `{addressable, self_unmodifiable, ref_unmodifiable}`.
   No such Go values currently.
   **This proposal will make Go support this genre of values (which are declared with `fixed!`).**
1. `{addressable, self_unmodifiable, ref_modifiable}`.
   No such Go values currently.
   *And not to be supported in this proposal.*
   _**[update]: this genre of values might also be supported in the proposal. See [the below comment](https://github.com/golang/go/issues/29422#issuecomment-451688197) for details.**_ (But the current formal proposal doesn't support it.)

**[update]**: _sorry, in fact, the property "**whether or not that value itself is addressable**" is not important for this proposal. But I'm lazy to remove it from the above and the below descriptions. This proposal talks about only 4 genres of values actually._

### The proposal

As the last section has suggested, two new genres of value will be supported in this proposal.
1. Values with properties: `{addressable, self_unmodifiable, ref_unmodifiable}`.
   Such values will be called specifically as **both-immutable values**,  which means both themselves and the values referenced by them are self_unmodifiable. Such values **must be declared with `fixed!` instead of `var`**.  `fixed!` should be used to declare some global error values. 
1. Values with properties: `{addressable, self_modifiable, ref_unmodifiable}`.
   Such values are called **ref-immutable values**. Such values **must be declared with `fixed` instead of `var`**. A ref-immutable value is modifiable but the values referenced by them are unmodifiable. `fixed` should be used to declare some immutable parameters and results.

**A both-immutable value must be bound a value in its declaration**. After the declaration, it can never be assigned any more. (Copies of, in fact) values of any genre can be bound to a both-immutable value, including constants, variable, literal, ref-immutable, or another both-immutable value.

(Another design is **only values which respective reference trees don't reference values out of the tree** can be bound to both-immutable values. Which design is chosen doesn't affect the following to be described rules.)

A ref-immutable value can be declared without an initial value. Same as both-immutable values, values of any genre can be assigned to a ref-immutable value, **including both-immutable values**.

When **_an immutable value_** is mentioned below, it may be a both-immutable value or a ref-immutable value.

Please note that, although a value **can't be modified through the (either both- or ref-) immutable values which are referencing it**, it **can be modified through other mutable values which are referencing it**. (Yes, this proposal doesn't solve all problems.)

There are three different design ideas for immutable-to-mutable assignments:
1. **any such assignments are disallowed**. (The recommended design.)
1. if the concrete type of the source value is a basic type (or any non-referencing type), then such an assignment is allowed.
1. nilify all referencing in source values in such assignments. (For simplicity, maybe nilfying the first-level direct referencing is acceptable. Nilifying any-level referencing may be cost expensive for some special cases.)

(The second and third designs may be useful for some cases but they are easy to cause many confusions.)

The proposal can really end here. The following are just the detailed rules for values of all kinds of types (assuming the first immutable-to-mutable assignment design idea is adopted). These rules are much straightforward and anticipated. **They are derived from the above basic rules.**

### Detailed rules of this proposal

Dereferences of immutable pointers are both-immutable values.  An immutable value may be addressable. Addressable immutable values can be taken addresses. Their addresses are ref-immutable pointer values.

Fields of ref-immutable struct values are ref-immutable values. Fields of both-immutable struct values are both-immutable values.

Elements of ref-immutable array values are ref-immutable values. Elements of both-immutable array values are both-immutable values.

Elements of immutable slice values are both-immutable values. We can't append new elements to immutable slice values. The subslice result of an immutable slice is still an immutable slice.

Elements of immutable map values are both-immutable values. We can't append new entries to (or replace entries of, or delete old entries from) immutable map values.

We can send values to a ref-immutable channel. Receiving from a ref-immutable channel results an immutable value. Yes, we can send values to (and received values from) ref-immutable channels. However, we can't send immutable values to mutable channels, and we can't send values to (or receive values from) both-immutable channels.

Function parameters and results can be declared as immutables (either `fixed` or `fixed!`), including receiver parameters. For the callers of a function, parameters/results declared with `fixed` have no differences to parameters/results declared with `fixed!`. **So the `fixed` keywords should be parts of a function prototype, but `!`s should not.** 

A `func(fixed T)` value is assignable to a `func(T)` value, not vice versa. A `func()(T)` value is assignable to a `func()(fixed T)` value, not vice versa.

**Every type has two method sets, one for mutable values, one for immutable values.** The immutable one is a subset of the mutable one.  For type `T` and `*T`, if methods can declared for them (either explicitly or implicitly), then the immutable method set of `T` is a subset of the immutable method set of `*T`.

Immutable values can't be boxed into a mutable interface value or a both-immutable interface value. it can only be boxed into a ref-immutable interface value. A mutable value can also be boxed into an immutable interface values (**as long as the immutable method set of its type implements the interface**). A type assertion on an immutable interface value results an immutable value. 

For this reason, the `xyz interface{}` parameter declarations of all the print functions in the `fmt` standard package should be changed to `fixed xyz interface{}` instead.

When an immutable value is assigned to a new declared variable with short declaration form, the new declared variable will be deduced as a ref-immutable value.

(**All above mentioned rules are enforced by compiler.**)

Many function and method implementations in the `refect` package should be modified accordingly. For example, a **_refImmutable_** property should be added for the `refect.Value` type, and the result of an `Elem` method call should inherit the **_refImmutable_** property from the receiver argument. More about reflection. please read the part following the below example.

An example:
```golang
var x = []int{1, 2, 3}
fixed y [][]int
y = [][]int{x, x} // ok

x[1] = 123     // ok
y[0][1] = 123  // error
var z = y[0]   // error
fixed z = y[0] // ok
z[0] = 123     // error

// The following line <=> fixed p = &z[0]
p := &z[0]     // ok. p is an immutable value.
*p = 123       // error
x[0] = *p      // ok
p = new(int)   // ok

fixed v interface{} = y
var w = v.([][]int)   // error
fixed w = v.([][]int) // ok
v = x                 // ok

// S is exported, but external packages have
// no ways to modify x and S (through S).
fixed! S = x     // ok.
S = x            // error
t := S[:]        // ok. <=> fixed t =  s[:]
_ = append(t, 4) // error

// The elements of R even can't be modified in current package!
fixed! R = []int{7, 8, 9}
```

### More about reflection

A `reflect.FixedValueOf` function is needed to create `reflect.Value` values for immutable Go values. Its prototype is
```golang
func FixedValueOf(fixed i interface{}) Value
```

For all the current methods of `reflect.Value`, their reciever parameter should be changed to immutable. The `reflect.Value` return results **should NOT** be changed to immutable.

A `reflect.Value.ToFixed` method is needed.

A `reflect.Value.FixedInterface` method is needed, it returns an immutable interface value. The old `Interface` method panics on immutable values.

Three methods `reflect.Type.NumFixedMethods`, `reflect.Type.FixedMethodByName` and `reflect.Type.FixedMethod` are needed.


## Rationales of Go needs read-only and immutable values

In [ediscretevaluation of read-only slices](https://docs.google.com/document/d/1-NzIYu0qnnsshMBpMPmuO21qd8unlimHgKjRD9qwp2A),
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

## Rationales for the `T:reader` notation.

1. It is less discrete than `reader T`. I think `func (Ta.reader) Tx.reader` has a better readibility than `func (reader Ta)(reader Tx)`.
1. It conforms to Go type literal design philosophy: more importants shown firstly.

## About the drawbacks of read-only values mentioned by Russ.

In [ediscretevaluation of read-only slices](https://docs.google.com/document/d/1-NzIYu0qnnsshMBpMPmuO21qd8unlimHgKjRD9qwp2A),
Russ mentions some drawbacks of read-only values.
This proposal provides solutions to avoid each of these drawbacks.

#### function duplication

Solved by role parameter.

Please see the end of [the function section](README.md#functions).

#### immutability and memory allocation

Solved by setting the capacities of immutable byte slices as `-1` at run time.

Please see the end of [the slice section](README.md#slices).

#### loss of generality

Solved by letting `interface {M(T:reader)}` inplement `interface {M(T)}`
and letting `interface {M() T}` inplement `interface {M() T:reader}`.

Please see [the interface section](README.md#interfaces) for details.


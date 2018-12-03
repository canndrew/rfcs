- Feature Name: default-types
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Have the compiler choose a "default" type in more situations where it can't
otherwise infer a type. Allow the user to control this "defaulting" process
through default type declarations of the form:

```rust
default SomeTrait = SomeType;
```

# Motivation
[motivation]: #motivation

This change will allow the compiler to be more liberal in accepting
type-ambiguous code, without causing it to generate programs that behave in
unexpected ways.

For example, this code will not compile in today's Rust:

```rust
fn make_ok() -> Result<(), impl Error> {
    Ok(())
}
```

The reason being that the error type of the result is unspecified. In this
situation though, since the caller isn't able to assume any specific error
type, the specific error type should not matter to their code. Also, since the
error type is never instantiated or used, *any* type which implements `Error`
would result in an almost identical program. As such, the compiler should be
allowed to take the liberty here of choosing a suitable type. The most suitable
type in this case is `!` since it reflects the fact that the error is never
constructed. Choosing `!` also allows the compiler to generate the most
efficient code since the `Result` can be represented as a ZST, branches that
match on the error can be elided, etc.

This RFC puts forth the notion that there is no downside to the compiler just
using `!` in situations like this. Rather, it allows the compiler to be less
whiny about errors while producing programs that do what the programmer said,
almost certainly what they intended, and which have no hidden costs or side-effects.
Additionally I propose that there are other situations where the compiler should
choose types other than `!`, (specifically where the chosen type needs to
satisfy certain trait bounds) and that the user should be able to guide the
compiler towards a sensible choice. This RFC proposes a mechanism
through which the user can do this.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are some cases where Rust code can leave a type unspecified. Some examples:

```rust
// The type of the error is unspecified.
fn make_ok() -> Result<(), impl Error> {
    Ok(())
}

// The type of `_unused` is unspecified.
let _unused = Default::default();

// The return type is unspecified.
fn make_future() -> impl Future<Output = u32> {
    unimplemented!()
}
```

In these situations the compiler will attempt to choose a suitable "default"
type. This is possible since, in order for the type to be unconstrained by type
inference, the type must not be being used in a way which assumes any specific
type.

## Examples of default types

### The never type `!`

The most common type to default to is `!`. If the unspecified type
is never constructed (as is usually the case, otherwise the type would not be
unspecified) then `!` is probably the simplest type which satisfies all the
constraints which the surrounding code places on it.

### The `NeverOutput<T>` type

Sometimes the unspecified type is required to be able to produce values of some
other type via a trait such as `Iterator` or `Future`. For these situations
the standard library offers the type:

```rust
struct NeverOutput<T> {
    _ph: PhantomData<T>,
    _never: !,
}
```

This type, like `!`, can never be constructed. However by virtue of having a
type parameter it is able to implement traits with an associated output type
for all values of that type. In the case of `Future`, for instance,
`NeverOutput<T>` comes equipped with the following `impl`:

```rust
impl<T> Future for NeverOutput<T> {
    type Output = T;

    fn poll(self: Pin<&mut Self>, lw: &LocalWaker) -> Poll<T> {
        self._never
    }
}
```

This allows `NeverOutput<T>` to be used as the default type wherever there's
an unconstrained type which must satisfy `Future<Output = T>`. It is important
to note that this `impl` is vacuous - the `poll` method can never be called.
The type and trait exist purely to make the code pass the type-checker without
introducing any side-effects to the program.

### Non-empty default types

It's not always true that an unspecified type is never constructed since some
traits are able to create instances of their implementing type. A simple example
is `Default`.

```rust
let _unused = Default::default();
```

Suppose we write a program which contains the above line of code. If there are
no other constraints on the type of `_unused` (say, because the line of code
was outputted by a macro and the value is never used) then any type with a
default value will likely yield a program which faithfully realizes the code.
This won't be the case if the default method itself is intended to have
side-effects, though this is unlikely given that `default` takes no arguments,
the value is unused, and the type was unspecified. It would probably be quite a
perverse use of the `Default` trait if calling `default()` then immediately
dropping the resulting value intentionally caused IO or something. In this case
the simplest type which satisfies `Default` is `()`, since it does nothing and
can be constructed without side-effects.  As with the previous examples that
used `!` and `NeverOutput<T>`, using `()` here effectively allows the compiler
to strip dead code from the program.

### Non-empty and effectful default types

Suppose we have an iterator that produces values of type `(A, B)`. We want to
extract all the `B` values from the iterator into an owned container and
then iterate over them separately. To do this we might write the following
code.

```rust
let vals = my_iter.into_iter().map(|(_a, b)| b).collect();
...
for val in vals {
    ...
}
```

In this code the type of `vals` has not been specified, all that we know is
that it is required to implement `Default + Extend<B> + IntoIterator<Item =
B>`. The canonical type which satisfies these constraints is, arguably,
`Vec<B>`.  Collecting into a `Vec<B>` will result in the exact same values
being returned from the iterator in the exact same order. No values will get
dropped and no extra constraints are placed on the type of `B` (such as that it
implement `Hash` or `PartialEq`). Although there are other types with behaviour
isomorphic to `Vec<B>` (eg. such as `LinkedList<B>`) `Vec<B>` is highly likely
to be the optimal choice.

This use of default types is likely to be more controversial than the others in
this RFC since the choice of type actually effects the behaviour of the
program. It can be argued though that there's one specific behaviour which is
more straight-forward than any other possible realisation of the given code. If
we recognize that `Default + Extend<B>` and `IntoIterator<Item = B>` are duals
of each other then we can see that one of the most basic and commonly used Rust
types (`Vec<B>`) provides implementations of these dual traits which neatly
cancel each other out.

## Overriding default types

In order for the compiler to choose sensible default types, we need to be able
to tell it what those sensible choices are. We do this by using default type
declarations, which say what type to use given a set of trait constraints. The
standard library provides at least the following declarations:

```rust
default Default = ();
default<T> Future<Output = T> = NeverOutput<T>;
default<T> IntoIterator<T> = NeverOutput<T>;
default<T> Default + Extend<T> + IntoIterator<T> = Vec<T>;
```

Default type declarations can specialize prior default type declarations by
referring to a more specific trait. We see this above with the `Default +
Extend<T> + IntoIterator<T>` declaration overriding `IntoIterator<T>`. As such,
default type declarations form a tree with the base of the tree being the
built-in default type `default ?Sized = !`.

When the compiler needs to choose a type which satisfies some trait, it will
look at all the default type declarations which refer to a sub-trait of that
trait, from most specific to least, and choose the first type which satisfies
the required trait. For example, suppose we have:

```rust
default TraitA = A;
default TraitA + TraitB = B;
```

If the compiler then needs to find a type which satisfies `TraitA + TraitB +
TraitC` it will:

1.  Check if `B: TraitA + TraitB + TraitC`. Although `B` must implement `TraitA
    + TraitB` it might not implement `TraitC` and so may not be an appropriate
    choice.

2.  Check if `A: TraitA + TraitB + TraitC`.

3.  Check if `!: TraitA + TraitB + TraitC`.

4.  If none of the above options work, fail with an error message saying that
    an appropriate type could not be inferred. This is the same error message
    we see today, except that notes should be added to the compiler output
    mentioning that `B`, `A` and `!` were each considered and explaining why
    each was not suitable.

On the subject of error messages, it's also important that any types chosen via
default type declarations should be marked as such by the compiler so that
their origin can be explained if they ever appear in error messages. (Note
though that they by their nature they should never themselves be the cause of
type errors).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

# Drawbacks
[drawbacks]: #drawbacks

This increases the complexity of the language by adding another type of
declaration.

This change could hypothetically result in code that misbehaves rather than
just failing to compile, at least where non-uninhabited types are chosen as
defaults (as in the `Vec<T>` example).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* Continue to default to `!` in some cases and to raise an error in others.
* Try to default to `!` in all situations rather than just some as Rust does
  today.
* Run with the proposal but only use uninhabited types as defaults, don't
  include defaults like `()` or `Vec<T>`.
* Somehow make `!` able to implement `Future<Output = T>` for all `T` so that
  it can be used as the default in more situations.

# Prior art
[prior-art]: #prior-art

None that I'm aware of.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Are there any corner cases to this idea that haven't been considered?

# Future possibilities
[future-possibilities]: #future-possibilities

None really, this RFC is fairly self-contained.


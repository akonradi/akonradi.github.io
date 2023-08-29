---
layout: post
title: Modeling per-type behavior differences in Rust with enums
---

Rust's first-class support for `enum` types lets us use them to model variation
between behavior on types. By using associated types with trait bounds and
uninstantiable types, we can reduce code duplication without compromising
performance.

## Enum types in Rust

Rust has plenty of neat features, but one of the really useful ones is the
ability to write and exhaustively match on [`enum` types][Rust book enums]. For
the more theory-inclined, Rust enums are "sum types", which along with `struct`s
("product types") make up Rust's support for [algebraic data types]. Unlike
C++'s [`std::variant`], whose cases are handled with a visitor object passed to
[`std::visit`] Rust's `enum` types have first-class language support for pattern
matching and case handling.

One of the things we can do with `enum` types is represent control flow. The
standard library's [`std::ops::ControlFlow`] type, for instance, can be used to
indicate whether a loop should continue running or exit early. The
[`std::task::Poll`] type indicates whether an `async` block or function has
completed or needs to be polled again. The [`Result`] type is a staple of Rust
code; a `Result<T, E>` value indicates whether a successful `T` value or an
error of type `E` needs to be handled.

Values of `enum` types are generally handled via [`match`] expressions, which
allows the programmer to exhaustively list the possible value patterns and write
handling code for each one. By providing an `enum` type as the return type, a
function can ensure that the caller has to at least consider every possible
shape of output value. That means that a function that returns a `Result` is
required (by the language) to make a conscious choice for how to handle a
`Result::Err(e)` value (even if that choice is to ignore it).

The decision of which case is being handled is usually made at run time, but
there's no reason it has to be; Rust's `match` works just as well in a [`const`]
block, and if the compiler detects (possibly after inlining) that only one match
arm can ever be entered, it [won't emit code](https://godbolt.org/z/3WdTcf4MT)
to handle the other ones.

## Generic code with traits

The usual way to handle compile-time polymorphism in Rust is to rely on traits,
and to define methods that comprise an interface for working with multiple concrete
types. Implementing the same trait for multiple types lets the programmer write
different implementations for the same interface. Meanwhile, generic code is
written with access only to methods and types from the trait definition. The
compiler ensures that when the code is run, the generic code uses the
definitions from the trait implementation for the type parameter.

The downside of this approach is apparent when the trait implementations are
substantially similar. Consider this example:

```rust
/// Generic food item that can be prepared.
pub trait Food {
    /// Create a new instance of `Self`.
    fn prepare() -> Self;
}

struct Pizza { /* ... */ }
struct Calzone { /* ... */ }

impl Food for Pizza {
    fn prepare() -> Self {
        let dough = Flour::new().add(Water::new()).knead();
        let sauce = Sauce::new();
        let toppings = vec![Pepperoni::new(), Mushroom::new()];
        let flat_dough = dough.stretch();
        let unbaked = flat_dough.apply(sauce).apply(toppings);
        let baked = unbaked.bake();
        Self::from_sliced(baked.cut_into_slices())
    }
}

impl Food for Calzone {
    fn prepare() -> Self {
        let dough = Flour::new().add(Water::new()).knead();
        let sauce = Sauce::new();
        let filling = vec![Pepperoni::new(), Mushroom::new()];
        let (dough, extra_dough_balls) = dough.cut_into_balls().split_first().unwrap();
        extra_dough_balls.save_for_later();
        let flat_dough = dough.roll();
        let unbaked = flat_dough.apply(sauce).apply(filling).fold_over();
        Self::from_baked(unbaked.bake())
    }
}
```

The code above defines two types of food, `Pizza` and `Calzone`, and implements
the `Food` trait for each by providing an implementation of the one trait
method, `Food::prepare`.

## Modeling variation with control flow

If we look at the code above, we can see that most of the code in the trait
implementations is duplicated. We could reduce that by extracting common code,
for example:

- We could extract the shared preamble to a private helper function, though
  we'd still end up with some un-shared bits of code along with a
  `fn prepare_dough_and_sauce_and_filling() -> (Dough, Sauce, Vec<Topping>)`.
- We could call a helper function that takes a callback, something like
  `fn with_prepped(f: impl FnOnce(Dough, Sauce, Vec<Topping>) -> Self) -> Self`,
  with type-specific closures, though we'd still end up with duplication in the
  closures.

Instead of trying to extract common code, though, let's try to model the
differences between our types as values of an `enum` type. That will let us use
standard control flow operations, like `match`, to write a generic
implementation that handles all possible cases. We'll need to introduce some
helper code:

```rust
trait Make {
    type Output;
}

trait MakePizza: Make {
    fn stretch_dough(&self, dough: Dough) -> FlatDough;
    fn from_sliced(self, slices: SlicedPizza) -> Self::Output;
}

trait MakeCalzone: Make {
    fn cut_and_roll_dough(&self, dough: Dough) -> FlatDough;
    fn from_baked(self, baked: BakedCalzone) -> Self::Output;
}

trait PizzaOrCalzone {
    type MakeP: MakePizza<Output=Self>;
    type MakeC: MakeCalzone<Output=Self>;

    fn pizza_or_calzone() -> EitherPizzaOrCalzone<Self::MakeP, Self::MakeC>;
}

enum EitherPizzaOrCalzone<P, C> {
    Pizza(P),
    Calzone(C),
}
```

The `MakePizza` and `MakeCalzone` traits above let us access pizza- or
calzone-specific functionality, respectively. The `PizzaOrCalzone` trait has two
associated types, one for each of these traits. We can write the implementation
of the `Food` trait as a blanket implementation over all types that implement
`PizzaOrCalzone` by matching on the output of the `pizza_or_calzone()` trait
method.

```rust
impl <PorC: PizzaOrCalzone> Food for PorC {
    fn prepare() -> Self {
        let mut dough = Flour::new().add(Water::new()).knead();
        let sauce = Sauce::new();
        let toppings = vec![Pepperoni::new(), Mushroom::new()];

        let pizza_or_calzone: EitherPizzaOrCalzone<Self::MakeP, Self::MakeC> =
            Self::pizza_or_calzone();
        let flat_dough = match &pizza_or_calzone {
            EitherPizzaOrCalzone::Pizza(pizza_base) => pizza_base.stretch_dough(dough),
            EitherPizzaOrCalzone::Calzone(calzone_base) => calzone_base.cut_and_roll_dough(dough)
        };
        let unbaked = flat_dough.apply(sauce).apply(toppings);

        match pizza_or_calzone {
            EitherPizzaOrCalzone::Pizza(pizza_base) => {
                let baked = unbaked.bake();
                pizza_base.from_sliced(baked.cut_into_slices())
            }
            EitherPizzaOrCalzone::Calzone(calzone_base) => {
                let unbaked = unbaked.fold_over();
                let baked = unbaked.bake();
                calzone_base.from_baked(unbaked)
            }
        }
    }
}
```

We get the benefit of having all our code in one place, and we get to
handle the differences using normal control-flow constructs, relying on Rust's
requirement for exhaustive matching on `enum` types to ensure we handled all
cases. Note that this method returns an instance of `PorC` from the final
`match` expression, which means its two branches need to produce values of the
same type (`PorC`). The output values are the result of calling
either `MakePizza::from_sliced` or `MakeCalzone::from_baked`. Since both of
these return an instance of `Make::Output` from the implementing type, and
we've constrained the associated type to `Output=Self` for both `MakeP` and
`MakeC` in `PizzaOrCalzone`, the compiler knows that for either branch, the
output value is an instance of `Self = PorC`.

The only thing that's left is to implement `PizzaOrCalzone` for our `Pizza` and
`Calzone` types.

## Defining the associated types

`PizzaOrCalzone` is a simple trait, and it's pretty straightforward to fill in
the method definitions for the `Pizza` and `Calzone` implementations. But what
do we use as the associated types that implement `MakePizza` and `MakeCalzone`?
The first half is easy: we can define types `MakePizzaImpl` and
`MakeCalzoneImpl` that implement `MakePizza` and `MakeCalzone`, respectively:

```rust
struct MakePizzaImpl;

impl Make for MakePizzaImpl {
    type Output = Pizza;
}

impl MakePizza for MakePizzaImpl {
    fn stretch_dough(&self, dough: Dough) -> FlatDough {
        dough.stretch()
    }
    fn from_sliced(self, slices: SlicedPizza) -> Pizza {
        Pizza::from_sliced(slices)
    }
}

struct MakeCalzoneImpl;

impl Make for MakeCalzoneImpl {
    type Output = Calzone;
}

impl MakeCalzone for MakeCalzoneImpl {
    fn stretch_dough(&self, dough: Dough) -> FlatDough {
        let (dough, extra_dough_balls) = dough.cut_into_balls().split_first().unwrap();
        extra_dough_balls.save_for_later();
        dough.roll()
    }
    fn from_baked(self, baked: BakedCalzone) -> Calzone {
        Calzone::from_baked(baked)
    }
}
```

We can see that `MakeCalzoneImpl` satisfies the trait bound
`MakeCalzone<Output=Calzone>`, which means we can use it for the associated
type `MakeC` in `impl PizzaOrCalzone for Calzone {...}`. We can do the same with
`MakePizzaImpl`: it satisfies `MakePizza<Output=Pizza>`, so it can be used as
`<Pizza as PizzaOrCalzone>::MakeP`.
What we can't do is mix the types: `MakePizzaImpl` doesn't implement
`MakePizza<Output=Calzone>`, so we can't use it for `Calzone`'s `MakeP`. In
fact, the trait bound `MakePizza<Output=Calzone>` is completely nonsensical;
what does it even mean to have a pizza maker type that produces a calzone?

## Using uninstantiable types

Taking a step back, implementations of `PizzaOrCalzone` will only ever produce
an instance of one of their `MakeP` or `MakeC` associated types. The other one
needs a definition, but the `Calzone` implementation of `fn pizza_or_calzone()`
will always return a `EitherPizzaOrCalzone::Calzone` with an instance of `MakeC`.
That means that while we need to be able to _name_ a type `MakeP` that
implements `MakePizza<Output=Calzone>`, we don't ever need to instantiate it.

That brings us to uninstantiable types. In type theory, these are called [empty
types], because the set of possible values is empty. In Rust, the simplest way
to define an uninstantiable type is to create an `enum` type with no variants.
A `struct` type that has one or more members of an uninstantiable type is also
uninstantiable.

The standard library defines the empty-`enum` type [`std::convert::Infallible`]
for use in `Result` types that can't hold an error, like the output of
`try_from` on the blanket impl of `B: TryFrom<A>` where `A: Into<B>`. There's
also the unstable [`!`] or "never" type that can be used as the output type of
methods that never return, and which will [eventually][infallible-!] be unified
with `Infallible`.

The important feature of uninstantiable types is that they can be defined but
never constructed. While that may seem somewhat useless, it's exactly what we
need to take care of our nonsensical `MakePizza<Output=Calzone>` bound:

```rust
/// Uninstantiable type.
enum CalzoneFromPizza {}

impl Make for CalzoneFromPizza {
    type Output = Calzone;
}

impl MakePizza for CalzoneFromPizza {
    fn stretch_dough(&self, dough: Dough) -> FlatDough {
        match *self {}
    }
    fn from_sliced(self, slices: SlicedPizza) -> Calzone {
        match self {}
    }
}
```

One of the really neat features of uninstantiable types in Rust is the ability
to use them to produce a value of any type. Intuitively, this is similar to the
[principle of explosion]: if we have a value that can't exist, we can do anything!
In Rust, that looks like the empty `match` blocks above. They meet Rust's
requirement for exhaustive matching since there's an arm for every possible
value pattern, and every arm produces a value of the method's return type.

On the surface this is similar to the standard library's [`unreachable!`] macro:
both assert that the code in question can never be reached. The advantage of the
uninstantiable type approach is that it relies on the type system to assert
correctness, where `unreachable!` produces code that asserts unreachability only
at run time (by panicking if it is ever reached).

Putting all of this together (with a corresponding `PizzaFromCalzone` type and
trait implementations), we can write out our `PizzaOrCalzone` implementations:

```rust
impl PizzaOrCalzone for Calzone {
    type MakeC = MakeCalzoneImpl;
    type MakeP = CalzoneFromPizza;

    fn pizza_or_calzone() -> EitherPizzaOrCalzone<CalzoneFromPizza, MakeCalzoneImpl> {
        EitherPizzaOrCalzone::Calzone(MakeCalzoneImpl)
    }
}

impl PizzaOrCalzone for Pizza {
    type MakeC = PizzaFromCalzone;
    type MakeP = MakePizzaImpl;

    fn pizza_or_calzone() -> EitherPizzaOrCalzone<MakePizzaImpl, PizzaFromCalzone> {
        EitherPizzaOrCalzone::Pizza(MakePizzaImpl)
    }
}
```

## Performance

One of the potential downsides of this approach is that it introduces additional
branches in the control flow path. Now, instead of a single linear path for each
`Food::prepare` implementation, there are two `match` expressions. Depending on
the result of `pizza_or_calzone`, execution could proceed down either of the two
possible pairs of branches.

There are a couple things that help us out here. The first is that `rustc` (the
Rust compiler) performs [monomorphization][rustc monomorphization] during compilation.
For each generic function, it produces a distinct copy of the code for each concrete
type that the function is invoked with. Assuming we have some non-generic code
that calls `Pizza::prepare()`, the compiler will produce a version without generics
from our blanket impl over `PizzaOrCalzone`. If we were to hand-write it, it might
look something like this:

```rust
fn prepare_pizza() -> Pizza {
    // This is the same as before.
    let mut dough = Flour::new().add(Water::new()).knead();
    let sauce = Sauce::new();
    let toppings = vec![Pepperoni::new(), Mushroom::new()];

    // This is different; now we know what the associated types are.
    let pizza_or_calzone: EitherPizzaOrCalzone<MakePizzaImpl, PizzaFromCalzone> =
        Pizza::pizza_or_calzone();

    let flat_dough = match &pizza_or_calzone {
        EitherPizzaOrCalzone::Pizza(pizza_base) => pizza_base.stretch_dough(dough),
        // In this branch, `calzone_base` has type `&PizzaFromCalzone`.
        EitherPizzaOrCalzone::Calzone(calzone_base) => calzone_base.cut_and_roll_dough(dough)
    };
    let unbaked = flat_dough.apply(sauce).apply(toppings);

    match pizza_or_calzone {
        EitherPizzaOrCalzone::Pizza(pizza_base) => {
            let baked = unbaked.bake();
            pizza_base.from_sliced(baked.cut_into_slices())
        }
        // Here, `calzone_base` has type `PizzaFromCalzone`.
        EitherPizzaOrCalzone::Calzone(calzone_base) => {
            let unbaked = unbaked.fold_over();
            let baked = unbaked.bake();
            calzone_base.from_baked(unbaked)
        }
    }
}
```

In both of the `match` expressions, the second case statement is unreachable. We
can deduce that from the type of `pizza_or_calzone` alone, without looking at
the actual implementation of `<Pizza as PizzaOrCalzone>::pizza_or_calzone()`
that produced it. The `Calzone` variant of the return type
(`EitherPizzaOrCalzone<MakePizzaImpl, PizzaFromCalzone>`) would contain an
instance of `PizzaFromCalzone`, which we know is an uninstantiable type.
Therefore the returned value must be a `EitherPizzaOrCalzone::Pizza` variant
holding a `MakePizzaImpl` instance. And while it's neat that _we_ can deduce
that, what's even better is that the compiler can do the same, and it can use
that fact to eliminate the dead branches during optimization. Once that's done,
we're left with effectively the same code we had before, back when we
implemented `Food` for `Pizza` directly. Even though our generic implementation
is more complex, `rustc`'s optimizations let us avoid any run-time performance
cost for our deduplication.

## Using this for real: dual-stack sockets for Netstack3

While at Google, I had the pleasure of working on Fuchsia's next-generation
networking stack, [Netstack3]. I did a lot of work on the core socket layer,
mostly around implementing POSIX-specified operations for various types of
network sockets.

Netstack3's core library [embraces static typing][netstack3 static typing] to an
extreme degree; there's almost no virtual dispatch anywhere in the codebase.
This allows for writing code and functions that are generic over the IP version
(IPv4, or IPv6) and restricting the type of arguments to only be IP addresses
(or ICMP error codes, or packet builder types) for the targeted IP version. So
it's impossible, for example, to call the POSIX `bind` function on a TCPv4
socket with an IPv6 address - the types wouldn't work out. This IP-generic
approach well because in general, each IP version is independent: devices are
shared, but all addresses, routing state, neighbor state, etc. are stored and
manipulated independently.

Once place where that's not true is for POSIX IPv6 sockets. [IETF informational
RFC 3493][RFC 3493] has the details, but the general gist is that, to make
it easy to add IPv6 support to previously IPv4-only applications, IPv6 TCP and
UDP sockets would support sending and receiving traffic to and from IPv4 hosts.
That is, they would support dual-stack operation (unless explicitly disabled).
By using the [IPv4-mapped IPv6] subnet to represent IPv4 addresses as IPv6
addresses for the purposes of calls like `bind`, `connect` and `recvfrom`, UDPv6
and TCPv6 sockets can be used to transparently communicate with IPv4 hosts.

To support the POSIX semantics in Netstack3 without duplicating our IP-generic
socket implementation code, we represented the decision of whether a given pair
_(IP version, transport protocol)_ was capable of sending and receiving IP
packets for the _other_ IP stack by defining a trait method that returned an
instance of the type `enum MaybeDualStack`. [Netstack3's docs][NS3 dual stack]
describe the implementation in more detail.

The body of the [`listen_inner`][NS3 datagram listen_inner] function (which
implements POSIX `bind` for UDP sockets) matches on the result of
`sync_ctx.dual_stack_context()` to help decide whether to insert the socket into
the map for receiving packets only for the "current" IP version, for the "other"
IP version, or for both versions. Even though only UDPv6 sockets can be
dual-stack capable, keeping the code generic reduces duplication for the "bind
in current stack" path.

## Conclusions

Modeling behavioral differences via control flow can be a useful technique for
reducing code duplication. It does require adding additional helper traits and
types, but the tradeoff can be worthwhile if there are many usages of the
control flow methods, as there are for Netstack3's multiple socket operations.

In addition, trait implementations on uninstantiable types let us safely name
types that satisfy required bounds while leveraging the type system to avoid
run-time panic opportunities. The Rust compiler knows about these types and can
leverage the fact that they are unconstructable when performing optimization.

[Rust book enums]: https://doc.rust-lang.org/book/ch06-00-enums.html
[algebraic data types]: https://en.wikipedia.org/wiki/Algebraic_data_type
[`std::variant`]: https://en.cppreference.com/w/cpp/utility/variant
[`std::ops::ControlFlow`]: https://doc.rust-lang.org/std/ops/enum.ControlFlow.html
[`std::task::Poll`]: https://doc.rust-lang.org/stable/std/task/enum.Poll.html
[`Result`]: https://doc.rust-lang.org/stable/std/result/enum.Result.html
[`match`]: https://doc.rust-lang.org/book/ch06-02-match.html
[`const`]: https://doc.rust-lang.org/reference/const_eval.html
[`std::convert::Infallible`]: https://doc.rust-lang.org/std/convert/enum.Infallible.html
[`!`]: https://doc.rust-lang.org/std/primitive.never.html
[principle of explosion]: https://en.wikipedia.org/wiki/Principle_of_explosion
[`unreachable!`]: https://doc.rust-lang.org/std/macro.unreachable.html
[rustc monomorphization]: https://rustc-dev-guide.rust-lang.org/backend/monomorph.html
[Netstack3]: https://fuchsia.googlesource.com/fuchsia/+/26a37c4d039d63b0abfc47a55b45ed7f339390d1/src/connectivity/network/netstack3/
[netstack3 static typing]: https://fuchsia.googlesource.com/fuchsia/+/26a37c4d039d63b0abfc47a55b45ed7f339390d1/src/connectivity/network/netstack3/docs/STATIC_TYPING.md
[RFC 3493]: https://datatracker.ietf.org/doc/html/rfc3493
[IPv4-mapped IPv6]: https://en.wikipedia.org/wiki/IPv6#IPv4-mapped_IPv6_addresses
[NS3 dual stack]: https://fuchsia.googlesource.com/fuchsia/+/26a37c4d039d63b0abfc47a55b45ed7f339390d1/src/connectivity/network/netstack3/docs/DUAL_STACK_SOCKETS.md#How-dual_stack-behavior-is-implemented
[NS3 datagram listen_inner]: https://fuchsia.googlesource.com/fuchsia/+/26a37c4d039d63b0abfc47a55b45ed7f339390d1/src/connectivity/network/netstack3/core/src/socket/datagram.rs#1974
[NS3 UDP dual_stack_context]: https://fuchsia.googlesource.com/fuchsia/+/26a37c4d039d63b0abfc47a55b45ed7f339390d1/src/connectivity/network/netstack3/core/src/transport/integration.rs#252
[tagged unions]: https://en.wikipedia.org/wiki/Tagged_union
[`std::visit`]: https://en.cppreference.com/w/cpp/utility/variant/visit
[empty types]: https://en.wikipedia.org/wiki/Empty_type
[infallible-!]: https://github.com/rust-lang/rust/issues/35121
`macro_rules!`
===============
*"Avoiding repeating ourselves by generating code with macros"*

### Topics
 - `macro_rules!`

What are we trying to do?
-------------------------
Oh no! We have a system which requires 4-tuple of all components *(named `print_state`)*. We would have to write another `IteratorTuple` implementation to handle 4-tuples but that would be mostly duplicate code. Worse, if we add more systems in the future, we could need 3-tuples, 5-tuples, 12-tuples, and so forth. It gets quite unmanageable quite fast.

Solution: There is a clear pattern on how the implementations expand as tuples grow in size. Use "variadic declarative macros" to generate appropriate implementations of different sizes. This means that we have all the possible tuple sizes handled, but we only need to write a single "blueprint" to handle all of them.

The "thing" here is that we take the simplest solution available: writing lots of more-or-less trivial complexity boilerplate, but we are skipping the writing part. Afterwards, we still have most of the elegance of the simple solution intact, without putting in much additional effort!

Declarative macros
------------------
We are going to use something called *"declarative macros"* as starting point of our implementation. To start defining a macro, we call `macro_rules! macro_name {...}`. One might have noticed that some calls we are making have the exclamation mark (`!`) appended: This marks a call to a macro. *(Which means that we are calling a macro to create a macro!)*

Now, we start with something simple: let's write a macro without any parameters which just has our `IteratorTuple` implementation in it.
```rust
macro_rules! implement_iterator_tuple {
    () => {
        impl<A, B> IteratorTuple for (A, B)
            where A: Iterator,
                  B: Iterator
        {
            type ItemTuple = (A::Item, B::Item);

            fn next_all(&mut self) -> Option<Self::ItemTuple> {
                match (self.0.next(), self.1.next()) {
                    (Some(pos), Some(vel)) => Some((pos, vel)),
                    _ => None,
                }
            }
        }
    };
}
```
In practice, we just wrapped our `IteratorTuple` implementation to a fancy macro declaration. Now, below our macro declaration, we can add the line
```rust
implement_iterator_tuple!();
```
to generate the implementation.

The first line with `macro_rules!` just names our macro. Inside that block, we can then define a number of macro subtypes using:
```rust
# macro_rules! example_macro {
() => {}; // Base type, called using "example_macro!()"
(name) => {}; // Named subtype, called using "example_macro! { name }"
# }
```

Each macro sub-type starts with identifier and parameter declarations within parentheses, and is followed by a block defining what the macro expands to. Currently, in our case, the macro takes no parameters and just expands to a fixed block of code. Next, let's fix that and start replacing things with parameterized values.

Parameter types for macros differ greatly from regular function parameters. We do not use concrete types for them, but rather *designators*, which are lower-level concept the compiler uses to differentiate whether or not some piece of code is valid in a context. The basic syntax is as follows:
```rust
# macro_rules! example_macro {
    ( $arg0:designator, $arg1:designator, $arg2:designator ) => { }
# }
```
macro argument name starts with `$name` and then the `:designator` part is substituted with a relevant designator. There are a lot of different variations and choice depends completely on what the argument is going to be used for. Here's a few examples of available desingators, just so that you get the idea:
 - `block` - accepts any code block
 - `ident` - accepts any valid *identifier* *(names like `A`, `cat`, `QuickBrownFox`, etc.)*
 - `literal` - literal constant
 - `expr` - any expression, like `a + b` or `42 + 24`
 - `tt` - token tree. I won't go into detail with this one, but translates *to something quite low-level*. The important thing to note is that `tt` designator accepts almost anything and can be used in most situations.

Now, let's replace our generic type parameter identifiers `A` and `B` with parameters!
```rust
macro_rules! implement_iterator_tuple {
    ($A:ident, $B:ident) => {
        impl<$A, $B> IteratorTuple for ($A, $B)
            where $A: Iterator,
                  $B: Iterator
        {
            type ItemTuple = ($A::Item, $B::Item);

#            fn next_all(&mut self) -> Option<Self::ItemTuple> {
#                match (self.0.next(), self.1.next()) {
#                    (Some(a), Some(b)) => Some((a, b)),
#                    _ => None,
#                }
#            }
        }
    };
}
```
We added parameter declarations `($A:ident, $B:ident) => { ...` and then just prefixed all occurences of `A` and `B` with `$`. Now, we can call
```rust
implement_iterator_tuple2!(B, C);
```
which expands to our implementation, but the `A` and `B` are now substituted with `B` and `C`, respectively! Not very useful yet, but getting there. Now, let's add more parameters. We just prefix things with dollar signs for now and get to actual useful stuff later:
```rust
macro_rules! implement_iterator_tuple {
    ($A:ident, $B:ident, $i0:tt, $i1:tt, $a:ident, $b:ident) => {
        impl<$A, $B> IteratorTuple for ($A, $B)
            where $A: Iterator,
                  $B: Iterator
        {
            type ItemTuple = ($A::Item, $B::Item);

            fn next_all(&mut self) -> Option<Self::ItemTuple> {
                match (self.$i0.next(), self.$i1.next()) {
                    (Some($a), Some($b)) => Some(($a, $b)),
                    _ => None,
                }
            }
        }
    };
}

// And call it
implement_iterator_tuple2!(B, C, 0, 1, a, b);
```
The tuple indices and pattern matching parameter names are now also substituted from macro parameters. Now, we re-organize our parameters a bit and group logically connected parameters to tuples. The parameter declaration becomes:
```rust
(( $i0:tt, $a:ident, $A:ident ), ( $i1:tt, $b:ident, $B:ident )) => {
    /* snip */
}
```

Now, notice how our parameters clearly repeat a pattern `(index, item_name, type_name)`? In order to expand from 2-tuple, to 3-tuple, we would need to add just another parameter tuple and add its substitutions to the macro. How could we automate that?

This is where we move into *variadic macros*. We actually can write a macro which takes in variable number of our parameter 3-tuples and expands them into a n-tuple implementation of `IteratorTuple`.

Basic variadic syntax for parameter definition is as follows:
```rust
( $( $x:designator ),* ) => { .. }  // Macro with argument x repeated 0..n times
( $( $x:designator ),+ ) => { .. }  // Macro with argument x repeated 1..n times
```

We can also use tuples in out repeat pattern by wrapping the arguments in parentheses. For example:
```rust
( $( ($x:ident, $y:expr) ),* ) => { .. } // Arguments x and y repeated 0..n times
```

Now, let's try this with our arguments:
```rust
($( ($i:tt, $item_name:ident, $type_name:ident) ),+) => { .. }
```
that breaks down to:
 - `($( ... ),+) => { .. }` - Arguments repeat `1..n` times
 - `($i:tt, $item_name:ident, $type_name:ident)` - Each argument is a 3-tuple, where
    - `$i:tt` - `i` is a token tree. Tuple indexing is a bit of a wild-card at language-level so  it's hard for compiler to validate what is a valid index for tuples. Just use `tt` to tell the compiler to "not to worry about it".
    - `$item_name` - a bit of an extra. We don't want to use the capital letters for variable names when pattern matching *(which we totally could)*, so pass an extra argument for that
    - `$type_name` - Names for the generic type parameters. This and the `$item_name` are both `ident` so any valid identifier could be used.

So, in other words:
> *"Macro accepts 3-tuples of `i`, an `item name` and a `type name`. There should be one or more of these tuples."*

How do we put these to use then? Expanding macro parameters uses the exact same syntax as defining them. Let's approach this with examples. Assuming call to some macro with argument `$some_arg` with parameter values `a, b, c, ...`
```rust
$( ... ),*              // Expand whatever it is inside the braces 0..n times
$( ... ),+              // Expand whatever it is inside the braces 1..n times

$( $some_arg ),*        // Expands to `a, b, c, ...` as many times as there
                        // are $some_arg values available.

$( $some_arg-1 ),*      //  Expands to `1+a-1, 1+b-1, 1+c-1, ...`
$( 1+$some_arg-1 ),*    //  Expands to `1+a-1, 1+b-1, 1+c-1, ...`
$( $a, $b ),*           //  Valid only if there are exactly the same number of
                        //  both of the arguments `a` and `b` available
```

Now, let's generate a 3-tuple with our macro by calling
```rust
define_iterator_tuple!((0, a, A), (1, b, B), (2, c, C))
```

First, the `impl` line. Let's see what happens:
```rust
impl<$( $type_name ),+> IteratorTuple for ($( $type_name ),+)
    where $($type_name: Iterator),+
```
Which expands to *(each individual expansion on its own line)*
```rust
impl<A, B, C> IteratorTuple
for (A, B, C)
where A: Iterator, B: Iterator, C: Iterator,
```

Then, we construct the `ItemTuple` associated type:
```rust
type ItemTuple = ($($type_name::Item),+);
```
which expands to
```rust
type ItemTuple = (A::Item, B::Item, C::Item);
```

Now, the pattern matching has quite a few expansions, but the basic idea:
```rust
match ($(self.$i.next()),+) {
    ($( Some($item_name) ),+) => Some(($( $item_name ),+)),
    _ => None,
}
```
and after expansion:
```rust
match (self.0.next(), self.1.next(), self.2.next()) {
    (Some(a), Some(b), Some(c)) => Some((a, b, c)),
    _ => None,
}
```

Thus, the complete macro is just:
```rust
macro_rules! implement_iterator_tuple {
    ($( ($i:tt, $item_name:ident, $type_name:ident) ),+) => {
        impl<$( $type_name ),+> IteratorTuple for ($( $type_name ),+)
            where $($type_name: Iterator),+
        {
            type ItemTuple = ($($type_name::Item),+);

            fn next_all(&mut self) -> Option<Self::ItemTuple> {
                match ($(self.$i.next()),+) {
                    ($( Some($item_name) ),+) => Some(($( $item_name ),+)),
                    _ => None,
                }
            }
        }
    };
}
```

Whew, that was a lot of stuff! Now, the only thing left to do is to call the macro for all tuple sizes we want to generate the implementations for. I chose to start with `n = 2..12`.

Brace yourself, for *Der Mostrositat*:
```rust
implement_iterator_tuple!((0, a, A), (1, b, B));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D), (4, e, E));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D), (4, e, E), (5, f, F));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D), (4, e, E), (5, f, F), (6, g, G));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D), (4, e, E), (5, f, F), (6, g, G), (7, h, H));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D), (4, e, E), (5, f, F), (6, g, G), (7, h, H), (8, i, I));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D), (4, e, E), (5, f, F), (6, g, G), (7, h, H), (8, i, I), (9, j, J));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D), (4, e, E), (5, f, F), (6, g, G), (7, h, H), (8, i, I), (9, j, J), (10, k, K));
implement_iterator_tuple!((0, a, A), (1, b, B), (2, c, C), (3, d, D), (4, e, E), (5, f, F), (6, g, G), (7, h, H), (8, i, I), (9, j, J), (10, k, K), (11, l, L));
```
This implements the `IteratorTuple` for all n-tuples of iterators, for tuple sizes ranging from 2 to 12.


What next?
----------
Now that iterating over component storages can be done easily for any number of components, it is time to start designing our dispatcher. Currently, our systems have nothing in common, they are just some arbitrary functions imported from some arbitrary modules. Along the same lines goes our components and their storage vectors. Thus, unifying the way we handle components and systems will be our first priority.

The full source code can be found in branch `part-5` ([link](https://github.com/Kailari/kokonaisuus/tree/part-5))

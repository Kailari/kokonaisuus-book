Iteratorception
===============
*"Iterating tuples of iterators using an iterator"*

### Topics
 - `trait Iterator`
 - Custom traits
 - Implementing traits on external types

What are we trying to do?
-------------------------
`while let` is needlessly verbose, and with it we are stuck doing the pattern matching in the systems themselves. We would like to *abstract* that logic out of there to allow better handling for situations where entities don't have all components in the future. We are going to use standard language feature, `Iterators` which, well, was desinged for this very purpose, iterating over collections of things.

There are some language limitations we need to take in account with our implementation, but mostly what we are doing here is quite simple once you wrap yuor head around it the correct way. For now, we are going with naïve approach where each system takes care of being able to iterate over their own component types.

One iterator to rule them all
-----------------------------
So, basically we are again in situation where we have two iterators:
```rust
let mut pos_iter = positions.iter_mut();
let mut vel_iter = velocities.iter();
```
and we would like to make mutation
```rust
pos.value += vel.value;
```
for each pair
```rust
(pos, vel) = (pos_iter.next(), vel_iter.next());
```
until either `pos_iter.next()` or `vel_iter.next()` returns `None`.

Written as a for loop, this would then look like:
```rust
for (pos, vel) in (pos_iter, vel_iter) {
    pos.value += vel.value;
}
```

Well, that would actually be perfectly valid code if the tuple `(pos_iter, vel_iter)` implemented the trait `Iterator`! How is the `Iterator` implemented then? Let's have a look at relevant parts of its definition:
```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

So, the iterator has an associated type which defines the type of the items the call to `.next()` produces. Then, the `.next()` itself returns an `Option`, which is either `None` *(no more items)* or `Some(value)` *(next item from the collection)*.

In our case, the item would be a tuple of our child iterators' `Item` types and `next()` would simply do the pattern matching we currently do in the `while let` loops!

For defining the `Item` from two iterators, assuming our iterators are `pos_iter` and `vel_iter` in the `apply_velocity`, we could just write the items "just by knowing" the type beforehand *(We know what `Iterator::Item` for `IterMut` and `Iter` look like)*:
```rust
type Item = (&mut PositionComponent, &VelocityComponent);
```
If we do not want to pull the child-iterators' item types out of thin air, and want more concrete way of referring to them, we can use *fully-qualified trait syntax* to refer to the `Iterator` implementations of our actual iterator types *(`IterMut` and `Iter` are structs)*. This looks a bit messy at first, but don't let that bother you.

The basic syntax is
```rust
<Struct as Trait>::AssociatedType
```
So, in our case, where we have `(IterMut, Iter)` from which we want the `Iterator::Item` types:
```rust
type Item = (<IterMut<'a, PositionComponent> as Iterator>::Item,
             <Iter<'a, VelocityComponent> as Iterator>::Item);
```
What we are doing is just: We refer to `Iter` and `IterMut` as `Iterators` and use the `Item` associated type from the implementation of that trait to define a tuple of those iterators' items. This compiles to exactly the same code as the another snippet above. However, this allows us to refer to the `Item` types, without "knowing" what they look like beforehand. In our case, this doesn't bring us any benefits over directly using concrete types, but I prefer the more explicit form anyway.

Now that we know how to implement the `Iterator`-trait, let's try implementing it on our tuple of component vector iterators:
```rust
impl<'a> Iterator for (IterMut<'a, PosComp>, Iter<'a, VelComp>) {
    type Item = (<IterMut<'a, PosComp> as Iterator>::Item,
                 <Iter<'a, VelComp> as Iterator>::Item);

    fn next(&mut self) -> Option<Self::ItemTuple> {
        match (self.0.next(), self.1.next()) {
            (Some(pos), Some(vel)) => Some((pos, vel)),
            _ => None,
        }
    }
}
```
Looking great! Except, it does not compile. **WHAT?!**

Yeah. What happens here, we take an **externally defined trait** `Iterator` and define it on tuple of iterators, which like all tuples is an **externally defined type**. This breaks trait implementation *orphaning rules*.

What is going on exactly?

Well, how does the compiler know what traits it needs to use? For standard library, implementations provided are always in scope. For any other crate, we need to explicitly `use` that trait to bring its implementations to scope. For instance, let's assume there is some `trait Application`, provided by `app_utils` crate. The `Application` trait provides a `fn run(self)` which starts the application. In order for implementations of that trait to be visible, we must add `use app_utils::Application;` to our code to be able to call `app.run();`

Now, we just implemented the trait `Iterator` on an arbitrary tuple, *what do we import to bring that to scope?* The `Iterator` trait? But what if another crate implements the `Iterator` trait for that same tuple? The conclusion is, we have no way of safely referring to that implementation; it is now an orphan, it has no parent we can use to bring it to scope.

But what can we do? Well, as per documentation, the preferred way seems to be to use a wrapper iterator type. However, we do not want to write individual wrapper iterators for all tuple sizes *(in the likely case we have tuples larger than 2-tuples in the future)*, so we define our own iterator trait in addition to creating our own iterator struct.

```rust
pub trait IteratorTuple {
    type ItemTuple;

    fn next_all(&mut self) -> Option<Self::ItemTuple>;
}

pub struct IterTuple<T>
    where T: IteratorTuple
{
    iterators: T,
}
```
The trait `IteratorTuple` is straightforward, it's basically the standard `Iterator` with everything renamed. The iterator struct however, has a few things worth explaining.

We define a type parameter `T` to be used as the type of the wrapped tuple of iterators. To keep things clean we declare the trait bounds on another line with the `where` keyword, followed by comma-separated list of desired bounds. In this case we have only one bound: *We require the type parameter `T` to implement `IteratorTuple`*.

Now, we know that the object wrapped into a `IterTuple`, whatever it is, always must implement `IteratorTuple`, thus now we can write `Iterator` implementation for `IterTuple` as easily as:
```rust
impl<T> Iterator for IterTuple<T>
    where T: IteratorTuple
{
    type Item = T::ItemTuple;

    fn next(&mut self) -> Option<Self::Item> {
        self.iterators.next_all()
    }
}
```
Here, we already know that `T` is an `IteratorTuple` so we do not need to use fully qualified syntax to refer to the `IteratorTuple::ItemTuple`, just `T::ItemTuple` suffices.

At this point, one might wonder, if `IterTuple` is just dumb wrapper around the `IteratorTuple`, couldn't we just write blanket `Iterator` implementation to cover everything that implements `IteratorTuple`?
```rust
impl<T: IteratorTuple> Iterator for T {
    type Item = T::ItemTuple;

    fn next(&mut self) -> Option<Self::Item> {
        // ...
    }
}
``` 

Sadly, no. The standard library already provides some blanket implementations over references of iterators, and our bounds overlap with those. I'm not 100% sure there is no way around this, but for now, it is just easier to work around it with our `IterTuple`-wrapper. For same reasons, we cannot implement `IntoIterator` either.

For a bit nicer time constructing our iterator wrappers, `From<T>` is just the trait we need:
```rust
impl<T> From<T> for IterTuple<T>
    where T: IteratorTuple
{
    fn from(iter_tuple: T) -> Self {
        IterTuple { iterators: iter_tuple }
    }
}
```

Now, in the `apply_velocity`, we can write the loop as:
```rust
for (pos, vel) in IterTuple::from((pos_iter, vel_iter)) {
    pos.value += vel.value;
}
```
...except it does not recognize `(pos_iter, vel_iter)` as a `IteratorTuple`, because we didn't implement that yet! Let's add that to the top of the `apply_velocity.rs` now:
```rust
impl<'a> IteratorTuple for (IterMut<'a, PositionComponent>, Iter<'a, VelocityComponent>) {
    type ItemTuple = (<IterMut<'a, PositionComponent> as Iterator>::Item,
                      <Iter<'a, VelocityComponent> as Iterator>::Item);

    fn next_all(&mut self) -> Option<Self::ItemTuple> {
        match (self.0.next(), self.1.next()) {
            (Some(pos), Some(vel)) => Some((pos, vel)),
            _ => None,
        }
    }
}
```
Looking familiar? This is the exact same thing we tried to implement using `Iterator`, but this time we use our own `IteratorTuple` trait so this is valid. `apply_velocity` should now compile again!

Also, note that the local variables for the iterators need not be mutable at all, as we do not mutate them before passing them to `IterTuple::from`. That is **we** do not mutate them, but as we pass the ownership down to `IterTuple`, we don't actually care what they do with it as *`IterTuple` now owns the iterators, thus it can anyway do whatever it wants with them*. In other words, when passing by value *(when moving the ownership)* mutability can "change".

We could add similar looking implementations to all other systems, but then again, for all 2-tuples, *the implementations are the same, but with different component types*. Do you know what that calls for?


### Generic implementation for the `IteratorTuple`
*(The naïve implementation without added generics where each system has their own `IteratorTuple` implementation is in its own branch `part-3`, version with generics is in `part-4`)*

TODO

What next?
----------
TODO

The full source code can be found in branches `part-3` ([without generics](https://github.com/Kailari/kokonaisuus/tree/part-3)) and `part-4` ([with generics](https://github.com/Kailari/kokonaisuus/tree/part-4))

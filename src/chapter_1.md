(Almost) walking skeleton
=========================
*"Adding more behavior"*

### Topics
 - Traits
 - `trait From<T>`
 - Functions
 - Basic borrowing
 - (Bonus) Trait blanket implementations

What are we trying to do?
-------------------------
We are building directly on top of what we got together in the previous part. We are adding more behavior by adding more "systems". This should make it a little more evident how uncontrollably cluttered things start to become, from just very simple behavior.

To keep things as tidy as one can manage with just a single module *(modules are covered briefly in the next part)* we break each of the systems into their own functions. These functions will be the de-facto systems for our ECS until we have time to figure out how to do them properly.

Enough blabbering, we would like to:

 1. Be able to define acceleration for the entities
 2. Be able to define friction for the entities
 3. Have the acceleration and friction applied before velocity is applied

For achieving this, we are going to:

 1. Add two new systems. One for acceleration and second for friction
 2. Create and initialize components for the systems

Moar components!
----------------

Like previously, declare new components:
```rust
struct FrictionComponent {
    amount: f64,
}

struct AccelerationComponent {
    x: f64,
    y: f64,
}
```

However, for a bit of added flavor, we are going to implement `From<T>` *trait* for these new components and the previous components we had. This might be a bit of a misuse of this trait, but for sake of an example:
```rust
impl From<f64> for FrictionComponent {
    fn from(source: f64) -> Self {
        FrictionComponent { amount: source }
    }
}
```

What's going on? `From` is a *trait* provided by the standard library, which is as per documentation:
> *Used to do value-to-value conversions while consuming the input value*

The definition for the `From`-trait looks something like:
```rust
trait From<T> {
    fn from(source: T) -> Self;
}
```
Ok, from this it is a bit easier to see what's going on. The trait provides a single *associated function*, which takes in something of type `T` and returns `Self`. In this case, we implemented the trait on `FrictionComponent`, thus in that implementation, the `Self`-type is `FrictionComponent`. Also, we provided `f64` for `T`, so we can substitute that for the source parameter.
 - "`impl From<f64> ...`": *"Implement the trait `From`. Use `f64` for the type parameter `T`"*, thus
 - `T` from the trait is substituted with `f64`
 - "`...for FrictionComponent`": *"...the implementation is for type `FrictionComponent`"*, thus
 - `Self` from the trait means `FrictionComponent`

Now, looking back at `From<f64>` implementation for the `FrictionComponent`, there isn't *really* that much anything special going on (Especially if you are familiar with generics from other languages). We just wrap the component construction into a fancy trait implementation.

Likewise, we can implement `From` for `PositionComponent` from a 2-tuple of `f64`s
```rust
impl From<(f64, f64)> for PositionComponent {
    fn from(source: (f64, f64)) -> Self {
        PositionComponent { x: source.0, y: source.1 }
    }
}
```

This then allows us to write:

```rust
// Previously
PositionComponent { x: 0.0, y: 0.0 };
FrictionComponent { amount: 1.0 };

// After
PositionComponent::from((0.0, 0.0));
FrictionComponent::from(1.0);
```

...yeah. Not much of an improvement; but it gave us an excuse to talk about traits so I count that as a win!

So, we saw that traits in essence are just something we can implement on structs to provide them with some well-defined behavior. In this case we implemented the trait which allowed conversion into our types from other types. Later on, we are going to use traits to define `System` with `tick`-method which then allows us to handle all systems equally from a `SystemDispatcher`. *This description is a daring oversimplification of what traits are, but hopefully it helps you get the basic idea.*


Let's not get ahead of ourselves! What we have here, is overly complex way of initializing components, but we are keeping it because I'm an idiot! Now, what shall we do with our new components?


### Applying velocity in a function

Next up, moving logic to functions and figuring out what we need to change for making that work.

Let's throw ourselves straight into the deep end and look at what the function `apply_velocity` looks like:
```rust
fn apply_velocity(positions: &mut Vec<PositionComponent>, velocities: &Vec<VelocityComponent>) {
    let mut pos_iter = positions.iter_mut();
    let mut vel_iter = velocities.iter();

    while let (Some(pos), Some(vel)) = (pos_iter.next(), vel_iter.next()) {
        pos.x += vel.x;
        pos.y += vel.y;
    }
}
```

There are the iterators and the `while let`-loop from the last time. But what's up with the parameters? Why are the `&`-symbols in there? This is due to ownership rules *(Rust book, chapter 4.)*. If we would just pass the component vectors as-is, their ownership would move into the function, and the vectors would ultimately be dropped when the function ends.

Obviously, this is not what we want, rather when we call the function, we would like to say

> *"Hey, here is this thing I own. You are allowed to do something with it, but I want it back when you are done with it, OK?"*

This can be achieved using *references*. The parameter declaration `positions: &mut Vec<VelocityComponent>` tells us that `position` is a reference to a component storage vector containing velocities, and that we are allowed to mutate those. The `velocities`-parameter on the other hand is an *immutable reference*, which allows us to read the components from the vector, but not mutate them. (`&mut` vs. just `&`, both are references, but adding the `mut` permits mutation)

Passing the vectors as references has the effect of *borrowing* the ownership of those vectors to the function for its *lifetime*. When the function ends, the lifetime of the function, with its parameters, ends. At this point the references received as parameters are dropped, ending the borrow, causing the ownership to return to the caller.

```rust
fn apply_velocity(params: &SomeType)
{ // Lifetime of the function starts

    // params is alive and usable, as here parameters share the lifetime of the function

} // Function lifetime ends
```

Above is actually sugared version of the function. It is self-evident from the context that the reference and the function have the same lifetime, but in some cases that might not be the case *(Now this goes a bit out of scope for this part)*. The full, de-sugared form of the above function would be
```rust
fn apply_velocity<'a>(params: &'a SomeType)
{ // Lifetime of the function ('a) starts

    // params is alive and usable as 'a has not ended yet

} // Function lifetime ('a) ends
```

The odd looking `'a` here is a *lifetime annotation*. As I said earlier, when the function and its reference parameters have the same lifetime, that is the self-evident common case and the annotations can be *elided* *(left out)*. After *elision of the lifetime annotations*, we get the "sugered" version.

Again, now that we have taken a peek at what lifetimes and references are, a short TL;DR:
 - Lifetime of something is the time from when it is allocated to when it goes out of scope
 - Function calls have lifetimes from start of the call until the function returns
 - Passing a value to a function *moves* that value with its ownership, preventing us from using it after that point *(unless the function returns it back to us)*
 - When lifetime of *a value* ends, it is dropped *(its memory is freed)*
 - Passing a reference to a function *borrows* the data in question to the function, for the lifetime of the reference.
 - When lifetime of *a reference* ends, only the reference is dropped, the borrow ends and the caller is again allowed to use the value normally.

Hopefully that starts to make sense. Understanding what lifetimes are and how they relate to borrowing and moving ownership around is critical when working with Rust.

So, we learned that *passing parameters as references has the effect of borrowing them instead of moving the ownership, allowing us to continue using them in the caller (pass them to multiple systems).* In this case, this is exactly what we wanted. On the other hand, in the case of the `From`-trait, the `from`-method takes the parameter by value, thus consuming the ownership. As the trait method does not return the value, the value is dropped after the call *(which is the desired behavior in that specific case)*.

### Calling the system functions
Calling the systems is now quite trivial, but let's have a look what borrowing looks for the "giving" side:
```rust
// For example, here we borrow `velocities` as mutable and `accelerations` as immutable. When
// the function ends, references' lifetimes end (because the references go out of scope), thus
// the borrow ends and...
apply_acceleration(&mut velocities, &accelerations);
// ...we are allowed to borrow the `velocities` again as it is no longer borrowed.
apply_friction(&mut velocities, &frictions);
apply_velocity(&mut positions, &velocities);

print_positions(&positions)
```

Here, we must explicitly tell the compiler that we aknowledge that the values are being passed as references. Additionally, we need to define the mutability of those references. Prefixing the variable name with the `&` turns it into a reference. Further adding the `mut` modifier turns the *(immutable) reference* into a *mutable reference*. We can only borrow as mutable if the value we are trying to borrow is mutable.


### Maths! ...err, darn it.
I really hate comparing floating point numbers against zero or some arbitrary `0.0001`. For this purpose, we are going to need *epsilon* which we ca... Oh.

As of writing, use of some associated numeric constants for number types is an *unstable feature*. These are language features that are still under development, and more or less subject to change. In this case, the required feature *(`assoc_int_const`)* has passed the review process and is due to *"be merged soon"*, but for now, it is unstable.

So, in order to use unstable features, we must use the nightly toolchain version and explicitly tell that we wish to enable those features in this project. In this case, we need to install the nightly toolchain
```bash
$ rustup toolchain install nightly
```
and then enable it
```bash
$ rustup default nightly # globally set default toolchain
# or
$ rustup override set nightly # set override for current directory only
```

With that done, at the very top of the `main.rs`, we add
```rust
#![feature(assoc_int_consts)]
```
...and we're good to go! Now we get access to `f64::EPSILON` and a whole lot of other goodies! *(Which we most likely won't even be using, but oh well)*


### Ok, Maths! (for real this time!)
And here is the anticlimatic maths part. I wanted to quickly showcase what some common operations like `sqrt`, `abs`, `max` and `signum` look like in rust. So here it is, in `apply_friction`, we have
```rust
if velocity_length_squared < f64::EPSILON { // We needed the assoc_int_consts for this ":D"
    continue;
}

// Note that `.sqrt()` is an associated function of `f64` instead of some utility class like
// Java's `Math.sqrt()`. A bit unintuitively, the call DOES NOT MODIFY THE ORIGINAL, though.
let velocity_length = velocity_length_squared.sqrt();

// ...same thing for `.abs()`
let abs_friction_x = (vel.x / velocity_length * fri.amount).abs();
let abs_friction_y = (vel.y / velocity_length * fri.amount).abs();

// ...and `.max()`
let magnitude_x = (vel.x.abs() - abs_friction_x).max(0.0);
let magnitude_y = (vel.x.abs() - abs_friction_y).max(0.0);

// ...and `.signum()`
vel.x = vel.x.signum() * magnitude_x;
vel.y = vel.y.signum() * magnitude_y;
```


### Blanket implementations (off-topic)

Earlier, I told that the `From` trait *"allowed conversion into our types from other types"*. The key word here is `Into`, which happens to be another trait provided by the standard library. It also happens to be the so-called *reciprocal operation* of the `From`-trait. Let's look at an example of what this means:
```rust
let a = SomeStruct::from(some_value);   // From<T>
let b: SomeStruct = some_value.into();  // Into<T>
```

These both achieve the very same thing; *they convert the `some_value` into an instance of `SomeStruct`*. The funny thing here, is that the implementation for `Into` of any struct implementing `From` can be written as
```rust
impl Into<TargetType> for SourceType {
    fn into(self) -> TargetType {
        TargetType::from(self)
    }
}
```

It would be quite cumbersome to write *(or copy-paste around)* a lot of such implementations. Luckily, the standard library provides a *blanket implementation* which implements `Into` for anything implementing the `From`-trait. That is, somewhere in the standard library there is written something like:
```rust
// "S may be anything implementing From for T, implement Into<T> for S"
impl<S: From<T>, T> Into<T> for S {
    fn into(self) -> T {
        T::from(self)
    }
}
```

Without going any more off-topic, a couple notes about that:
 - `S: From<T>` imposes a *trait bound*, meaning that anything used as `S` must satisfy that bound.
 - `impl<S: From<T>, T>` looks scary, but really, we are just pulling `S` and `T` out of thin air, telling that they have this sort of relationship and then we are allowed to use those in our type definitions for the `impl`-block. *(there are some rules on "constraining" the type parameters, but we won't cover those here)*


What next?
----------
We got a quite a lot of explanation from very little code changes, but hopefully that's a good sign. Next we should start moving things to new files as our `main.rs` is getting quite cluttered.

The full source code can be found in branch `part-1` ([link](https://github.com/Kailari/kokonaisuus/tree/part-1)).

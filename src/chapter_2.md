Spring cleaning
=========================
*"Breaking things down and moving them into modules"*

### Topics
 - Modules
 - Associated functions and struct methods
 - `trait Add`
 - `trait Mul`
 - `trait AddAssign`
 - Deriving `Copy` and `Clone`
 - `trait Display`

What are we trying to do?
-------------------------
Our `main.rs` has turned into an ugly mess with everything residing in a single file. We should move things around a bit and make things tidier by extracting everything  to their own modules. Additionally, I got a bit carried away with maths stuff and ended up writing a simple `Vector2d` implementation we can use to simplify system logic.

Crates and Modules
------------------
In Rust, a single project, application binary or library is called a *"crate"*. For example, this project, *"kokonaisuus"* is a single binary *(application)* crate. As this is a binary crate, we have a `src/main.rs` which contains the *main entry point* for our application.

So, we have *crates* which then consist of one or more *modules*. Currently, we have only one module `main`. What we would like to do next, is to split our current main-module into multiple *submodules*.

Declaring new modules always happens at the *module root*. In this case, we have only the *main* module. Let's say, we would like to create a module for all of our systems. At the top of the `main.rs` we add

```rust
mod systems;
```
Now, the compiler starts the compilation process from the `main.rs`. Next, it sees the module declaration and searches for *module root* of the `systems` module from two paths:

 1. `src/systems.rs`
 2. `src/systems/mod.rs`

These two are mutually exclusive, we must pick one of them *(adding both files would mean that we have two modules with the same name!)*. Here, `mod.rs` is special file name, a bit like `main.rs` is. It is specifically used to declare the module root in situations where module is declared as a separate directory. There are notable differences between these two:

 1. In first case, whole `systems` module hierarchy needs to be defined in a single file. Typically this means that `systems` does not have any submodules and is rather simple on its own.
 2. Second case, all other files *(including directories)* in `src/systems/` directory can be treated as submodules of `systems` by adding `mod <module name>;` to `src/systems/mod.rs`

We use modules like 1. for simple, self-contained things, like `vector.rs`. Variant 2. is more suitable for `systems` in this case, as we can then declare all our actual systems as submodules of the `systems` module, keeping the systems themselves in their own modules. Thus, our file structure is something like
```
src/
|--/main.rs
|--/vector.rs
|--/systems/
|   |------/mod.rs
|   |------/apply_velocity.rs
|   |------/apply_friction.rs
|   |------/...
|--/components/
    |---------/mod.rs
    |---------/...
```
Now, in `src/main.rs` we define
```rust
pub mod vector;
pub mod systems;
pub mod components;
```
similarly in the `src/systems/mod.rs` we add
```rust
pub mod apply_velocity;
pub mod apply_friction;
// ... the rest of the systems
```
...and so on. After this, each file represents their own modules. The `pub` keyword means that the submodules should be visible to the outside modules. This allows accessing things like `systems::apply_friction::something_defined_in_apply_friction`. Additionally, to make functions, structs and traits visible, we need to add the `pub` keyword to their definitions, too. For example, the `apply_velocity` module then has
```rust
pub fn apply_velocity(/* snip */) { /* snap */ }
^^^ This here
```

From here, we perform a trivial cut-and-paste and move our systems and components to their own files. Also, we must add `pub` to everything we want to be visible to other modules. After this, however, we have a ton compilation errors as the components are defined in separate modules and the systems can no longer find them!

Now, to fix this, how can we use something from, say, `systems/apply_velocity` in our `main.rs`?


### Including modules from our own crate

Importing dependencies happens using the `use`-keyword. In this case, we are referring to modules in our own crate, thus to import the `fn apply_velocity(...)` to the `main.rs`, we can add the line
```rust
use crate::systems::apply_velocity::apply_velocity;
```

Here, we import the function with name `apply_velocity` from the module `systems/apply_velocity`. Additionally, as we refer to our own crate, we add the `crate::` to the beginning. There is still room for improvement, as the name `apply_velocity` repeats, making the line a bit ugly to look at. Also, after we add more imports, we start to see a pattern here:
```rust
use crate::systems::apply_velocity::apply_velocity;
use crate::systems::apply_friction::apply_friction;
use crate::systems::apply_acceleration::apply_acceleration;
use crate::systems::print_positions::print_positions;
```

All these have the `crate::systems` at the beginning. Thus, we can write this alternatively as:
```rust
use crate::systems::{
    apply_velocity::apply_velocity,
    apply_friction::apply_friction,
    apply_acceleration::apply_acceleration,
    print_positions::print_positions,
}
```

Now, what about the repeats? Here, one thing to note here is that we actually only ever export one thing from each of our system modules. Wouldn't it be neat if we could import them directly from the `systems` module, without needing to refer to them using the `systems::<system_name>`?

Re-exporting to the rescue! We can *re-export* the system functions from the `systems` module to remove the repetition. First, remove the `pub` keywords from system modules:
```rust
mod apply_velocity;
mod apply_friction;
mod apply_acceleration;
mod print_positions;
```
Now, no-one can again refer to those modules as they are private. To make them visible, re-export them using `pub use`, like this:
```rust
pub use self::apply_velocity::apply_velocity;
pub use self::apply_friction::apply_friction;
pub use self::apply_acceleration::apply_acceleration;
pub use self::print_positions::print_positions;
```

Here, `self::` means that we start traversing the module paths starting from the current module. *(We could also write `crate::systems::`, but `self::` is shorter)*. By re-exporting we have made the exported functions seem like they originate from the `systems`-module, instead of the submodules. Now, in `main.rs`, we can re-write the system imports as:
```rust
use crate::systems::{apply_velocity, print_positions, apply_friction, apply_acceleration};
```
which according to my personal subjective opinion, is a lot cleaner than the original. We have hidden the actual, more complex, internal module structure by re-exporting the contents of the internal modules from a common root-module. This allows us to keep the imports more readable, neat!

Additionally, we re-export the components from `components/mod.rs`, too:
```rust
mod position; // Note: no "pub" modifier
mod velocity;
mod friction;
mod acceleration;

pub use self::{
    position::PositionComponent,
    velocity::VelocityComponent,
    friction::FrictionComponent,
    acceleration::AccelerationComponent,
};
```

As `crate::` always allows us to refer to crate-local modules using absolute paths, the same principle can be used to import systems and components elsewhere in our codebase.

### Formatting print output on per-type basis
Currently, our `print_positions` formats the print output with this spell:
```rust
println!("Position: ({},{})", pos.x, pos.y)
```
Here, the `{}` will be substituted with a parameter, resulting in coordinates like
```
Position: (12.34567980123, 23.126356789)
```
We would like to get similar output with just *(this is the actual code we end up using in `print_positions`)*
```rust
println!("Position: {}", pos)
```

In Java, we would just override the `toString`-method and be done with it. But is there a `toString`-counterpart in Rust?

Obviously, there is. Rust has a `ToString` trait, but that rarely needs to be directly implemented. Instead, we implement the `Display` trait, which is actually meant for producing printable output from our types. As a bonus, just implementing the `Display` automagically implements the `ToString`, through a standard blanket implementation.

Implementing `Display` is quite straightforward, the only oddity is that we need to use the `write!`-macro, but apart from odd syntax this does not complicate things too much
```rust
impl Display for PositionComponent {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "Pos[value: {}]", self.value)
    }
}
```
Here, we just use the same string formattign syntax as with `println!` to nicely lay out what we are trying to print. Also, now that the `PositionComponent::value` is actually a `Vector2d` we can rely on its `Display` implementation *(which we have to write ourselves)*.

### 2D Vector (Math)!
As we are for the foreseeable future dealing with two dimensional coordinates, I decided to add a simple struct for helping with 2d vector math.
```rust
pub struct Vector2d {
    pub x: f64,
    pub y: f64,
}
```

Now, we could leave the definition as-is and be happy with it. The thing is, though, that we are for most of time going to use vector instances as values very much like primitive types *(`i32`, `f64`, etc.)*. Wouldn't it be quite nice if `Vector2d` would actually behave the same way, in situations like, say *(from rust book, chapter 4.2, "Stack-only data: Copy")*:
```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```

This would normally contradict ownershiprules, as `x` is being used while its value seems to have already moved to `y`. What happens here is that primitive types implement a special so-called `Copy`-trait which allows creating implicit value copies in situations like these. So, in above example, because `i32` has the `Copy`-trait, the assinment `y = x` does not move the value of `x` to the `y` but rather creates a copy of it.

Another special thing about the `Copy` trait is that it cannot be written manually. Copying is a standard language procedure, which in all cases is a full bit-by-bit copy of the original. If more special logic when creating copies is required, `Clone`-trait can be used to implement a `.clone()`-method. Then again, in simple cases, `Clone` implementation is trivial for anything implementing the `Copy` trait.

But if we cannot manually implement copy, then how are we supposed to make our `Vector2d` copyable? Here we utilize the convenient fact that `Copy` and `Clone` are *derivable traits*. Deriving traits means that their implementation is more-or-less trivial, and can easily be generated from the struct definition itself and/or other traits implemented on the struct. In this case, as we want both of the discussed traits, we add a derive annotation on our struct
```rust
#[derive(Copy, Clone)]
pub struct Vector2d {
    pub x: f64,
    pub y: f64,
}
```

Now, our `Vector2d` is copyable, making its use in calculation context much more convenient as we do not need worry about accidentally moving it when assigning it to a temporary value!

Speking of using it in calculations, next we would like to implement common operations like vector summation *(the "`+`"-operator)*, scalar multiplication *("`*`"-operator with `f64` RHS)* and for convenience, the "`+=`"-operator.

Standard library provides traits for all of these. First, the summation of two vectors. This allows the use of the "`+`"-operator
```rust
let a = Vector2d { x: 1, y: 2 };
let b = Vector2d { x: 3, y: 4 };

// Before:
let c = Vector2d { x: 0, y: 0 };
c.x = a.x + b.x;
c.y = a.y + b.y;

// After:
let c = a + b;
```

This is achieved using the `Add` trait. Implementation is straightworward:
```rust
impl Add for Vector2d {
    type Output = Vector2d;

    fn add(self, rhs: Self) -> Self::Output {
        Vector2d {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}
```
The trait has one associated type, which is used to declare what type of output is expected from the operator. In this case we expect the summation of two vectors to produce, unsurpisingly, an another vector.

Next up, the `Mul` trait, to allow scalar multiplication
```rust
let a = Vector2d { x: 1, y: 1 };
let scalar = 10.0;

// Before:
let b = Vector2d { x: 0, y: 0 };
b.x = a.x * scalar;
b.y = a.y * scalar;

// After:
let b = a * scalar;
```

And the implementation
```rust
impl Mul<f64> for Vector2d {
    type Output = Vector2d;

    fn mul(self, rhs: f64) -> Self::Output {
        Vector2d {
            x: self.x * rhs,
            y: self.y * rhs,
        }
    }
}
```
There is actually an interesting thing to note here. We do not necessarily need to provide the type parameter to `Mul` is actually optional and defaults to `Self`. This can be seen from the definition of `Mul`
```rust
pub trait Mul<Rhs = Self> {
    /* snip */
}
```
So, in cases where `Rhs` is not provided, it defaults to `Self`, which would in this case be `Vector2d`. We do not want that, so we provide it manually as `f64`. Now, if one was curious enough to peek at the definition of `Add`, there is a similar situation going on in there. As we implemented the addition of two vectors, the default `Self` was sufficient. For example, we could additionally write
```rust
impl Add<(f64, f64)> for Vector2d {
    /* snip */
}
```
which would be perfectly legitimate as `Add<Vector2d>` and `Add<(f64,f64)>` do not conflict!

The last one is something I actually didn't implement, but could be nice addition, at it could allow initializing temporary vectors inline, as 2-tuples. This would then result in something like
```rust
let a = Vector2d { x: 0, y: 0 };
let b = a + (10.0, 10.0);
```
but I'm not quite sure if that is just plain confusing or is it actually useful.

Implementing the `Sub` trait is very similar to the `Add` implementation.

Last trait we are implementing for now is the `AddAssign`, which enables use of the "`+=`" operator. This is a bit different from the other operators as this actually mutates the LHS. This is visible from the function signature as we take `&mut self` in instead of just `self`. Apart from that, there is nothing special here. *(Again, the RHS is an optional type parameter which we omit here, causing it to use the default value `Self`)*
```rust
impl AddAssign for Vector2d{
    fn add_assign(&mut self, rhs: Self) {
        self.x += rhs.x;
        self.y += rhs.y;
    }
}
```

*Fun fact: the implementations of operator traits for most of the situations in numeric types are basically the same, to the point where the standard library uses macros to generate them. For example, for `AddAssign` there is this one-liner*
```rust
add_assign_impl! { usize u8 u16 u32 u64 u128 isize i8 i16 i32 i64 i128 f32 f64 }
```
*which then leverages a single "generic" implementation of the trait and just substitutes the types where required*..


Actually, that was the last *operator*-trait we are going to implement. We still need to implement the `Display`-trait, remember? Luckily, it is quite trivial:
```rust
impl Display for Vector2d {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "({:.3}, {:.3})", self.x, self.y)
    }
}
```
We use the pattern `{:.3}` to tell the formatter that we want the `x` and the `y` limited to the precision of three decimal digits.

Now, let's put our vector to use. We change all components with `x` and `y` to use `value: Vector2d` instead.

Previously, in `apply_acceleration` we had
```rust
vel.x += acc.x;
vel.y += acc.y;
```
As we implemented the `AddAssign`-trait and changed the components to use vectors, we can replace this with just
```rust
vel.value += acc.value;
```
The same goes for `apply_velocity`.

There is much more going on in `apply_friction`, however. Let's see.
```rust
let velocity_length_squared = vel.x * vel.x + vel.y * vel.y;

if velocity_length_squared < f64::EPSILON {
    continue;
}
```
How about no. Let's simplify this by adding a method for our vector type. We can add our own methods directly to struct without specifying a trait like this
```rust
impl Vector2d {
    // methods here
}
```

Let's add a convenience method for calculating (squared) vector length. Inside the `impl`-block declared above, we add
```rust
pub fn length(&self) -> f64 {
    self.length_squared().sqrt()
}

pub fn length_squared(&self) -> f64 {
    self.x * self.x + self.y * self.y
}
```
*When length is defined as the square root of the squared length, I just love how trivial it is to implement :)*

Now, our length-zero-check can be written as:
```rust
if vel.value.length_squared() < f64::EPSILON {
    continue;
}
```

How about this:
```rust
let velocity_length = velocity_length_squared.sqrt();
let abs_friction_x = (vel.x / velocity_length * fri.amount).abs();
let abs_friction_y = (vel.y / velocity_length * fri.amount).abs();
```
We are calculating a new vector here. The `friction` is the amount of friction to apply, projected on the current direction, which in turn is calculated by normalizing the velocity. So, why not write just that, normalization, multiplication and taking a component-wise `abs`. As we want negative wriction to count as acceleration *for no reason whatsoever*, we take the `abs` before scalar multiplication.
```rust
let friction = vel.value.normalize().abs() * fri.value;
```

Now, we need the methods `normalize` and `abs` for our `Vector2d`. These are implemented as
```rust
pub fn normalize(&self) -> Self {
    let length = self.length();
    Vector2d {
        x: self.x / length,
        y: self.y / length,
    }
}

pub fn abs(&self) -> Self {
    Vector2d {
        x: self.x.abs(),
        y: self.y.abs(),
    }
}
```

Ok! Calculating the length of the new velocity *(magnitude)*. For this we are going to implement component-wise `max`
```rust
pub fn max(&self, max: f64) -> Self {
    Vector2d {
        x: self.x.max(max),
        y: self.y.max(max),
    }
}
```

What we had before:
```rust
let magnitude_x = (vel.x.abs() - abs_friction_x).max(0.0);
let magnitude_y = (vel.x.abs() - abs_friction_y).max(0.0);
vel.x = vel.x.signum() * magnitude_x;
vel.y = vel.y.signum() * magnitude_y;
```

What we can write now:
```rust
let magnitude = (vel.value.abs() - friction).max(0.0);
vel.value.x = vel.value.x.signum() * magnitude.x;
vel.value.y = vel.value.y.signum() * magnitude.y;
```

Thus, our `apply_friction` has been reduced down to:
```rust
if vel.value.length_squared() < f64::EPSILON {
    continue;
}

let friction = vel.value.normalize().abs() * fri.value;
let magnitude = (vel.value.abs() - friction).max(0.0);
vel.value.x = vel.value.x.signum() * magnitude.x;
vel.value.y = vel.value.y.signum() * magnitude.y;
```


What next?
----------
Now that we have split everything to modules, things are starting to look tidy enough for us to start actually doing things again. Next up, we start to poke at getting rid of the `while let`-loops and replace those with something that scales a bit better in the future.

The `From`-trait seems not to have been very good idea for components, probably going to scrap it in the next part. For this part, I was lazy and just changed the existing implementations to use `Vector2d` where applicable.

The full source code can be found in branch `part-2` ([link](https://github.com/Kailari/kokonaisuus/tree/part-2)).

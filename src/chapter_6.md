Component `TypeId`
==================
*"First draft for generalized storage access"*

### Topics
 - `trait Any`
 - `RefCell<T>` for interrior mutability of immutable references
 - `dyn Trait` *(Trait objects)*

What are we trying to do?
-------------------------
First step towards getting the dispatcher working in an automated manner is to unify the way we access the component storage. This is seemingly though issue from the perspective of the ownership and borrow-checker, as we simultaneoulsy require references to multiple component vectors stored inside the component storage. *(We soon learn that mutably borrowing just one storage using a "getter" method renders the rest of the storage inaccessible)*

So, what we are going to do this time is to try and come up with some way of fetching the relevant component storage vector, knowing just its type. When we are finished, ticking the `apply_acceleration` looks like:
```rust,noplaypen
self.apply_acceleration.tick((
    storage.fetch_mut::<VelocityComponent>().as_mut(),
    storage.fetch_ref::<AccelerationComponent>().as_ref(),
));
```
*(Note that the component type is resolved just from the type! We do not provide any arguments to the `fetch_ref`/`fetch_mut` methods, they get the component type from the generic parameter!)*


Doing it the wrong way
----------------------
Let's see, what do we currently have?
```rust,noplaypen
pub fn dispatch(&self, components: &mut ComponentStorage) {
#     println!("State before tick:");
#     self.print_state.tick((&components.positions, &components.velocities, &components.accelerations, &components.frictions));
#     self.apply_acceleration.tick((&mut components.velocities, &components.accelerations));
#     self.apply_friction.tick((&mut components.velocities, &components.frictions));
    // ...
    self.apply_velocity.tick((&mut components.positions, &components.velocities));
    // ...
#     println!("\nPositions after tick:");
#     self.print_positions.tick(&components.positions);
}
```
We refer to the individual component storage vectors by directly getting them from the `ComponentStorage` struct and borrowing them directly. This works, but on the long run we won't have such fields anymore so an unified way of accessing the storage is required.

First, we create naïve getters for those storages to see what issues there could be when direct references are no longer used. We add two new methods to the component storage:
```rust,noplaypen
impl ComponentStorage {
    /* snip */

    pub fn fetch_mut_positions(&mut self) -> &mut Vec<PositionComponent> {
        &mut self.positions
    }

    pub fn fetch_ref_velocities(&self) -> &Vec<VelocityComponent> {
        &self.velocities
    }
}
```

Then, when we try to change the dispatcher to use these, this happens:
```rust,noplaypen,does_not_compile
self.apply_velocity.tick((
    storage.fetch_mut_positions(),
    storage.fetch_ref_velocities(),
));
```
```
error[E0502]: cannot borrow `*storage` as immutable because it is also borrowed as mutable
  --> src\dispatcher.rs
|
|           self.apply_velocity.tick((
|  __________________________________-
| |             storage.fetch_mut_positions(),
| |             ------- mutable borrow occurs here
| |             storage.fetch_ref_velocities(),
| |             ^^^^^^^ immutable borrow occurs here
| |         ));
| |_________- mutable borrow later used here
```
Here the compiler knows less than we do. Due to the way we store our vectors inside a single struct, compiler sees the first "getter" as mutable borrow *for the storage wrapper* which then further borrows the component vector. The mutable borrow for the storage wrapper stays in effect until the component vector borrow ends.

...which in turn prevents us from borrowing the another vector, because we cannot borrow the storage wrapper for borrowing its contents!

This is bit annoying, as to us, at least, it is clear that there is no risk of simultaneous mutable access to either of the vectors. But what can we do?

Well, when you think of it, we are mutating and accessing the *contents* of the storage, but the storage itself stays immutable for the most part. What we wish to do here is *interrior mutability of an immutable object*. That is, we wish to allow mutating the vectors' contents, but not adding or removing new vectors to the storage.

### Standard library interrior mutability: `RefCell<T>`

Luckily again, the standard library got our back here. `RefCell<T>` is meant for doing just what we are trying to accomplish here; skipping the borrow-checking at compile time and trusting that we know what we are doing and performing the borrow-checkign dynamically at runtime.

Sounds scary, huh? Well, we are sacrificing a bit of our compile-time safety-net for more flexible way of doing things. All in all, as long as you don't try to fetch same storage mutably two times, or try to provide the same storage mutably **and** immutably to the same system, there should be no risk of accidentally causing panics with the runtime borrow checker.

*There are actually a lot of cases what can go wrong, but we'll try to eliminate those with safe wrappers later on!*

Now, even though the implications are quite complex, the usage itself is quite straightforward. First, change our component vectors to `RefCell<T>`s:
```rust,noplaypen
pub struct ComponentStorage {
    positions: RefCell<Vec<PositionComponent>>,
    velocities: RefCell<Vec<VelocityComponent>>,
    accelerations: RefCell<Vec<AccelerationComponent>>,
    frictions: RefCell<Vec<FrictionComponent>>,
}

impl ComponentStorage {
    pub fn new() -> ComponentStorage {
        ComponentStorage {
            positions: RefCell::new(vec![
                // ...
#                PositionComponent::new(0.0, 0.0),
#                PositionComponent::new(-42.0, -42.0),
#                PositionComponent::new(234.0, 123.0),
#                PositionComponent::new(6.0, 9.0),
            ]),
            velocities: RefCell::new(vec![
                // ...
#                 VelocityComponent::new(40.0, 10.0),
#                 VelocityComponent::new(30.0, 20.0),
#                 VelocityComponent::new(20.0, 30.0),
#                 VelocityComponent::new(10.0, 40.0),
            ]),
            frictions: RefCell::new(vec![
                // ...
#                 FrictionComponent::new(1.0),
#                 FrictionComponent::new(2.0),
#                 FrictionComponent::new(3.0),
#                 FrictionComponent::new(4.0),
            ]),
            accelerations: RefCell::new(vec![
                // ...
#                 AccelerationComponent::new(2.0, 16.0),
#                 AccelerationComponent::new(4.0, 2.0),
#                 AccelerationComponent::new(8.0, 4.0),
#                 AccelerationComponent::new(16.0, 8.0),
            ]),
        }
    }
```

There are slight changes to the getters too:
```rust,noplaypen
pub fn fetch_positions_mut(&self) -> RefMut<Vec<PositionComponent>> {
    self.positions.borrow_mut()
}

pub fn fetch_velocities_ref(&self) -> Ref<Vec<VelocityComponent>> {
    self.velocities.borrow()
}
```

We use the `borrow()` methods to indicate a *runtime borrow*. This then returns us a `Ref` or `RefMut` which are simply just *"wrapper types for mutably/immutably borrowed value from `RefCell`"*. This has something to do with the runtime borrow checking, I do not understand this fully, but `RefCell` requires ability to track when its mutable/immutable references go out of scope and these wrappers allow implementing that. Or something.

The important thing is, if we tried to do this instead to match the previous implementation:
```rust,does_not_compile,noplaypen
pub fn fetch_positions_mut(&self) -> &mut Vec<PositionComponent> {
    self.positions.borrow_mut().as_mut()
}

pub fn fetch_velocities_ref(&self) -> &Vec<VelocityComponent> {
    self.velocities.borrow().as_ref()
}
```
```
error[E0515]: cannot return value referencing temporary value
  --> src\component_storage.rs
 |
 |         self.positions.borrow_mut().as_mut()
 |         ---------------------------^^^^^^^^^
 |         |
 |         returns a value referencing data owned by the current function
 |         temporary value created here

error[E0515]: cannot return value referencing temporary value
  --> src\component_storage.rs
 |
 |         self.velocities.borrow().as_ref()
 |         ------------------------^^^^^^^^^
 |         |
 |         returns a value referencing data owned by the current function
 |         temporary value created here
```
The wrapped `Ref` and `RefMut` need to be alive and in scope in order to use the wrapped references. Here, the wrappers are dropped as the method ends, causing the returned references to point to potentially inaccessible data. This is a no-go, so just return the wrappers.

Also, I almost forgot, the most important thing: In the `fetch_mut_positions` now takes the `&self` as immutable reference!

Now, we add the `as_mut` and `as_ref` calls to our dispatcher:
```rust,noplaypen
self.apply_velocity.tick((
    storage.fetch_mut_positions().as_mut(),
    storage.fetch_ref_velocities().as_ref(),
));
```
Our investigation allowed us to resolve issues around interrior mutability before taking on the more complex concepts. Sadly, we only have methods for getting positions and velocities so we still need to figure out how to select the correct vector based on type.


### Dynamic typing

What we are trying to accomplish is essentially dynamic, runtime typing. More sacrifices to compile-time safety-nets, yay!

As showcased at the start of this part, we are going to implement something like:
```rust,noplaypen
pub fn fetch_mut<C>(&self) -> RefMut<Vec<C>> {
    // ...
}

pub fn fetch_ref<C>(&self) -> Ref<Vec<C>> {
    // ...
}
```

Now, how on earth can we take the type `C` and convince the compiler to believe it is, for example, `PositionComponent`. Or on the other hand, how can we figure out ourselves that the `C` is some of our component types?

Again, standard library provides us with some useful utilities. This time, the `Any`-trait and `TypeId` type. Compiler assigns each type an unique ID, the `TypeId` is essentially this identifier, thus, we can do comparisons like:
```rust,noplaypen
let component_type_id = TypeId::of::<C>();

if component_type_id == TypeId::of::<PositionComponent>() {
    do_something();
}
```
Here, we call the `TypeId::of()` associated function, providing an explicit generic parameter. This is so-called *turbofish*-syntax *(The `::<>` looks a bit like a speeding fish)*.

Great! The requirement for being able to use the `TypeId::of` is that the type parameter must implement the `Any` trait, thus, we add a trait bound on the type parameter:
```rust,noplaypen
pub fn fetch_mut<C>(&self) -> RefMut<Vec<C>>
    where C: Any
{
    // ...
}
```

Our `fetch_mut` and `fetch_ref` are very similar in implementation. We can write both as:
```rust,noplaypen
pub fn fetch_mut<C>(&self) -> RefMut<Vec<C>>
    where C: Any
{
    self.fetch_component_storage::<C>()
        .borrow_mut()
}

pub fn fetch_ref<C>(&self) -> Ref<Vec<C>>
    where C: Any
{
    self.fetch_component_storage::<C>()
        .borrow()
}
```
Both use magical `self.fetch_component_storage::<C>()` utility method, which returns us a `RefCell` which we then borrow either mutably or immutably.

The only thing left to implement is the actual `fetch_component_storage`, for getting the appropriate `RefCell`. Now, how could we resolve `C` or more accurately `TypeId::of::<C>()` to a `RefCell<Vec<C>>`? 

For now, as our component vectors are in individual struct fields, we are going full-retard here. Just use `if-elseif` for now and start looking into actual generic component storage later on.
```rust,noplaypen
fn fetch_component_storage<C>(&self) -> &RefCell<Vec<C>>
    where C: Any
{
    let component_type_id = TypeId::of::<C>();
    let storage =
        if component_type_id == TypeId::of::<PositionComponent>() {
            // return positions
        } else if component_type_id == TypeId::of::<VelocityComponent>() {
            // return velocities
        } else if component_type_id == TypeId::of::<AccelerationComponent>() {
            // return accelerations
        } else if component_type_id == TypeId::of::<FrictionComponent>() {
            // return frictions
        } else {
            panic!("Unknown component type!")
        };

    // Do something with `storage` and return it as `RefCell<Vec<C>>`
}
```
Now, there is one slight issue, what is the type of `storage` here? Intuition is to just try something like
```rust,noplaypen
if component_type_id == TypeId::of::<PositionComponent>() {
    &self.positions
} else if component_type_id == TypeId::of::<VelocityComponent>() {
    &self.velocities
} // ...
```
...and rely on the compiler to figure out itself what we are trying to do. But it cannot, it's not smart enough for that. Instead, we are going to use information we have but the compiler does not understand.

Note that:
 1. If the `if` expression passes, we know for sure that the component type in question and `C` are actually the same type *(If type IDs match, then the two types must be the exact same type)*
 2. We are returning *references* to the vectors, not vectors themselves.

Now, how are these two points useful? The first one means that if we end up returning a component vector from the if-expression, we know for sure that the returned vector is of type `Vec<C>`. The compiler does not know that, but we can make use of that information anyway. The second point allows us to do wacky stuff with the types. References themselves have constant size, known at compile time, regardless of what type of data they point to. Thus, we can practically store reference to any type of object in a single variable.

In order to utilize either of the observations, we are going to need the `Any` trait. Luckily, almost everything implements the `Any` trait out of the box, so we can just:
```rust,noplaypen
let storage =
    if component_type_id == TypeId::of::<PositionComponent>() {
        &self.positions as &dyn Any
    } else if component_type_id == TypeId::of::<VelocityComponent>() {
        &self.velocities as &dyn Any
    } else if component_type_id == TypeId::of::<AccelerationComponent>() {
        &self.accelerations as &dyn Any
    } else if component_type_id == TypeId::of::<FrictionComponent>() {
        &self.frictions as &dyn Any
    } else {
        panic!("Unknown component type!")
    };
```
Now, the type of `storage` is a reference to `dyn Any`, that is:
```rust,noplaypen
let storage: &dyn Any = // ...
```
This seems confusing, why are we allowed to cast our component vectors to `dyn Any` and what on earth is `dyn Any` anyways!?


### Trait objects

We are dealing with a concept called *trait objects*. Here `Any` is a trait, and `dyn Any` simply refers to *"Some type, which implements `Any`"*. As our component storages *(which have the type `RefCell<Vec<ComponentType>>`)* happen to be, well "Any type", they implement the `Any` trait *(There are actually few rules on which types are considered `Any`, refer to `std::any` module documentation if interested)*. As refences have constant size, after we cast individual component storage references to `&dyn Any`, they are actually all the same type.

Then again, after getting the correct concrete storage type, we do not care about what type it *actually* is; we are going to return it as `&RefCell<Vec<C>>` anyways!

Another slight issue though, how do we convert the `&dyn Any` to our return value? As long as we do not mess anything up with our if-elseif mess, we *do* know that the types are compatible. But how do we convince the compiler?

Again, we rely on the `Any`-trait. The standard library implements a few useful utilities on the `dyn Any` trait object *(yes, trait objects actually can have associated functions and methods!)*.

Enough blabber, here, now that `storage` is a `Any` trait-object, we can just *downcast* it to our desired type, as we already know that they should be compatible. The full implementation is then:
```rust,noplaypen
pub fn fetch_mut<C>(&self) -> RefMut<Vec<C>>
    where C: Any
{
    self.fetch_component_storage::<C>()
        .borrow_mut()
}

pub fn fetch_ref<C>(&self) -> Ref<Vec<C>>
    where C: Any
{
    self.fetch_component_storage::<C>()
        .borrow()
}

fn fetch_component_storage<C>(&self) -> &RefCell<Vec<C>>
    where C: Any
{
    let component_type_id = TypeId::of::<C>();
    let storage =
        if component_type_id == TypeId::of::<PositionComponent>() {
            &self.positions as &dyn Any
        } else if component_type_id == TypeId::of::<VelocityComponent>() {
            &self.velocities as &dyn Any
        } else if component_type_id == TypeId::of::<AccelerationComponent>() {
            &self.accelerations as &dyn Any
        } else if component_type_id == TypeId::of::<FrictionComponent>() {
            &self.frictions as &dyn Any
        } else {
            panic!("Unknown component type!")
        };

    storage.downcast_ref::<RefCell<Vec<C>>>()
           .expect("Downcasting storage RefCell failed!")
}
```
*The `.expect` at the end should actually be impossible to reach as unknown component types cause panic anyway.*

With this done, we can now remove the `pub` modifier from our component storage fields
```rust,noplaypen
pub struct ComponentStorage {
    positions: RefCell<Vec<PositionComponent>>,
    velocities: RefCell<Vec<VelocityComponent>>,
    accelerations: RefCell<Vec<AccelerationComponent>>,
    frictions: RefCell<Vec<FrictionComponent>>,
}
```

and proceed to change all our system tick calls to use the new methods in the dispatcher
```rust,noplaypen
pub fn dispatch(&self, storage: &mut ComponentStorage) {
    println!("State before tick:");
    self.print_state.tick((
        storage.fetch_ref::<PositionComponent>().as_ref(),
        storage.fetch_ref::<VelocityComponent>().as_ref(),
        storage.fetch_ref::<AccelerationComponent>().as_ref(),
        storage.fetch_ref::<FrictionComponent>().as_ref(),
    ));

    self.apply_acceleration.tick((
        storage.fetch_mut::<VelocityComponent>().as_mut(),
        storage.fetch_ref::<AccelerationComponent>().as_ref(),
    ));

    self.apply_friction.tick((
        storage.fetch_mut::<VelocityComponent>().as_mut(),
        storage.fetch_ref::<FrictionComponent>().as_ref(),
    ));

    self.apply_velocity.tick((
        storage.fetch_mut::<PositionComponent>().as_mut(),
        storage.fetch_ref::<VelocityComponent>().as_ref(),
    ));

    println!("\nPositions after tick:");
    self.print_positions.tick(
        storage.fetch_ref::<PositionComponent>().as_ref()
    );
}
```
Sadly as we are mixing raw references and wrapped `Ref` and `RefMut` from `RefCell`, we need those ugly `as_ref` and `as_mut` calls everywhere. I don't see that as much of an issue for now though.


### A few gotchas

We are doing relatively unsafe stuff in `fetch_mut`, `fetch_ref` and `fetch_component_storage` methods. First of all, `fetch_mut` and `fetch_ref` can fail in multitude of different situations, for example:
```rust,noplaypen,panics
let positions_ref = storage.fetch_ref::<PositionComponent>().as_ref();
let positions_mut = storage.fetch_mut::<PositionComponent>().as_mut();
//                          ^^^^^^^^^
// The compiler does not complain, but we cannot bypass borrowing rules,
// even when we are using `RefCells`. This panics at runtime.
```
The code panics on the second line as we are trying to fetch mutably while immutable borrow is still alive. Fetching multiple times mutably is valid, though:
```rust,noplaypen
let positions_ref_a = storage.fetch_ref::<PositionComponent>().as_ref();
let positions_ref_b = storage.fetch_ref::<PositionComponent>().as_ref();
```

The first example causes issues if we were to move the storage accesses to temporary variables:
```rust,panics,noplaypen
let mut velocities = storage.fetch_mut::<VelocityComponent>();
let frictions = storage.fetch_ref::<FrictionComponent>();
self.apply_friction.tick((
    velocities.as_mut(),
    frictions.as_ref(),
));

let mut positions = storage.fetch_mut::<PositionComponent>();
let velocities = storage.fetch_ref::<VelocityComponent>(); // panics!
self.apply_velocity.tick((
    positions.as_mut(),
    velocities.as_ref(),
));
```
This example is a bit more interesting. Intuitively, the original velocities has already been exhausted as we are shadowing it with a new one anyway. But things do not work that way! The original `velocities` won't be dropped until it goes out of scope, in this case, that is at the same time as the new `velocities` we shadow the original with. This, obviously is not valid as the original is still borrowed while we try to borrow it again.

We can fix this, however, and even without creating any new methods or anything stupid:
```rust,noplaypen
{
    let mut velocities = storage.fetch_mut::<VelocityComponent>();
    let frictions = storage.fetch_ref::<FrictionComponent>();
    self.apply_friction.tick((
        velocities.as_mut(),
        frictions.as_ref(),
    ));
} // <-- `velocities` is dropped here

let mut positions = storage.fetch_mut::<PositionComponent>();
let velocities = storage.fetch_ref::<VelocityComponent>();
self.apply_velocity.tick((
    positions.as_mut(),
    velocities.as_ref(),
));
```
This is valid code and should compile just fine. We use a relatively common trick, which we can use when writing sequences like these where we do not want to extract things to their own methods but would like previous values to be dropped before starting work on next ones. We simply define a new scope to limit the lifetime of the storage references. Now, the second definition of `velocities` is not actually shadowing the original anymore, as the original has already gone out of scope and thus has been dropped!

So, the takeaway here is that one must be careful with `storage.fetch_XXX` methods, as they can easily cause panics at runtime. I have not figured out how this could be done safely, or how I could get the compiler to complain about trying to simultaneously fetch the same component storage two times.

Then onto the next topic; the `fetch_component_storage`. The trickery we do is luckily limited to the scope of a single method, thus its not *that* huge of an issue. However, I strongly dislike the approach we took here. We momentarily lose type-safety almost completely here. Yes, the application immediately crashes in case we manage to accidentally return wrong type of vector, but there is no compile-time guarantee that we have written code that is actually safe. I *might* invest some time in the future to create some *(at least a bit)* safer wrapper for doing this conversion, but that must wait for after we have finished with generalizing the storage.


What next?
----------
Ok! We got a huge milestone out of the way here; we have API we can use to fetch arbitrary component types from our component storage. Yes, the storage currently only supports four different component types, but we are not far from being able to support any number of components. The dispatcher is getting messy, but we are not going to touch on that just yet.

For the next part, we are going to focus on packing the individual component storage vectors to some collection so that we can truly support arbitrary component types. This requires some registration procedure for component types and a few changes to way our `fetch_component_storage` works, but should be quite simple change now that we have the framework laid down.

The full source code can be found in branch `part-7` ([link](https://github.com/Kailari/kokonaisuus/tree/part-7))

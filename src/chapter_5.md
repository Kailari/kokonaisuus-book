The Lifetime of `'a` system
===========================
*"Defining the basic concept of a system"*

### Topics
 - Lifetime annotations
 - Lifetime elision
 - *(bonus)* Upcoming language feature: GAT

What are we trying to do?
-------------------------
The systems in our current implementation have nothing in common. We are going to change that by making each system their own struct and implementing a `System` trait on them. This allows us to have a single unified `System::tick`-method we can call on any system.

This complicates parameter definition for the system tick function, however, and we are required to add *lifetime annotations* to few places to make the compiler understand that references to component storage vectors are alive for long enough.

Also, we are adding the first *"dumb"-implementations* for the dispatcher and the component storage by moving system ticking and component initialization and storage to their own modules, out of the `main`.

Spring cleaning, vol. 2
-----------------------
First, in `main.rs`, we create two new submodules:
```rust,noplaypen
mod component_storage;
mod dispatcher;
```

Then, today's goal is to replace the contents of our `pub fn main()` with
```rust,noplaypen
pub fn main() {
    let mut components = ComponentStorage::new();
    let dispatcher = Dispatcher::new();
    dispatcher.dispatch(&mut components);
}
```

Let's start with the component storage. In our new module `component_storage.rs`, we define a simple wrapper for our storage vectors:
```rust,noplaypen
pub struct ComponentStorage {
    pub positions: Vec<PositionComponent>,
    pub velocities: Vec<VelocityComponent>,
    pub accelerations: Vec<AccelerationComponent>,
    pub frictions: Vec<FrictionComponent>,
}
```

Then, for convenience, we add a `::new()` associated function for constructing the default instance. What we do here is just cut-paste the old component initialization code, and use that to initialize a `ComponentStorage`

```rust,noplaypen
impl ComponentStorage {
    pub fn new() -> ComponentStorage {
        ComponentStorage {
            positions: vec![
                // ...
#                PositionComponent::new(0.0, 0.0),
#                PositionComponent::new(-42.0, -42.0),
#                PositionComponent::new(234.0, 123.0),
#                PositionComponent::new(6.0, 9.0),
            ],
            velocities: vec![
                // ...
#                VelocityComponent::new(40.0, 10.0),
#                VelocityComponent::new(30.0, 20.0),
#                VelocityComponent::new(20.0, 30.0),
#                VelocityComponent::new(10.0, 40.0),
            ],
            frictions: vec![
                // ...
#                FrictionComponent::new(1.0),
#                FrictionComponent::new(2.0),
#                FrictionComponent::new(3.0),
#                FrictionComponent::new(4.0),
            ],
            accelerations: vec![
                // ...
#                AccelerationComponent::new(2.0, 16.0),
#                AccelerationComponent::new(4.0, 2.0),
#                AccelerationComponent::new(8.0, 4.0),
#                AccelerationComponent::new(16.0, 8.0),
            ],
        }
    }
}
```

Ok! We are going more in-depth with the component storage in the next part, so we're keeping it at that for now. Next up, unifying the systems.


### Defining a `System` trait

So, systems are *stateless mutators* that take in varying selection of different components and perform some data mutation. The state after ticking is considered the output of a system and due to statelessness, should always be the same, assuming the same input. That is, the systems should be *deterministic*: The same input should always produce the same output.

The latter point is bit far-fetched for now, but good to keep in mind. For now, let's focus on being able to tick the system.

To allow systems to define arbitrary data as inputs *(different systems need different storage vectors)*, we define the inputs as an *associated type*. Now, first draft of our system trait can be written as:
```rust,noplaypen
pub trait System {
    type InputData;

    fn tick(&self, data: Self::InputData);
}
```

Now, there is a slight flaw with our approach right now due to limitations of associated types. Let's try implementing `apply_acceleration` as a `System` to see what's wrong.

First, define an empty struct to act as our system type:
```rust,noplaypen
pub struct ApplyAccelerationSystem;
```
Later on, we could add things like configuration parameters to our systems, but for now, the struct can very well be "empty".

Now, the system implementation. *Following code, while mostly correct, does not compile*
```rust,does_not_compile
impl System for ApplyAccelerationSystem {
    type InputData = (&mut Vec<VelocityComponent>,
                      &Vec<AccelerationComponent>);

    fn tick(&self, (velocities, accelerations): Self::InputData) {
        let vel_iter = velocities.iter_mut();
        let acc_iter = accelerations.iter();

        for (vel, acc) in IterTuple::from((vel_iter, acc_iter)) {
            vel.value += acc.value;
        }
    }
}
```
```
error[E0106]: missing lifetime specifier
|     type InputData = (&mut Vec<VelocityComponent>,
|                       ^ expected lifetime parameter
```

The compiler complains something about requiring explicit lifetimes for references. What does that even mean? Well, the culprit is the fact that our parameter types *(which are references)* have now moved out of the method definition, into scope of the implementation block, and their lifetimes are now non-self-evident, as the compiler does not know where they will be used. If that does not make any sense, fear not, this is unintuitive as heck at first. *We actually briefly touched on the following in the first part, but here it is again:*.

Let's see, what did we have before?
```rust,noplaypen
pub fn apply_acceleration(vs: &mut Vec<VelComp>, as: &Vec<AccComp>) { // ...
```
Here, it is clear *(self-evident from the context)* that the references are going to live for the duration of the method call. Due to that, the compiler is actually able to apply some syntax-sugar called *parameter lifetime elision* to our method signature. What the signature would look like de-sugared to its full form is actually:
```rust,noplaypen
pub fn apply_acceleration<'a>(vs: &'a mut Vec<VelComp>, as: &'a Vec<AccComp>) { // ...
```
which roughly translates to:
```rust,noplaypen
pub fn apply_acceleration<'a>(vs: &'a mut Vec<VelComp>, as: &'a Vec<AccComp>)
{   // lifetime 'a starts

    // everything with lifetime 'a is guaranteed to be alive

}   // lifetime 'a ends, reference parameters with lifetime 'a go out of scope
    // and as they are dropped, their borrow ends
```

However, as said before, as it is self-evident from the context of the function signature that the parameters are probably going to have the same lifetime as the function, the lifetimes can normally be *elided*.

But now that the parameter type is actually an associated type, we no longer have the context of the function at our disposal when defining the type! Well, let's try to fix that by adding a lifetime parameter to our `impl`:
```rust,does_not_compile,noplaypen
impl<'a> System for ApplyAccelerationSystem {
    type InputData = (&'a mut Vec<VelocityComponent>,
                      &'a Vec<AccelerationComponent>);

    fn tick(&self, (velocities, accelerations): Self::InputData) {
        /* snip */
#         let vel_iter = velocities.iter_mut();
#         let acc_iter = accelerations.iter();
# 
#         for (vel, acc) in IterTuple::from((vel_iter, acc_iter)) {
#             vel.value += acc.value;
#         }
    }
}
```
```
error[E0207]: the lifetime parameter `'a` is not constrained by the impl trait, self type, or predicates
| impl<'a> System for ApplyAccelerationSystem {
|      ^^ unconstrained lifetime parameter
```
...and it does not compile. Remember when we discussed about how generic parameters must be *constrained* by the types being used in the implementation definition? Well, here the problem is that the lifetime `'a` is not being constrained by neither `System` trait nor `ApplyAccelerationSystem`.

The simple solution is to add dummy lifetime parameter to our `System`-trait:
```rust,noplaypen
pub trait System<'a> { // <-- we added lifetime 'a
    type InputData;

    fn tick(&self, data: Self::InputData);
}
```

Now, we can constrain the lifetime in the implementation
```rust,noplaypen
impl<'a> System<'a> for ApplyAccelerationSystem {
    type InputData = (&'a mut Vec<VelocityComponent>,
                      &'a Vec<AccelerationComponent>);

    fn tick(&self, (velocities, accelerations): Self::InputData) {
        /* snip */
#         let vel_iter = velocities.iter_mut();
#         let acc_iter = accelerations.iter();
# 
#         for (vel, acc) in IterTuple::from((vel_iter, acc_iter)) {
#             vel.value += acc.value;
#         }
    }
}
```
Now it actually compiles! Looks confusing? Yes, but it works and is exactly how we are going to leave it for the foreseeable future.

Lastly, one might wonder, what on earth is going on with the method `tick` parameters:
```rust,noplaypen
fn tick(&self, (velocities, accelerations): Self::InputData) { // ...
```
well, as we know what the `Self::InputData` is going to contain, and accessing the tuple items by indexing the tuple would look messy, we use pattern matching to destructure the tuple into named arguments. These have types from the `Self::InputData`, as in, what we are doing is equivalent to:
```rust,noplaypen
fn tick(&self, data: Self::InputData) {
    let velocities: &'a mut Vec<VelocityComponent> = data.0;
    let accelerations: &'a Vec<AccelerationComponent> = data.1;

    // ...
#     let vel_iter = velocities.iter_mut();
#     let acc_iter = accelerations.iter();
# 
#     for (vel, acc) in IterTuple::from((vel_iter, acc_iter)) {
#         vel.value += acc.value;
#     }
}
```

Now, after doing similar implementations to other systems, we can implement a simple "dispatcher" using the `ComponentStorage` from before. *(Actually, we just sequentially call `tick()` on all our systems, just like before, but at least it is inside a nice wrapper)*.

### Dispatcher

First, for now, the struct is just wrapper for storing all our systems *(and the `new` function implementation is trivial as our systems do not contain data)*:
```rust,noplaypen
pub struct Dispatcher {
    print_state: PrintStateSystem,
    print_positions: PrintPositionsSystem,
    apply_acceleration: ApplyAccelerationSystem,
    apply_friction: ApplyFrictionSystem,
    apply_velocity: ApplyVelocitySystem,
}

# impl Dispatcher {
#     pub fn new() -> Dispatcher {
#         Dispatcher {
#             print_state: PrintStateSystem,
#             print_positions: PrintPositionsSystem,
#             apply_acceleration: ApplyAccelerationSystem,
#             apply_friction: ApplyFrictionSystem,
#             apply_velocity: ApplyVelocitySystem,
#         }
#     }
# }
```

And the `dispatch` method is then just copy-paste of our original system function calls, but everything is now a call to `System::tick` and component vectors are in `components`:
```rust,noplaypen
# impl Dispatcher {
#     pub fn new() -> Dispatcher {
#         Dispatcher {
#             print_state: PrintStateSystem,
#             print_positions: PrintPositionsSystem,
#             apply_acceleration: ApplyAccelerationSystem,
#             apply_friction: ApplyFrictionSystem,
#             apply_velocity: ApplyVelocitySystem,
#         }
#     }

    pub fn dispatch(&self, components: &mut ComponentStorage) {
        println!("State before tick:");
        self.print_state.tick((&components.positions, &components.velocities, &components.accelerations, &components.frictions));

        self.apply_acceleration.tick((&mut components.velocities, &components.accelerations));
        self.apply_friction.tick((&mut components.velocities, &components.frictions));
        self.apply_velocity.tick((&mut components.positions, &components.velocities));

        println!("\nPositions after tick:");
        self.print_positions.tick(&components.positions);
    }
# }
```

This should be enough of a foundation we can start building the actual storages and dispatcher on.


### Bonus: More about lifetimes

One might wonder:
> *"Ok, in the system implementation earlier `'a` was a lifetime defined in the trait. But what does it actually represent? It is lifetime of **what**, like actually?"*

Now, that right there, is an exquisite question!
*Short answer:* lifetime of the reference, whatever it means in the context where it is used
*Long answer:* feeling adventurous? Read along!

#### OUT OF SCOPE ALERT
*I personally don't fully undestand everything we discuss here. This explanation is based on my own, limited, understanding of what is going on. Take everything with a grain of salt.*

This particular situation is a very interesting corner-case of the current way of how associated types work. We have very well-defined context where our `type InputData` will be used, but then again, we have no way of conveying things like lifetimes from that context onto that type. That is, there is no concept of generics when dealing with associated types.

Let's think for this a bit. Before, the lifetimes were *self-evident from the method context*. Has our situation changed in some way so that it should no longer be the case? Answer is actually "no". However, Rust as a language is currently not able to accurately represent what we are trying to do.

Why are the lifetimes non- self-evident in our new implementation?

This is due to our choice of using an associated type for ensuring that system with any kind of input data can be called with the same `System::tick(&self, data: Self::InputData)`-method. More precisely, the associated type does not know about the lifetime of any particular call to the tick- method, thus it needs to have "just some arbitrary lifetime", which we seemingly just make out of thin air.

This "making things out of thin air" is obviously, sub-optimal.

Now, in context of a call to `tick()`, the `'a` lifetime **actually represents just what it did earlier**, the lifetime of the method. We just need to have the extra lifetime parameter on the trait for seemingly no reason at all, **in order to be able to define the associated type** `InputData` properly.

#### EVEN MORE OUT OF SCOPE ALERT
*Next up, some quite specific topics I don't fully understand, on unfinished language features that likely won't be stable for couple more years.*

In distant future, what we could do is to utilize a upcoming language feature called *"generic associated types" (or GAT in short)*. This would allow us to define associated types with generics, so we could then write something like:
```rust,noplaypen
trait System {
    type InputData<'a>;

    fn tick<'a>(&self, _: Self::InputData<'a>);
}

impl System for ApplyAccelerationSystem {
    type InputData<'a> = (&'a mut Vec<VelComp>, &'a Vec<AccComp>)

    fn tick<'a>(&self, (vs, ac): Self::InputData<'a>) {
        // ...
    }
}
```
Note that the trait does not have the lifetime parameter anymore, anyone using the `InputData` associated type now needs to provide their own lifetime! This removes a lot of ambiguity from what the lifetime `'a` actually represents. As an added bonus, it might even be possible to elide the lifetime in the definition of the `tick` again *(as it should be self-evident from the context again!)*, so that the code then sugars to:
```rust,noplaypen
trait System {
    type InputData<'a>;

    fn tick(&self, _: Self::InputData);
}

impl System for ApplyAccelerationSystem {
    type InputData<'a> = (&'a mut Vec<VelComp>, &'a Vec<AccComp>)

    fn tick(&self, (vs, ac): Self::InputData) {
        // ...
    }
}
```

But sadly, there is currently a ton of issues blocking the progress on GATs, so we won't be able to do this code cleanup for quite a long while :c


What next?
----------
Now our systems and components are finally out of `main.rs`. Next, we start taking steps towards unifying the way we access component storage, so that ultimately the dispatcher does not need to know what types of components the system actually wants, and can then *"just call `tick`"* without further complications on deciding which component vectors to provide.

The full source code can be found in branch `part-6` ([link](https://github.com/Kailari/kokonaisuus/tree/part-6))

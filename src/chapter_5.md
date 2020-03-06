The Lifetime of `'a` system
===========================
*"Implementing the basic concept of a system"*

### Topics
 - Lifetime annotations
 - Lifetime elision
 - *(bonus)* Upcoming language concept: GAT

What are we trying to do?
-------------------------
The systems in our current implementation have nothing in common. We are going to change that by making each system their own struct and implementing a `System` trait on them. This allows us to have a single unified `System::tick`-method we can call on any system.

This complicates parameter definition for the system tick function, however, and we are required to add *lifetime annotations* to few places to make the compiler understand that references to component storage vectors are alive for long enough.

Also, we are adding the first *"dumb"-implementations* for the dispatcher and the component storage by moving system ticking and component initialization and storage to their own modules, out of the `main`.

Spring cleaning, vol. 2
-----------------------
First, in `main.rs`, we create two new submodules:
```rust
mod component_storage;
mod dispatcher;
```

Then, today's goal is to replace the contents of our `pub fn main()` with
```rust
pub fn main() {
    let mut components = ComponentStorage::new();
    let dispatcher = Dispatcher::new();
    dispatcher.dispatch(&mut components);
}
```

Let's start with the component storage. In our new module `component_storage.rs`, we define a simple wrapper for our storage vectors:
```rust
pub struct ComponentStorage {
    pub positions: Vec<PositionComponent>,
    pub velocities: Vec<VelocityComponent>,
    pub accelerations: Vec<AccelerationComponent>,
    pub frictions: Vec<FrictionComponent>,
}
```

Then, for convenience, we add a `::new()` associated function for constructing the default instance. What we do here is just cut-paste the old component initialization code, and use that to initialize a `ComponentStorage`

```rust
impl ComponentStorage {
    pub fn new() -> ComponentStorage {
        ComponentStorage {
            positions: vec![
#                PositionComponent::new(0.0, 0.0),
#                PositionComponent::new(-42.0, -42.0),
#                PositionComponent::new(234.0, 123.0),
#                PositionComponent::new(6.0, 9.0),
            ],
            velocities: vec![
#                VelocityComponent::new(40.0, 10.0),
#                VelocityComponent::new(30.0, 20.0),
#                VelocityComponent::new(20.0, 30.0),
#                VelocityComponent::new(10.0, 40.0),
            ],
            frictions: vec![
#                FrictionComponent::new(1.0),
#                FrictionComponent::new(2.0),
#                FrictionComponent::new(3.0),
#                FrictionComponent::new(4.0),
            ],
            accelerations: vec![
#                AccelerationComponent::new(2.0, 16.0),
#                AccelerationComponent::new(4.0, 2.0),
#                AccelerationComponent::new(8.0, 4.0),
#                AccelerationComponent::new(16.0, 8.0),
            ],
        }
    }
}
```


What next?
----------
TODO

The full source code can be found in branch `part-6` ([link](https://github.com/Kailari/kokonaisuus/tree/part-6))

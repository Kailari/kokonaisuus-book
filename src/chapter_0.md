The Bare-bones
==============
*"Just the minimal something that gets the job done"*

### Topics
 - Structs
 - Vectors
 - Iterators
 - `enum Option<T>`
 - Shadowing
 - Inline pattern matching

### We start the project with
 - A main function
 - 2 components
 - a few iterators
 - a few `while let` loops
 
What are we trying to do?
-------------------------
In this part, we have simplified the complex task of creating an ECS into a small specific situation. The situation we are trying to solve is as follows:

*We have a collection of entities with a position and a velocity. We would like to translate the entities by their velocities, so that given an entity index \\(i\\)*

\\[
    pos_i = pos_i + vel_i
\\]

That is, we would like to have a collection of entities and move all of them with their assigned velocities.


Now, let's get started!
-----------------------
Ok, seems simple enough! So, we need some way of describing the velocities and the positions. Let's create *"components"* for those:

```rust
struct PositionComponent {
    x: f64,
    y: f64,
}

struct VelocityComponent {
    x: f64,
    y: f64,
}
```

KISS is my motto here *(Keep It Simple, Stupid!)*. Nothing fancy, no traits, no nothing, just plain good ol' structs. Both describe a property our "entities" will have.

Now, let's initialize our entities at the beginning of `fn main()` using these components. For our current implementation, as all our *Entities* are the same *(a position and a velocity)* it is conceptually enough to treat them just as list indices. So, we initialize four entities:
```rust
pub fn main() {
    let mut positions = vec![
        PositionComponent { x: 0.0, y: 0.0 },
        PositionComponent { x: -42.0, y: -42.0 },
        PositionComponent { x: 234.0, y: 123.0 },
        PositionComponent { x: 6.0, y: 9.0 },
    ];
    let velocities = vec![
        VelocityComponent { x: 40.0, y: 10.0 },
        VelocityComponent { x: 30.0, y: 20.0 },
        VelocityComponent { x: 20.0, y: 30.0 },
        VelocityComponent { x: 10.0, y: 40.0 },
    ];
}
```

Here we use the `vec!`-macro to initialize the component "storage vectors" using the array initializer syntax. Note that in our described situations, the velocities are constant and only positions change, thus the positions is defined with `let mut` and velocities with `let`. This means that `positions` is *mutable*, so that its contents are allowed to change and `velocities` is *immutable* so its contents are guaranteed to stay at their initial values *(This causes trying to change any velocity to trigger a compiler error)*. Its useful to differentiate between *mutability* of the data like this for multiple reasons, but we won't go in-depth with that quite yet.

The component initialization itself is done by struct construction syntax like this:
```rust
PositionComponent { x: 0.0, y: 0.0 }
```

Rust has no concept of constructors apart from the aforementioned syntax. Custom constructors are then just associated functions, which *just happen* to produce instances of the type. For instance, we could define *an associated constructor function* for the `PositionComponent` as follows:

```rust
impl PositionComponent {
    fn new(x: f64, y: f64) -> PositionComponent {
        // Names match so we don't need to write "{ x: x, y: y }"
        PositionComponent { x, y }
    }
}
``` 

However, in this case construction is quite trivial, so we avoid added complexity and use the default syntax instead. We'll add constructors later on, if needed.


### Mutating data

Now, we would like to apply the data mutation, or in more general terms, we'd like to apply the velocities to the positions. This is the first draft of a *System* or the *S* in our *ECS*. First things first, what we are trying to do *(pseudocode)*:
```
positions[0] += velocities[0];
positions[1] += velocities[1];
positions[2] += velocities[2];
positions[3] += velocities[3];
```

If we knew we would always have just four entities, this would be fine, but as this does not scale very well. We are required to do better than that. Let's write that again, but this time with Rust's *iterators*:
```rust
fn main() {
    // ...
    let mut pos_iter = positions.iter_mut();
    let mut vel_iter = positions.iter();

    loop {
        let pos = pos_iter.next().unwrap();
        let vel = vel_iter.next().unwrap();
        pos.x = pos.x + vel.x;
        pos.y = pos.y + vel.y;
    }
}
```

Okay, so here we create iterators for iterating over the storages, get next element from each iterator and apply the mutation. The `Iterator::next`-method returns an `Option<T>` which is either `Some(value)` or `None`. Call to `Option::unwrap` blindly assumes the value is indeed of the variant `Some` and returns the wrapped value. However, if the `Option` happened to be `None` the method panics and the program crashes.

On the other hand, the `Iterator::next` returns `None` when it has iterated over the whole collection and there are no more items to iterate over. ...which is exactly what will happen on fifth iteration of that loop, causing the `Option::unwrap` to panic and crash.

Darn, this doesn't quite work either.

We need to somehow break the execution of that loop as soon as either of the iterators returns `None`. For this purpose, we can use *inline pattern matching*, more specifically the `while let` -loop. Let's forget about the mutation part for a bit and look at how we could print the values out, as it is a simpler situation, where we only read positions.

```rust
fn main() {
    // Component initialization ...
    // Applying the velocities ...

    let mut pos_iter = positions.iter();

    while let Some(position) = pos_iter.next() {
        println!("Position: ({},{})", position.x, position.y)
    }
}
```

Here, we match the next item from the `pos_iter`, and have two possible outcomes based on what that item ends up being.
 1. The item is `Some(position)`, the loop executes and `pos_iter.next()` is called again or
 2. The item is `None`, the loop breaks as the condition is not met

Now, let's use this same pattern for applying the velocity:

```rust
fn main() {
    // Component initialization ...

    let mut pos_iter = positions.iter_mut();
    let mut vel_iter = positions.iter();

    while let (Some(pos), Some(vel)) = (pos_iter.next(), vel_iter.next()) {
        pos.x = pos.x + vel.x;
        pos.y = pos.y + vel.y;
    }

    // Printing ...
}
```

Note the extra parentheses around the expressions on each side of the equivalence operator (`=`). Here, we are constructing a tuple `let item_tuple = (pos_iter.next(), vel_iter.next())` and matching that against the pattern `(Some(pos), Some(vel))` *(which is also a tuple)*. So what happens, if both items in the tuple are `Some(value)`, we execute the loop with `pos` being the next value from `pos_iter` and `vel` being the next value from `vel_iter` and in all other cases *(either one or both of the iterators returned `None`)*, the pattern will fail to match and the loop will break safely. Neat! 

Note that we use the same name for the `pos_iter` in both cases. By doing this, we are actually *shadowing* the iterator `pos_iter` when we define it for the second time.

*"Shadowing"* is just hiding some old unused variable by creating a new one with the exact same name. Here, we create a new `pos_iter` for iterating over the positions. Note that the newly created is an *immutable iterator* *(does not allow mutation of the underlying data)*, whereas the original, now shadowed iterator was a *mutable iterator*. *(Type of shadowed variable need not be the same as the new variable)*


What next?
----------
Now, while this "works", there are a number of limitations here
 1. Iterators have to be initialized separately, creating a lot of clutter
 2. Even with while-let, the loop is mighty ugly and with more than two components, it could get unwieldy quite quick. On the other hand if we could use actual iterators, that would allow using `.filter()`, `.map()`, `.fold()`, etc. on the component collections. Is that useful? I have no clue, but that would be neat!
 3. Later down the line, when we want to parallelize things, raw vectors are not going to cut it anymore.

The full source code can be found in branch `part-0` ([link](https://github.com/Kailari/kokonaisuus/tree/part-0)).

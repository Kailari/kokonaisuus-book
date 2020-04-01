Arbitrary Component Types
==================
*"Component storage should not be concerned of the types of the components stored".*

### Topics
 - `HashMap`
 - `Box<T>`
 - `Box<dyn Trait>`
 - Match guards

What are we trying to do?
-------------------------
Hard-coding the component types to the component storage is not particularly great long-term approach. What we would like instead, is to be able to register arbitrary component types to our storage without having to worry type definitions or structs exploding out of control.

When we take a look at our current implementation of the component storage, it is quite evident that there is a common pattern among our component storage vectors.
```rust,noplaypen
components: RefCell<Vec<ComponentType>>
```

What we are trying to achieve is something more like
```rust,noplaypen
components: Map<TypeId, RefCell<Vec<ComponentType>>>
```

Last, we suffer from bit of a feature creep and end up re-writing a lot of stuff to achieve optional components on per-entity basis *(all entities no longer must neccessarily have all components)*.

### Whoops!
There are some interesting details about how Rust handles memory and how that affects the performance of our current component storage implementation. Currently, our components are all stored in vectors, which in turn are wrapped to a `RefCell`. The cell is there just to allow runtime borrow-checking and does not affect storage much. However, the vector part has some additional things I'd like to point out.

Note that for the rest of this chapter, I'm going to assume that you have at least a vague understanding of stack and the heap. For a good TL;DR on the topic, refer to relevant parts in the chapter *4.1. What is Ownership* in [the book](https://doc.rust-lang.org/stable/book/ch04-01-what-is-ownership.html#the-stack-and-the-heap).

Let's get this out of the way; for the longest time, I thought that `Vec` was stack-allocated, unless it is explicitly allocated on the heap. I've learnt that this is not the case. Elements of a `Vec` are indeed allocated on the heap.

This is not bad news though! I have been worried if the component storage could cause stack overflows with too many components, but due to the components actually residing on the heap, that is not an issue. Also, I've been avoiding boxing things to avoid doing costly heap allocations, but now that they are in there anyways, I'm not worried anymore about having to use boxes and trait objects. *(The allocations themselves only happen at initialization, when the storages are first created, thus mitigating some of the downsides of having to use heap)*.


Using `HashMap` for component types
-----------------------------------
Like many other languages, the Rust standard library comes with plethora of useful common collections. Here, we want to store a map of `TypeId`s to component storage vectors, so a standard `HashMap` is quite straightforward choice *(For starters, at least. Performace can be improved Later™)*.

There is a slight issue, though. Our storage vectors, have very distinct types. Each component vector needs to have known component type when it is first allocated. That is
```rust,noplaypen
// Previously, this was just fine:
struct ComponentStorage {
    as: RefCell<Vec<ComponentA>>,
    bs: RefCell<Vec<ComponentB>>
}

// Now:
struct ComponentStorage {
    components: HashMap<TypeId, RefCell<Vec<???>>>
    //            What can we specify here? ^^^
}
```

The first option that comes to mind is to use a trait object. Trait objects do not have known size at compile-time, so we need to *box* them *(non-sized types must be allocated on the heap)*. I love the way how Rust handles this; we just wrap the thing in a `Box<T>`. Here, we are lazy and just use `Any` trait to allow anything to be used as a component:
```rust,noplaypen
components: HashMap<TypeId, RefCell<Vec<Box<dyn Any>>>>
```

However, there is a slight issue to this. Or rather, there is a rather huge glaring issue to this. We are storing boxes in a vector, which by itself is not an offense of any sort, but in this context it becomes quite problematic.

Let's think a bit how this could look like in memory. We have a `Vec` which holds a pointer to a block of memory containing... Oh wait! It contains `Box<dyn Any>`, which means the vec contains *pointers* to the components, not the components themselves! Unless compiler does something *really* smart and optimizes this away, we have an additional layer of indirection here and our components might potentially be scattered all around in the memory! Uh oh, this is no good.

Let's think this for a bit; what are we trying to do? Trait objects are great for when we are trying to store something with varying sizes. Here, however, our vectors always contain entries of equal sizes, but the vectors themselves are of different sizes. Thus it makes no sense to pack the individual components to boxes. We would like to store vectors themselves as "trait objects" as the vectors are at the layer at which we have variable sizes!

This is where things get ugly; I have yet to find a "clean" approach to this. What we are trying to do is something like:
```rust,noplaypen,does_not_compile
// ERROR: dyn Any does not have known size at compile time!
components: HashMap<TypeId, RefCell<Box<Vec<dyn Any>>>>

// or

// ERROR: dyn Any does not have known size at compile time!
components: HashMap<TypeId, Box<RefCell<Vec<dyn Any>>>>
```
The problem here is that we do not have any way of telling the compiler what we are trying to do. I'm fairly confident this could be worked around with a custom map or vector type, but I *really* don't feel like spending time on figuring that out just yet.

*(I spent a bit more time than I'm willing to admit to trying to get the type signatures nicer but I could not figure out a safe way of doing this. I'll likely revisit this later when I figure something out)*

First, from the two options in the snippet above, the latter is the one we are actually going to use. Why? When the box is the outer wrapper type, we get to do downcasting *before* we have to borrow the underlying vector. In my experiments, this simplified the implementation *a lot*, but oh boy, the cost we have to pay. These more or less required compromises simplify our map's type signature down to:
```rust,noplaypen
components: HashMap<TypeId, Box<dyn Any>>
```

...which does not tell us *darn thing* about what the map contains. However, this gives us all the freedom in the world to cast the contents of that box to anything we'd like *(Although if we try to cast to anything else than the exact correct type, the program panics)*.

Now, with that in mind, let's add a constructor and a bit dumb version of component initialization. Our component storage becomes:
```rust,noplaypen
pub struct ComponentStorage {
    components: HashMap<TypeId, Box<dyn Any>>,
}

impl ComponentStorage {
    pub fn new() -> ComponentStorage {
        ComponentStorage { components: HashMap::new() }
    }

    pub fn register_component_type<C>(&mut self, components: Vec<C>)
        where C: Any
    {
        self.components.insert(TypeId::of::<C>(), Box::new(RefCell::new(components)));
    }

    // ... (fetching etc.)
}
```

We use `HashMap::new` to allocate a new map in the constructor and the `HashMap::insert` to put items to it. The keys are unsurprisingly generated with `TypeId::of`. Additional thing to note is that we do the boxing and wrapping to a `RefCell` here, too.

From here on, fetching the component storage is quite easy to implement. The `HashMap` provides `get`-method for fetching `Option<T>`. The `fetch_component_storage` then becomes:
```rust,noplaypen
fn fetch_component_storage<C>(&self) -> &RefCell<Vec<C>>
    where C: Any
{
    let component_type_id = TypeId::of::<C>();
    let storage = self.components.get(&component_type_id)
                      .unwrap_or_else(|| panic!("Unregistered component type: {}",
                                                std::any::type_name::<C>()));

    storage.downcast_ref::<RefCell<Vec<C>>>()
           .expect("Downcasting storage RefCell failed!")
}
```

Wait a minute! The only actual change is how we fetch the `storage`!? How can this be, we just added additional layer of indirection, the `Box` around our storage vectors?

The trick is that the `downcast_ref` we are calling here is actually different from what we were calling previously. If you inspect the code carefully, the type of `storage` has actually changed. It used to be `&dyn Any`, but now it is `Box<dyn Any>`.

The thing is, both of these have their own implementations of `downcast_ref`, doing a very similar thing, but in case of the box, hiding the additional indirection involved. In practice, the cost of this *"hidden indirection"* here is negligible, as it is present only once when storages are being fetched. Therefore, the way how `Box<dyn Any>` essentially allows us to cast pointers to other types is just very convenient here.


### Is this safe? 
Being able to downcast pointers is convenient. However, getting the casts right the first time was not before a lot of trial-and-error. There is no type safety here; should the storage types ever change or other types of storages be allowed, there is no guarantee these downcasts continue being valid. Most importantly, the compiler does not complain if anything is wrong.

That being said, only place we write to the map is in the component type registration. As long as we make sure the type signatures are correct and correspond to the `TypeId`s there, the downcasts in `fetch_component_storage` are guaranteed to be valid. The type tokens make sure of that.

So, in my personal opinion, as internal implementation detail, this is fine. Yes, it is mighty ugly and smells worse than what I do after a week long hike, but it works and there should not be that much reasons for fiddling with registration and fetching in ways that could break it in the near future. I do not love it, but I'll leave it at that for now.


Registering component types
---------------------------
Only thing left to do is to actually register the component types. Our `main` becomes:
```rust,noplaypen
pub fn main() {
    let mut storage = ComponentStorage::new();
    let positions = vec![
        // ...
#         PositionComponent::new(0.0, 0.0),
#         PositionComponent::new(-42.0, -42.0),
#         PositionComponent::new(234.0, 123.0),
#         PositionComponent::new(6.0, 9.0),
    ];
    let velocities = vec![
        // ...
#         VelocityComponent::new(40.0, 10.0),
#         VelocityComponent::new(30.0, 20.0),
#         VelocityComponent::new(20.0, 30.0),
#         VelocityComponent::new(10.0, 40.0),
    ];
    let frictions = vec![
        // ...
#         FrictionComponent::new(1.0),
#         FrictionComponent::new(2.0),
#         FrictionComponent::new(3.0),
#         FrictionComponent::new(4.0),
    ];
    let accelerations = vec![
        // ...
#         AccelerationComponent::new(2.0, 16.0),
#         AccelerationComponent::new(4.0, 2.0),
#         AccelerationComponent::new(8.0, 4.0),
#         AccelerationComponent::new(16.0, 8.0),
    ];

    storage.register_component_type(positions);
    storage.register_component_type(velocities);
    storage.register_component_type(accelerations);
    storage.register_component_type(frictions);

    let dispatcher = Dispatcher::new();
    dispatcher.dispatch(&mut storage);
}
```


(Arguably) Nicer component intialization
----------------------------------------
That was the first draft. Now that we can register any arbitrary component type we'd like it becomes quite evident that having to do it for each and every *"entity"* is quite a lot of work! Also, I do not like the way how `main` actually allocates the underlying memory when it initializes the component storage vectors as this strongly couples the component storage memory layout with our `main` method.

So, next we are going to move those vectors inside the component storage and almost by accident, we are getting closer to having actual concrete representation of *"entities"* in our ECS.

Plan is to allow initializing the component storage with capacity of `n` components per type. Effectively this means that the storage is for `n` entities.

For this, we add another field for the `ComponentStorage` struct:
```rust,noplaypen
struct ComponentStorage {
#     components: HashMap<TypeId, Box<dyn Any>>,
    // ...
    capacity: usize,
}

impl ComponentStorage {
    pub fn new(capacity: usize) -> ComponentStorage {
        ComponentStorage {
            capacity,
            components: HashMap::new()
        }
    }
    // ...
}
```

Next, let's change the signature of `register_component_type` a bit:
```rust,noplaypen
pub fn register_component_type<C>(&mut self, default: C)
        where C: Any
```

Instead of getting the component vector as parameter, we prepare to allocate it in the storage. For this, we need a default component to be used as the default value for entities which we do not otherwise specify the component values.

The implementation is straightforward, although I'm bad with Rust's vectors so this is a bit messy:
```rust,noplaypen
pub fn register_component_type<C>(&mut self, default: C)
    where C: Any + Copy
{
    let mut storage_vector = Vec::<C>::with_capacity(self.capacity);
    storage_vector.resize_with(self.capacity, || default);
    self.components.insert(TypeId::of::<C>(), Box::new(RefCell::new(storage_vector)));
}
```

There really isn't that much to it. We initialize a new component vector with a capacity and use `resize_with` to fill it with the default component. Then, use the old one-liner to box it, wrap it to a ref cell and then insert it with appropriate key to the map.

There is also an additional `Copy` trait bound on the component types. This is required to be able to copy the default value to all vector cells. This bound then requires that we add `#[derive(Copy,Clone)]` to all our component types. I'm not sure if having components `Copy` is a good idea or not *(Currently I do not have them copyable to avoid accidental copies)*, but for this step it is required. *(Just as a heads up, I have later on removed the derives after I implemented optional components)*

Component type registration in `main.rs` then becomes:
```rust,noplaypen
let mut storage = ComponentStorage::new(4);
storage.register_component_type(PositionComponent::new(0.0, 0.0));
storage.register_component_type(VelocityComponent::new(0.0, 0.0));
storage.register_component_type(AccelerationComponent::new(0.0, 0.0));
storage.register_component_type(FrictionComponent::new(0.0));
```

You might have noticed that we now have no way of adjusting component properties without systems. Let's change that; in component storage, we add:
```rust,noplaypen
pub fn add_to_entity<C>(&self, entity_id: usize, component: C)
    where C: Any
{
    let mut storage = self.fetch_component_storage::<C>()
                          .borrow_mut();
    storage[entity_id] = component;
}
```

Here, we use `fetch_component_storage` to fetch the storage cell and then just blindly change the component for the given entity id. This is the first time we have actual concrete entity IDs in use, neat!

Now, we can add initial values for our components:
```rust,noplaypen
pub fn main() {
#     let mut storage = ComponentStorage::new(4);
#     storage.register_component_type(PositionComponent::new(0.0, 0.0));
#     storage.register_component_type(VelocityComponent::new(0.0, 0.0));
#     storage.register_component_type(AccelerationComponent::new(0.0, 0.0));
#     storage.register_component_type(FrictionComponent::new(0.0));
    // ...

    storage.add_to_entity(0, PositionComponent::new(0.0, 0.0));
    storage.add_to_entity(1, PositionComponent::new(-42.0, -42.0));
    storage.add_to_entity(2, PositionComponent::new(234.0, 123.0));
    storage.add_to_entity(3, PositionComponent::new(6.0, 9.0));

    // ...
# 
#     storage.add_to_entity(0, VelocityComponent::new(40.0, 10.0));
#     storage.add_to_entity(1, VelocityComponent::new(30.0, 20.0));
#     storage.add_to_entity(2, VelocityComponent::new(20.0, 30.0));
#     storage.add_to_entity(3, VelocityComponent::new(10.0, 40.0));
# 
#     storage.add_to_entity(0, FrictionComponent::new(1.0));
#     storage.add_to_entity(1, FrictionComponent::new(2.0));
#     storage.add_to_entity(2, FrictionComponent::new(3.0));
#     storage.add_to_entity(3, FrictionComponent::new(4.0));
# 
#     storage.add_to_entity(0, AccelerationComponent::new(2.0, 16.0));
#     storage.add_to_entity(1, AccelerationComponent::new(4.0, 2.0));
#     storage.add_to_entity(2, AccelerationComponent::new(8.0, 4.0));
#     storage.add_to_entity(3, AccelerationComponent::new(16.0, 8.0));
# 
#     let dispatcher = Dispatcher::new();
#     dispatcher.dispatch(&mut storage);
}
```

This by itself did not achieve very much for us. The most important part is that the `ComponentStorage` is now actually completely in control of how the components are stored. This allows us to implement the next feature.


Optional components
-------------------
Issue with our current storage model is that all entities currently have all components. Next we are going to change our storage vectors from
```rust,noplaypen
storage: Box<RefCell<Vec<ComponentType>>>
```
to
```rust,noplaypen
storage: Box<RefCell<Vec<Option<ComponentType>>>>
```

Ok. Right. Does not look too hard, huh? Just add the optionals and figure out how to iterate the options, *can't be that hard*.

Heads up: we are going to more or less rewrite everything so brace yourselves.

### Quick cleanups
I finally got fed up with passing raw `Ref`s and `RefMut`s around. Also defining the read/write requirements of a system with raw references was a bit ugly. I wrote wrappers for those references, like this:
```rust,noplaypen
pub struct Write<'a, C> {
    storage: RefMut<'a, Vec<C>>,
}

pub struct Read<'a, C> {
    storage: Ref<'a, Vec<C>>,
}
```

I then added `From<T>` implementations for them to convert the `RefCell` reference wrappers to these our own wrapper-wrappers:
```rust,noplaypen
impl<'a, C> From<RefMut<'a, Vec<C>>> for Write<'a, C> {
    fn from(source: RefMut<'a, Vec<C>>) -> Self {
        Write { storage: source }
    }
}

impl<'a, C> From<Ref<'a, Vec<C>>> for Read<'a, C> {
    fn from(source: Ref<'a, Vec<C>>) -> Self {
        Read { storage: source }
    }
}
```

Now, we get to the good bits, I added a methods to convert the darn things into iterators:
```rust,noplaypen
impl<'a, C> Write<'a, C> {
    pub fn iterate(&mut self) -> IterMut<C> {
        self.storage.iter_mut()
    }
}

impl<'a, C> Read<'a, C> {
    pub fn iterate(&self) -> Iter<C> {
        self.storage.iter()
    }
}
```

Then, I changed `fetch_ref` and `fetch_mut` to return these
```rust,noplaypen
pub fn fetch_mut<C>(&self) -> Write<C>
    where C: Any
{
    Write::from(self.fetch_component_storage::<C>().borrow_mut())
}

pub fn fetch_ref<C>(&self) -> Read<C>
    where C: Any
{
    Read::from(self.fetch_component_storage::<C>().borrow())
}
```

After which we need to tweak system `InputData` and dispatcher a bit. For example, for `ApplyAccelerationSystem` the changes are:
```rust,noplaypen
// Systems:
type InputData = (Write<'a, VelocityComponent>,
                  Read<'a, AccelerationComponent>);

fn tick(&self, (mut velocities, accelerations): Self::InputData) {
    let vel_iter = velocities.iterate();
    let acc_iter = accelerations.iterate();

    // ...
}

// Dispatcher:
self.apply_acceleration.tick((
    storage.fetch_mut::<VelocityComponent>(), // removed `as_mut`
    storage.fetch_ref::<AccelerationComponent>(), // removed `as_ref`
));
```

With these done, the component storage implementation details are now hidden from dispatcher and the systems, behind an additional layer of abstraction. We also got rid of the ugly `as_mut` and `as_ref` calls, which is neat! This should come in handy later on if we want to do something more special with our system inputs.

### Rewriting everything
Now with system inputs cleaned up, the first step to getting data as optionals is changing the storage type in the first place. This step is actually quite simple, as we create all our storage vectors in `register_component_type`:
```rust,noplaypen
pub fn register_component_type<C>(&mut self)
    where C: Any
{
    let mut storage_vector = Vec::<Option<C>>::with_capacity(self.capacity);
    storage_vector.resize_with(self.capacity, || None);
    self.components.insert(TypeId::of::<C>(), Box::new(RefCell::new(storage_vector)));
}
```
We've got rid of the default parameter and the `Copy` trait bound on component type. Also, the component type in vector initialization is now wrapped to `Option` and the default component is substituted with `Option::None`. Apart from those, the initialization remains surprisingly untouched.

Now that we have the logic in place so that there is concept of entities either having or not having a component, I would like add an additional layer of security, so that we cannot add component to an entity which already has one. The `add_to_entity` method has for now just blindly overridden the previous component, not caring if it was the default component or not. Let's change this a bit.

We get the storage, just like before, with the `fetch_component_storage` and borrow the cell. Vectors provide a `fn get(&self, index) -> Option<T>` which we can use to try to fetch components with the given entity ID. As we initialize the component vectors with initial capacity and all elements as `None`, we should be fine as long as we pass in entity IDs which are `entity_id < capacity`.

Note that the type of the vector is now `Vec<Option<C>>`. Due to that, the aforementioned validation now requires a bit more complex pattern matching as `get` actually returns a `Option<Option<C>>`. Luckily, pattern matching in Rust is quite powerful and we can just write
```rust,noplaypen
pub fn add_to_entity<C>(&self, entity_id: usize, component: C)
    where C: Any
{
    let mut storage = self.fetch_component_storage::<C>().borrow_mut();

    // Nested enums in pattern matching, yay!
    if let Some(Some(_)) = storage.get(entity_id) {
        panic!("Tried adding component {} to entity_id={}, but it already has one!",
                std::any::type_name::<C>(),
                entity_id);
    }

    // ...
```

We are only interested in the fact that the entity should *NOT* have a component when we are adding one, so the actual component is substituted with `_` in the pattern as we are not going to use it for anything.

Then, lastly, when adding the component, we need to wrap it to `Option::Some`
```rust,noplaypen
    // ...

    storage[entity_id] = Some(component);
}
```

Now everything is broken, first of all, `fetch_component_storage` still returns non-option vectors. Let's fix that next.

Only actual changes required are the return type and the type used in the downcast:
```rust,noplaypen
// Method signature:
fn fetch_component_storage<C>(&self) -> &RefCell<Vec<Option<C>>>
    where C: Any
{
    // ...

    // Downcast at the end:
    storage.downcast_ref::<RefCell<Vec<Option<C>>>>().unwrap()
}
```

After this, `add_to_entity` no longer has gazillion errors. However, we have now broken `fetch_ref` and `fetch_mut` as our `Read` and `Write` wrappers do not understand that the components should be options. Let's look into those next. There are quite a few places where we need to change the types to use options instead of raw components:

First, the structs themselves:
```rust,noplaypen
pub struct Write<'a, C> {
    storage: RefMut<'a, Vec<Option<C>>>,
}

pub struct Read<'a, C> {
    storage: Ref<'a, Vec<Option<C>>>,
}
```

Second, the `From` implementations:
```rust,noplaypen
impl<'a, C> From<RefMut<'a, Vec<Option<C>>>> for Write<'a, C> {
    fn from(source: RefMut<'a, Vec<Option<C>>>) -> Self {
        Write { storage: source }
    }
}

impl<'a, C> From<Ref<'a, Vec<Option<C>>>> for Read<'a, C> {
    fn from(source: Ref<'a, Vec<Option<C>>>) -> Self {
        Read { storage: source }
    }
}
```

And third, the `iterate` methods:
```rust,noplaypen
impl<'a, C> Write<'a, C> {
    pub fn iterate(&mut self) -> IterMut<Option<C>> {
        self.storage.iter_mut()
    }
}

impl<'a, C> Read<'a, C> {
    pub fn iterate(&self) -> Iter<Option<C>> {
        self.storage.iter()
    }
}
```

With this done, we still need to change our component registration a bit before we are done with the storage changes:
```rust,noplaypen
storage.register_component_type::<PositionComponent>();
storage.register_component_type::<VelocityComponent>();
storage.register_component_type::<FrictionComponent>();
storage.register_component_type::<AccelerationComponent>();
```

Whew! The component storage should be done now. However, all our systems are now broken as they are relying on `Read` and `Write` on unwrapping to `Vec<C>` and not `Vec<Option<C>>`.  Fixing this is going to be a bit more tasking.


### Matchcasesplosion

For now, iterating entities in systems has been nice and simple as all entities have had all of the component types. Due to this, all systems have been able to just iterate over all entities. The iterators themselves had two distinct states:
```rust,noplaypen
let item = iter.next();

match item {
    Some(_) => println!("The iterator has items"),
    None => println!("The iterator has been exhausted"),
}
```

However, now that we have optional components, the situation has got a bit more complex. The `iter.next()` now actually returns a `Option<Option<C>>` so matching that becomes slightly cumbersome:
```rust,noplaypen
let item = iter.next();

match item {
    Some(Some(_)) => println!("The iterator has items!"),
    Some(None) => println!("The current entity does not have the component!"),
    None => println!("The iterator has been exhausted!"),
}
```

When we add in another components, it just gets worse:
```rust,noplaypen
let a = iter_a.next();
let b = iter_b.next();

match (a, b) {
    (Some(Some(a)), Some(Some(b))) => println!("Both iterator have items!"),
    (None, None) => println!("Both iterators have been exhausted!"),
    (None, _) => println!("The iterator A has been exhausted!"),
    (_, None) => println!("The iterator B has been exhausted!"),
    (Some(None), Some(None)) => println!("The current entity does not have either of the components!"),
    (Some(None), _) => println!("The current entity does not have the component A!"),
    (_, Some(None)) => println!("The current entity does not have the component B!"),
}
```

So, as this seems to be quickly exploding out of control, we need to identify some common patterns here. One such case is the simple case where all desired components are present. Let's start creating an enum for these:
```rust,noplaypen
// Here `T` is the type of the tuple with all components from iterators
enum IteratorTupleOption<T> {
    /// All iterators in the tuple returned a `Some(Some(value))`
    All(T),
}
```

This handles the first case. Now we still need to handle the following cases:
```rust,noplaypen
(None, None) => ..,             // 1.
(None, _) => ..,                // 2.
(_, None) => ..,                // 3.
(Some(None), Some(None) => ..,  // 4.
(Some(None), _) => ..,          // 5.
(_, Some(None)) => ..,          // 6.
```

Cases 1. to 3. describe situations after which we know for sure that the `All` case is impossible to reach, as one or more iterators have been exhausted. This gives us another enum variant:
```rust,noplaypen
enum IteratorTupleOption<T> {
#     /// All iterators in the tuple returned a `Some(Some(value))`
#     All(T),
    /// One or more iterators in the tuple returned a `None`
    None,
}
```

This leaves us with cases 4. to 6. which actually form a single group. They are all *"partial"* matches, where the entity in question did not have all required components. This can be handled with a third variant.

Our enum for handling these cases is then
```rust,noplaypen
enum IteratorTupleOption<T> {
    /// All iterators in the tuple returned a `Some(Some(value))`
    All(T),
    /// One or more iterators in the tuple returned a `None`
    None,
    /// All iterators in the tuple returned a `Some(Option)`,
    /// but one or more iterators produced a `Some(None)`.
    Partial,
}
```


### Iterating tuple of `Iter(Mut)<Option<T>>` as if they were `Iter(Mut)<T>`

The next issue is that we want to avoid adding more complexity to our systems. The tuple iterator should handle the aforementioned partial cases behind-the-scenes so that the systems do not get any partial entities in the first place. This requires figuring out how our `Option<T>`-spewing iterators could nicely be made output `T` directly.

In the end, the idea is again quite simple once you wrap your head around it. As we figured out, there are only three different cases: *All*, *None* and *Partial*. In case of *All*, we should output a tuple with the components. *None* means that we need to output, well, *None*. The last one, *Partial* is a bit trickier as we actually need to *skip the current element* and keep polling new ones until either All or None is reached.

As Rust-like pseudocode, this could then be written as something like:
```rust,noplaypen
fn next() -> IteratorTupleOption<ItemTuple> {
    loop {
        let next_tuple = (iter_a.next(), iter_b.next(), ...);
        match next_tuple {
            (Some(Some(a)), Some(Some(b)), ...) => return All((a, b)),
            (None, None, ...) => return None,
            _ => Partial,
        }
    }
}
```

Here, I took a shortcut to make writing this *(and macros later on)* easier. We might at some point be in a situation where our component vectors have different sizes, but for now, they all have equal lengths, giving us guarantee that *if any of the iterators produces `None`, all of them should produce `None`* *(Cases 1. to 3.)*. Due to that, I'm going to handle these as a single case for now.

Also, leaving the partial cases as the last one makes handling it dead-simple. We know that all other cases are already being handled by the other matcher arms, so we can use the wildcard pattern to match the partial cases *(as they are the only remaining ones)*.

Now there's a single oddity about this pseudocode; the `Partial` is actually never returned as we want to simply skip to the next `loop` iteration in those cases. This is Ok for the pseudocode, but for our implementation, we are going to move the actual loop elsewhere.

Now, let's take a look at our `ApplyVelocitySystem` and more precisely, its `tick` method:
```rust,noplaypen
let pos_iter = positions.iterate();
let vel_iter = velocities.iterate();

for (pos, vel) in IterTuple::from((pos_iter, vel_iter)) {
    pos.value += vel.value;
}
```
Now, where should we start applying our modifications? We see that `pos_iter` and `vel_iter` are of type `Iter(Mut)<Option<C>>`, but there is not much we can do in case of single iterators as they know nothing about each others' state. Therefore, the only place we know the state of all iterators at once is the `IterTuple`.

First change, `IteratorTuple` should return `IteratorTupleOption` instead of `Option`
```rust,noplaypen
pub trait IteratorTuple {
    type ItemTuple;

    fn next_all(&mut self) -> IteratorTupleOption<Self::ItemTuple>;
}
```

*Try not to worry about the gazillion errors due to our macro breaking from that change, we'll tackle that later on.*

Now, in the `impl<T> Iterator for IterTuple<T>` we need to convert the `IteratorTupleOption` returned by `next_all` into a `Option<ItemTuple>`. We are going to use parts of our pseudocode loop for this:
```rust,noplaypen
impl<T> Iterator for IterTuple<T>
    where T: IteratorTuple
{
    type Item = T::ItemTuple;

    fn next(&mut self) -> Option<Self::Item> {
        loop {
            match self.iterators.next_all() {
                IteratorTupleOption::All(values) => return Some(values),
                IteratorTupleOption::None => return None,
                IteratorTupleOption::Partial => ()
            }
        }
    }
}
```
This is just the loop from the pseudocode, but the complexity of constructing the tuple option has been abstrated to `IteratorTuple` *(which we still need to implement later on before this actually starts working)*.

Digging bit deeper into this, we are using the forever-loop as the iterators should under no circumstances be infinite, so at some point we will hit the `(None, None, ..)` case, which produces `IteratorTupleOption::None` from the `next_all`, returning `None` from the loop. Another option is that we get an actual match and return the matching component tuple. In cases where there is a partial match, we simply output *an empty tuple* *(The strange `()` in the partial matcher arm)* from the match, but **we do not use return in this case**, thus we stay in the loop and effectively skip to the next entity.

### Implementing the `IteratorTuple`
The `IteratorTuple` implementation proved to be a tough one and I had to resort to some interesting trickery to get things done. One limitation I met was that the standard library does not seem to provide any `trait` that could be used to describe something that can be unwrapped into something else. That is, there is no such trait that we could add bound on `Iterator::Item` that it must be `Option<C>` which could then be matched against `Some(component)`.

If that did not make any sense, here's some almost-valid code for 2-tuple implementation:
```rust,noplaypen
impl<A, B> IteratorTuple for (A, B)
    where A: Iterator,
          B: Iterator
{
    type ItemTuple = (/* A::Item unwrapped */, /* B::Item unwrapped */);
    // The problem    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    fn next_all(&mut self) -> IteratorTupleOption<Self::ItemTuple> {
        let option_tuple = (self.0.next(), self.1.next());
        match option_tuple {
            (Some(Some(a)), Some(Some(b))) => IteratorTupleOption::All((a, b)),
            (None, None) => IteratorTupleOption::None,
            _ => IteratorTupleOption::Partial,
        }
    }
}
```
Idea is to match next items from the iterators against the three known case groups and construct the `IteratorTupleOption` based on that. The big problem, however, is how we get the type system to understand what we are trying to do.

As I mentioned, the `Option<T>` *(which is what the iterators return)* does not implement out-of-the-box any traits that we could use to get the *"wrapped" type* for constructing the item tuple type. Particularly tricky is getting the returned mutable/immutable references match the iterators.

My workaround was to ditch some of the nested pattern matching and implement a custom trait on component `Option`s. Then, some syntatically horrifying trickery involving match-guards is needed to make things actually work.

First, let's write the above non-compiling snippet again with *match guards*.
```rust,noplaypen
impl<A, B> IteratorTuple for (A, B)
    where A: Iterator,
          B: Iterator
{
    type ItemTuple = (/* A::Item unwrapped */, /* B::Item unwrapped */);

    fn next_all(&mut self) -> IteratorTupleOption<Self::ItemTuple> {
        let option_tuple = (self.0.next(), self.1.next());
        match option_tuple {
            (Some(a), Some(b)) if a.is_some() && b.is_some() => All((a.unwrap(), b.unwrap())),
            (None, None) => None,
            _ => Partial,
        }
    }
}
```
We still do not know the types for the items, but here is an example of match guards in action. What they essentially do is just that they add an additional layer of filtering on our matcher pattern. That is, if we were to do that if-expression on the right hand side of the matcher arm *(inside the executed block)*, in case of the condition evaluating to `false`, **we would not hit the other matcher arms**. With guards, in case the condition is false, the pattern is not accepted and next arms are matched.

That being said, the match expression is equivalent to the previous one where we just used patterns. The key difference, though, is that we are now using method calls and method calls are something we have much more access to from traits!

Let's create a new trait, I'll explain the purpose of this thing as we go:
```rust,noplaypen
pub trait OptionLike {
    type Item;

    fn unwrap(self) -> Self::Item;

    fn is_some(&self) -> bool;
}
```

Anything implementing `OptionLike` is something that can be *"unwrapped"* into another type. Additionally, it provides `is_some` to check whether or not the wrapped value is present.

This now actually allows us to define the `type ItemTuple`!
```rust,noplaypen
impl<A, B> IteratorTuple for (A, B)
    where A: Iterator,
          B: Iterator,
          A::Item: OptionLike,
          B::Item: OptionLike,
{
    type ItemTuple = (<A::Item as OptionLike>::Item, <B::Item as OptionLike>::Item);

    fn next_all(&mut self) -> IteratorTupleOption<Self::ItemTuple> {
        let option_tuple = (self.0.next(), self.1.next());
        match option_tuple {
            (Some(a), Some(b)) if a.is_some() && b.is_some() => IteratorTupleOption::All((a.unwrap(), b.unwrap())),
            (None, None) => IteratorTupleOption::None,
            _ => IteratorTupleOption::Partial,
        }
    }
}
```
We impose additional bounds on the iterators in the `IteratorTuple`, the items must be *unwrappable*. Then, we use only a single layer of pattern matching, with guards for checking if the results are `Some(T)` or `None` *(the `has_value` calls in guards)*. Finally, we need to unwrap the `OptionLike`s by calling `x.unwrap()`.

...yeah, a bit messy, but does the trick and contains no arcane black-magic trickery! Just plain ol' trait bounds and traits.

Trying to run now results in bunch of errors from `Option<T>`s not implementing the `OptionLike`. Also, the macros are still broken.


### Implementing `OptionLike`
As we are not creating new functionality, this is quite straightforward. The only gotcha is the fact that iterators produce *references* to options, not concrete options themselves. This requires us to write a few additional implementations.

There are total of three implementations. First one, a bit of extra for sake of completeness, the concrete `Option`:
```rust,noplaypen
impl<T> OptionLike for Option<T> {
    type Item = T;

    fn unwrap(self) -> Self::Item {
        self.unwrap()
    }

    fn is_some(&self) -> bool {
        self.is_some()
    }
}
```

Another one, for immutable references:
```rust,noplaypen
impl<'a, T> OptionLike for &'a Option<T> {
    type Item = &'a T;

    fn unwrap(self) -> Self::Item {
        self.as_ref().unwrap()
    }

    fn is_some(&self) -> bool {
        Option::<T>::is_some(self)
    }
}
```

And last, but not least, very similar one for mutable references:
```rust,noplaypen
impl<'a, T> OptionLike for &'a mut Option<T> {
    type Item = &'a mut T;

    fn unwrap(self) -> Self::Item {
        self.as_mut().unwrap()
    }

    fn is_some(&self) -> bool {
        Option::<T>::is_some(self)
    }
}
```

And that's it actually. This is one of the reasons I love traits; Not only can we add new traits to existing types, but we also can add new traits that describe existing behavior! Here we added in a new trait to overcome the limitation of the standard library not providing us with all required traits.

### Rewriting macros
This is again just basically expanding our 2-tuple implementation with macro variadics.

After `where`, we need to add anohter expansion for adding trait bounds for `OptionLike`. The full `impl`-block signature then is
```rust,noplaypen
impl<$( $type_name ),+> IteratorTuple for ($( $type_name ),+)
    where $( $type_name: Iterator, )+
          $( $type_name::Item: OptionLike, )+
{
    // ...
```
Note that I've also moved the commas inside the repeat pattern to ensure that even the last one is postfixed with it. Providing the comma as separator *(the syntax is `$( pattern ) sep repeat`, where sep is separator and repeat is either `+` or `*`)* causes the last one to be ommited, which then causes there to be no comma between the `Iterator` bounds and the `OptionLike` bounds. This is not what we want, so just move the commas to be part of the pattern.

With 3-tuples, this expands to something like
```rust,noplaypen
impl<A, B, C> IteratorTuple for (A, B, C)
    where A: Iterator, B: Iterator, C: Iterator,
          A::Item: OptionLike, B::Item: OptionLike, C::Item: OptionLike,
{
    // ...
```

Then, we need to construct the `ItemTuple`. `T::Item` is no longer enough as the items are options. Therefore, we need to use fully qualified syntax.
```rust,noplaypen
type ItemTuple = ($( <$type_name::Item as OptionLike>::Item ),+);
```
which expands to
```rust,noplaypen
type ItemTuple = (<A::Item as OptionLike>::Item,
                  <B::Item as OptionLike>::Item,
                  <C::Item as OptionLike>::Item);
```

Then, the `next_all` method is quite messy as the first matcher arm requires three expansions on a single line. Ignore the `replace_expr!` macro for now, we'll get to that, eventually.
```rust,noplaypen
fn next_all(&mut self) -> IteratorTupleOption<Self::ItemTuple> {
    match ($( self.$i.next() ),+) {
        // The All case
        ($( Some($item_name) ),+)
        if $( $item_name.is_some() )&&+
        => IteratorTupleOption::All(($( $item_name.unwrap() ),+)),

        // None cases
        ($( replace_expr!( ($i) None ) ),+) => IteratorTupleOption::None,

        // Partial cases
        _ => IteratorTupleOption::Partial,
    }
}
```
The `All` case is split on three lines here. First line just expands the pattern to unwrap the first layer of `Option`s away. The second line matches the second layer of options, by expandign to the match guards. Note the use of `&&` as pattern expansion separator here. The third and last line just unwraps the `OptionLike`s and constructs the `IteratorTupleOption::All` from them.

That is, the first matcher arm expands to
```rust,noplaypen
(Some(a), Some(b), Some(c))
if a.is_some() && b.is_some() && c.is_some()
=> IteratorTupleOption::All((a.unwrap(), b.unwrap(), c.unwrap())),
```

For implementing the second matcher arm, we need a bit of trickery. We want the pattern to repeat `n` times, or in this case as many times as there are parameters, but we do not care about parameter contents.

We create a `replace_expr!`-macro for replacing any other expression with another expression, in this case, we want to substitute one of our parameters with `None`.

The macro looks like this:
```rust,noplaypen
/// Replaces a repetition sequence contents with some other expression.
macro_rules! replace_expr {
    ($_t:tt $sub:tt) => {$sub};
}
```

Now, the matcher arm expands simply to
```rust,noplaypen
(None, None, None) => IteratorTupleOption::None,
```

As a final touch, I moved most commas inside the repeat patterns, so that the macro works even for 1-tuples. *(1-tuple can be forced by adding a comma, as in `(x,)`)*

The final, full thing is then
```rust,noplaypen
macro_rules! replace_expr {
    ($_t:tt $sub:tt) => {$sub};
}

macro_rules! implement_iterator_tuple {
    ($( ($i:tt, $item_name:ident, $type_name:ident) ),+) => {
        impl<$( $type_name, )+> IteratorTuple for ($( $type_name, )+)
            where $( $type_name: Iterator, )+
                  $( $type_name::Item: OptionLike, )+
        {
            type ItemTuple = ($( <$type_name::Item as OptionLike>::Item, )+);

            fn next_all(&mut self) -> IteratorTupleOption<Self::ItemTuple> {
                match ($( self.$i.next(), )+) {
                    ($( Some($item_name), )+) if $( $item_name.is_some() )&&+
                    => IteratorTupleOption::All(($( $item_name.unwrap(), )+)),
                    
                    ($( replace_expr!( ($i) None ), )+) => IteratorTupleOption::None,

                    _ => IteratorTupleOption::Partial,
                }
            }
        }
    };
}
```

Now, the `PrintPositionsSystem` is broken, but as we added support for 1-tuples, it can be unified to use the `TupleIter` like the other systems:

```rust,noplaypen
impl<'a> System<'a> for PrintPositionsSystem {
    type InputData = Read<'a, PositionComponent>;

    fn tick(&self, positions: Self::InputData) {
        let pos_iter =  positions.iterate();
        for (pos,) in IterTuple::from((pos_iter,)) {
            println!("Position: {}", pos)
        }
    }
}
```

And finally, we are done. Everything should compile and run now just fine!


Other notes
-----------
Happy news! The associated constants feature has been stabilized in *1.43.0*, so we can remove its feature attribute from the `main.rs` *(Still need to use nightly though. As of writing 1.42.0 is the newest version on the stable toolchain)*.

I also removed the `entity_index` counter from the `PrintStateSystem` by using `Iterator::enumerate`, but that was quite trivial change. The entity IDs there are incorrect, however as they do not accommodate for skipped entities. Getting access to entities and their IDs from systems is something I need to address very soon.

What next?
----------
Whew! That was quite a lot of changes. Things are shaping up quite nicely, and I'm surprised that I actually got the optional components working. I'm particularly fond of the `OptionLike` workaround we implemented as it really highlights the awesomeness of traits.

Next up is getting entity IDs available to systems and unifying the system tick so that dispatcher could be given the same treatment as with component storage. That is, I would like to be able to register arbitrary system types to the dispatcher.

I have some ideas on how latter could be implemented. The idea is to *kindly ask the systems what resources they want* instead of dispatcher "deciding" the resources for them.

The full source code can be found in branch `part-8` ([link](https://github.com/Kailari/kokonaisuus/tree/part-8))

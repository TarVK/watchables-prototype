# Planning

Watchable primitives should provide a way of creating fields that are settable and observable, as well as observable derived values that can be computed using other watchable values. For example:

```ts
const someField = new Field(25);
const someDerivedValue = new Derived(watch => {
    return watch(someField) * 4;
});
```

The desired interface (ignoring feasibility of implementation) looks as follows:

```ts
interface IWatchable<X> {
    /** The current value */
    get(): X;
    /** A function to register listeners for value changes, returns the unsubscribe function */
    onChange(listener: () => void): () => void;
}
```

The value of a watchable can be read, and we can be informed of value changes.

In attempting to design such a system of watchable values, several challenges are encountered when considering specific use-cases or desirable traits. Each of these challenges is a section of this file.

## Challenge: cached derived values and invalid state

When we compute derived values each time they are accessed, its value is very predictable. It however seems like a waste to compute the value on every access, since we know exactly when the value has to be recomputed (when any of its dependencies change). This is a perfect opportunity to provide caching capabilities that improve performance, while remaining completely transparent.

When using the discussed interface, we however encounter a problem. Consider the following graph, where `A` is a watchable field, and `B` is a derived value depending on `A` and `C` is a derived value depending both on `A` and `B`:

```txt
┌-> A <-┐
|       |
B       |
^       |
|       |
└------ C
```

Then when `A` changes, it dispatches a change event both to `C` and `B`. The order between `C` and `B` is arbitrary, so `C` may be called first. When `C` dispatches its events now, it could cause `C`'s value to be read. When this happens, `B` still has the old computed value since it's not yet aware of `A`'s value change. Therefore the value computed by `C` will be incorrect since it uses an outdated version of `B`. Thus this approach yields invalid states.

### Solution

By providing a two-phase event system, we can make sure that all derived values are marked as dirty before dispatching change events. Similarly we can make sure that only the first change event after a dirty marking happens, in order to ensure no exponential number of change events can occur per update.

#### General updated interface

Interface:

```ts
interface IWatchable<X> {
    /** The current value */
    get(): X;
    /** A function to register listeners for value dirtying, returns the unsubscribe function */
    onDirty(listener: () => void): () => void;
    /** A function to register listeners for value changes, returns the unsubscribe function */
    onChange(listener: () => void): () => void;
}
```

Invariants for every watchable `w`:

1.  Notifies: if for `const v1 = w.get()` and `const v2 = w.get()` with `v1` obtained before `v2` we have `v1 != v2`, then `w` must have dispatched `w.change` events before `v2` could have been obtained
2.  NoRedundantEvents: Between any two `w.get()` calls, at most a single `w.dirty` (or symmetrically `w.change`) event is dispatched
3.  DirtyBeforeChange: When `w.change` is dispatched between two `w.get()` calls, `w.change` is always preceded by `w.dirty`: `w.get() ⋅ w.dirty ⋅ w.change ⋅ w.get()`

Usage constraint for watchables:

1.  DirtyBeforeRead: during `w.dirty` event calls, `w.get()` behavior is disallowed and throws an error

Note that these invariants do not specify anything about redundancy of change events when values are read. The value of a field after a `change` event, might be equivalent to the value before said event. The redundancy property only describes that no events are dispatched if events are not being read.

#### Derived values

Derived values will track dependencies, forward dirty and change events to its dependencies, and only recompute the value if necessary on access.

Code sketch:

```ts
class Derived<X> implements IWatchable<X> {
    protected compute: (watch: <T>(dependency: IWatchable<T>) => T) => X;
    protected value: X;

    protected dependencies: Map<IWatchable<unknown>, () => void> = new Map();

    protected dirty: boolean = true;
    protected signaled: boolean = false; // Whether a broadcast has occured since marked dirty

    public constructor(compute: (watch: <T>(dependency: IWatchable<T>) => T) => X) {
        this.compute = compute;
    }

    /** @override */
    public get(): X {
        this.checkNotDispatchingDirty(); // Make sure this getter isn't called while callDirtyListeners is being invoked
        if (this.dirty) this.updateValue();
        return this.value;
    }

    protected updateValue() {
        this.dirty = false;

        const newDependencies = new Set<IWatchable<unknown>>();
        this.value = this.compute(dependency => {
            newDependencies.add(dependency);
            return dependency.get();
        });

        // Remove old dependencies
        for (const [dependency, unsub] of this.dependencies)
            if (!newDependencies.contains(dependency)) {
                unsub();
                this.dependencies.remove(dependency);
            }

        // Add new dependencies
        for (const dependency of newDependencies)
            if (!this.dependencies.has(dependency)) {
                const unsubDirty = dependency.onDirty(this.dirtyListener);
                const unsubChange = dependency.onChange(this.changeListener);
                this.dependencies.set(dependency, () => {
                    unsubDirty();
                    unsubChange();
                });
            }
    }

    protected dirtyListener = () => {
        if (this.dirty) return;
        this.dirty = true;
        this.signaled = false;
        this.callDirtyListeners();
    };

    protected changeListener = () => {
        if (this.signaled) return;
        this.signaled = true;
        this.callChangeListeners();
    };

    // TODO: write (trivial) code for listener management
}
```

Invariants/properties of `w = new Derived(c)` for some compute function `c`, with a set of dependencies `D: Set<IWatchable<unknown>>`:

1.  Transparency: At any given point `w.get() == c(...)`
2.  Caching: Between any two internal calls to `c`, there is a `d` in `D` such that `d.dirty` was dispatched
3.  Laziness: Any internal call to `c` is always followed by (completing) a call to `w.get`. I.e. `c` is never called if its value is not requested

## Challenge: derived value garbage collection

Derived values register callbacks to their dependencies. As a consequence, the dependency now has a runtime reference to the derived value. This means that even if no reference to the derived value exists elsewhere that would actually want to read the value, the mere existence of this reference prevents the derived value from being garbage collected. Similarly the listeners will forever remain registered in the dependencies, even if the derived value is never read again.

### Solution

By using weak references, we make sure that derived value listeners do not prevent garbage collection. Additionally we could make it so as soon as a derived value is dirty or signaled it will remove its dependency listeners, but this is not essential. Doing this will simply remove the amount of listeners that have to be invoked for derived values whose values may be read in the future still, but are not actively being read now.

```ts
type IOptionallyWeakRef<X> = {
    /** Retrieves the current value, which is either the set value, or undefined if the object got disposes */
    get(): X|undefined;
    /**
     * Sets whether the referenced object is allowed to be disposed
     * @param Whether the object is disposable
     */
    setDisposable(disposable: boolean): void;
};
/**
 * Creates an optionally weak reference from the given value
 * @param value The value to create a weak reference of
 */
function createOptionallyWeakRef<X>(value: X): IOptionallyWeakRef<X> {
    return createOptionallyWeakRefFromRef(new WeakRef(value));
}
/**
 * Creates an optionally weak reference from the given weak reference
 * @param value The weak reference to create an optionally weak reference for
 */
function createOptionallyWeakRefFromRef<X>(value: WeakRef<X>): IOptionallyWeakRef<X> {
    return {
        get: ()=>value.deref(),
        setDisposable(disposable) {
            if(disposable) value.hardRef = undefined;
            else value.hardRef = value.deref();
        }
    }
}

/**
 * Adds a new listener using the given `register` target, using a weak reference to the listener, allow for garbage collection
 * @param listenerRef The weak reference to the listener
 * @param register The function used to register the listener
 * @returns A function that can be used to deregister the dependency
 */
function createWeakListener(
    listenerFef: IOptionallyWeakRef<()=>void>,
    register: (listener: ()=>void)=>(()=>void)
): ()=>void {
    const l = () => {
        const listener = listenerFef.get();
        if(listener) listener();
        else deregister();
    };
    const deregister = register(l);
    const r = ()=>{
        finalizationRegistry.unregister(r);
        deregister();
    }
    finalizationRegistry.register(listenerFef.get(), deregister, r);
    return r;
}
const finalizationRegistry = new FinalizationRegistry<IRunnable>(remove=>remove());

class Derived<X> implements IWatchable<X> {
    ...
    this.weakDirtyListener = createOptionallyWeakRef(this.dirtyListener);
    this.weakChangeListener = createOptionallyWeakRef(this.changeListener);
    ...
    protected updateValue() {
        ...
        const unsubDirty = createWeakListener(this.weakDirtyListener, l=>dependency.onDirty(l));
        const unsubChange = createWeakListener(this.weakChangeListener, l=>dependency.onChange(l));
        this.dependencies.set(dependency, () => {
            unsubDirty();
            unsubChange();
        });
        ...
    }
    ...

    protected dirtyListeners = new Set<()=>void>();
    public onDirty(listener: ()=>void): ()=>void {
        this.dirtyListeners.add(listener);
        this.weakDirtyListener.setDisposable(false);
        return ()=>{
            this.dirtyListeners.remove(listener);
            if(this.dirtyLIsteners.size==0)
                this.weakDirtyListener.setDisposable(true);
        }
    }

    // TODO: Add logic of change listeners similar to dirty listeners
}
```

We use weak-references to allow for garbage collection, however we're also careful to not allow garbage collection as long as the watchable has listeners itself. When the derived value has listeners itself, even when it's not accessible from anywhere else, we want to keep it alive. The listeners can be user defined side-effects that should continue to be invoked. Additionally, whenever a listener is garbage collected (which can only occur if the derived value itself is also garbage collected), the finalization listener ensures that the listener is unregistered. This helps remove the listeners in case of a chain of derivables, such that when the leaf of the tree (/graph) is garbage collected, the newly created leaf can also be garbage collected. Consider the following scenario:

```
A          A
^          ^
|          |
B    ->    B
^
|
C
```

`C` has no listeners, and hence can be garbage collected. Then when it's disposed, it will remove its listeners to `B`. Now `B` also has no listeners anymore, allowing it to be garbage collected.

## Challenge: injecting dependencies into fields

A common scenario would be that an API specifies something as a field, but the user decides they want to synchronize this with the value of another watchable.

This could be done by observing the watchable, and whenever the value changes manually calling the field's setter. This approach however comes with 2 problems:

1. Invalid state: Similarly to the derived value invalid state problem, one may obtain an invalid state here. Consider fields `A` and `B`, and derived values `C` and `D`. Additionally we make field `B` mirror `C`'s value, based on the change event as described above. The graph looks as follows:
    ```txt
    A <-┐
    ^   C
    |   ^
    |   |
    |   B
    |   ^
    |   |
    └-- D
    ```
    Now when the value of `A` changes, `C` and `D` are marked as dirty. If the `change` event of `A` reaches `D` first, `D` will use the new value of `A`, but the old value of `B`. Only afterwards the `change` event will reach `C`, causing the value of `B` to be updated with the newly computed `C` value and releasing a new `dirty` and `change` event causing `D` to finally compute the correct value. In this chain of events, an intermediate invalid state is computed for `D`.
2. Instability: Multiple "users" could decide that a field should be synchronized with their value. This could cause the final value to be unpredictable and unstable, as the field's value will always reflect the last changed dependency. This would make it very hard to reason about the behavior of the system.

### Solution

Allow fields to be assigned the value of another watchable, after which the field will start behaving like this watchable. This more directly reflects the user's intent, and allows us to nicely manage the intended behavior from within the field itself.

Code sketch:

```ts
class SimpleField<X> implements IWatchable<X> {
    protected value: X | undefined;
    protected dirty: boolean = true;

    public constructor(init: X) {
        this.value = init;
    }

    /** @override */
    public get(): X {
        this.checkNotDispatchingDirty(); // Make sure this getter isn't called while callDirtyListeners is being invoked
        this.dirty = false;
        return this.value;
    }

    public set(value: X): void {
        this.value = value;

        if (this.dirty) return;
        this.callDirtyListeners();
        this.callChangeListeners();
        this.dirty = true;
    }

    // TODO: write (trivial) code for listener management
}
class Field<X> extends Derived<X> implements IWatchable<X> {
    protected simpleField: SimpleField<{value: X} | {source: IWatchable<X>}>;

    public constructor(init: X, source?: false);
    public constructor(init: IWatchable<X>, source: true);
    public constructor(init: X | IWatchable<X>, source: boolean | undefined) {
        super(watch => {
            const fieldValue = watch(this.simpleField);
            if ("value" in fieldValue) return fieldValue.value;
            else return watch(fieldValue.source);
        });
        this.simpleField = new SimpleField(source ? {source: init} : {value: init});
    }

    public set(value: X): void {
        this.simpleField.set({value});
    }

    public setSource(source: IWatchable<X>): void {
        this.simpleField.set({source});
    }
}
```

We split the code into `SimpleField` and `Field`. Here `SimpleField`'s implementation is quite trivial, and ensures that the watchable invariants hold. Trying to make sure the invariants hold when adding the option of specifying a source watchable is not as trivial, and therefore we made `Field` extend a derived value, such that adherence to invariants by derived values ensures our field also adheres to them. The logic of switching between a basic field value or another observed watchable then becomes trivial. This approach adds a bit of overhead to every field, but it's only a constant amount and I think it's worth the simplicity.

## Challenge: updating multiple fields without invalid states

Fields dispatch `dirty` and `change` events right after changing their value. This can be problematic if two fields are updated by the same procedure, and the intermediate state when only one field is updated reflects an invalid state for the system. We would like to be able to update multiple fields at once atomically, while dispatching all update events afterwards.

### Solution

Instead of making a field's `set` (or `setSource`) function update the field, it returns a "mutator" which can be used to dispatch updates. These dispatchers can then also be used to update values without releasing events.

It would have the following interface:

```ts
interface IMutator {
    /** Fully performs the mutation (calling both perform and signal)*/
    commit(): void;
    /** @deprecated Performs the change and dispatches the dirty event, without signalling a change */
    perform(): void;
    /** @deprecated Broadcasts the change event, requires perform to be invoked first */
    signal(): void;
}
```

This would require direct field usage to have one extra step:

```ts
someField.set("newValue").commit();
```

It however also makes it easy for a function to return a mutator:

```ts
function doSomething(): IMutator {
    const value = ...; // Compute the value according to some logic
    return myField.set(value);
}
```

And finally, it allows mutators to be combined, such that all mutations can be invoked together and treated as atomic:

```ts
function doSomething2(): IMutator {
    return synchronized(add => {
        add(myField2.set(25));
        add(doSomething());
    });
}
```

The goal would be that `commit` calls are only made within event handlers that deal directly with user interaction events, while all reusable functions/methods simply return mutators.

There are 2 caveats that this system introduces:

1. Notifies are not ensured, I.e. users can now call `perform` without calling `signal`. This breaks watchable invariant `1`.
2. Users might forget to call the mutators, causing the mutation not to occur at all, leading to bugs / unexpected behavior

## Challenge: change events may occur without the read value being different

When updating a field, there is potential for descendants to be affected, but this is not always the case. In some cases, updating a field leads to an equivalent value in the descendent. It is a shame that this causes recomputations nevertheless. Consider the following scenario:

```ts
const field = new Field(10);
const isPositive = new Derived(watch=>watch(field)>0);
const somethingHeavy = new Derived(watch=>{
    if(watch(isPositive)) return ... ; // Some intensive computation
    return ... ;
});

field.set(25).commit();
```

somethingHeavy will dispatch a `change` event, and upon reading the `somethingHeavy.get()` during this event, we notice that the value is recomputed but is semantically (if it weren't for mutability) equivalent to the old value. This means that we performed a redundant (heavy!) computation.

Notice we can not read values during the `dirty` events, and hence we can not prevent values from being marked `dirty`. We could read dependency values during `change` events in derived values before deciding whether to dispatch a `change` event ourselves, but this would contradict the laziness invariant of derived values.

### Solution

We will not attempt to prevent dispatching of dirty/change events, since we did not find a way to do this without harming the other invariants we established and desire. Instead we shall focus on preventing recomputations, such that despite `change` events occurring, a (shallow) equivalence check on the value obtained from `get` before and after the event can filter out superfluous events.

E.g. watchable usage as follows:

```ts
subscribe(myWatchable, (newValue)=>{
    ...; // Do something with the value when it changes
});
```

would be able to only call the callback function when actual value changes occur.

We can incorporate this idea into derived value updates. Before computing the new value of a derived value, we check whether any of the dependencies have changed. If not, we do not need to recompute our value either.
Since we are already depend on these values, and will read them to compute the new value, we don't invalidate the laziness property of derived values either. One notable nuance here is that the values that the new computation depends on, is not necessarily the same as the old computation. However, under the assumption that our computation is a pure function with the exception of watched fields, we know that we must be in one of two cases:

1. None of the dependencies changed, in which case the new computation being pure results in all the same dependencies being read, hence the laziness not being invalidated.
2. One of the dependencies changed, in which case the first accessed changed dependency would have to be read again by the computation, in order for it to make an alternative decision. Hence if stop checking dependencies right after hitting the first detected difference, no extra unused dependencies will be read and hence the laziness is not invalidated. This argument requires us to be able to access the dependencies in the same order as the first access that occurred in the computation.

We can now modify our derived value class to add this behavior. Code sketch:

```ts
type IDependency = {
    watchable: IWatchable<unknown>,
    value: unknown,
    remove: ()=>void
};
class Derived<X> implements IWatchable<X> {
    protected dependencies: IDependency[] = [];
    ...

    /** @override */
    public get(): X {
        this.checkNotDispatchingDirty(); // Make sure this getter isn't called while callDirtyListeners is being invoked
        if (this.dirty) this.updateValue();
        return this.value;
    }

    protected updateValue() {
        this.dirty = false;
        if(!this.requiresRecompute()) return;

        let i = 0;
        const foundDependencies = new Set<IWatchable<unknown>>();
        const newDependencies: IDependency[] = [];
        this.value = this.compute(dependency => {
            const value = dependency.get();

            // If we already registered this dependency, continue
            if(foundDependencies.contains(dependency)) return value;

            const curDependencyI = this.dependencies[i];
            if(curDependencyI?.watchable == dependency)
                newDependencies.push(curDependencyI);
            else {
                // Dispose the old dependency (but only if we didn't subscribe to it earlier)
                if(!foundDependencies.contains(curDependencyI.watchable))
                    curDependencyI.remove();

                // Subscribe to the new dependency
                const unsubDirty = dependency.onDirty(this.dirtyListener);
                const unsubChange = dependency.onChange(this.changeListener);
                newDependencies.push({
                    watchable: dependency,
                    value,
                    remove: ()=>{
                        unsubDirty();
                        unsubChange();
                    }
                });
            }

            // Track the used dependencies and corresponding current index, and return the value
            foundDependencies.add(dependency);
            i++;
            return value;
        });

        // Remove all remaining dependencies we did not encounter
        this.dependencies.slice(i).forEach(({watchable, remove})=>{
            if(!foundDependencies.contains(watchable))
                remove();
        });

        // Update the dependency list
        this.dependencies = newDependencies;
    }

    protected requiresRecompute(): boolean {
        for(const {watchable, value} of this.dependencies)
            if(watchable.get() != value) return true;
        return false;
    }

    ...
}
```

This implementation might unsubscribe and resubscribe to the same dependency during a computation, but this overhead should not be significant.

To make this improvement more powerful, we could also introduce a class used to check whether newly computed values are actually different (based on user specified equivalence) which returns the old value if equal:

```ts
const field = new Field([2, 3, 4]);
const checked = new Checked(
    field,
    (vOld, vNew) => vOld.length == vNew.length && vOld.every((v, i) => v == vNew[i])
);
const derived = new Derived(watch => watch(checked).map(v => v * 2));

field.set([2, 3, 4]).commit();
```

Now `derived` does not have to recompute, at the expense of `checked` having to check for list equivalence every time the value is read.

## Challenge: dealing with asynchronous computations that should be watchable/lazy/cached

We would like to be able to use asynchronous computations together with our watchables system. Such that we can make use of data fetches in our system too.

### Solution

The idea is that a default value is provided, and only once a async computation finishes, the value is updated. This converts an async value, to a synchronous one that updates over time.

```ts
const smth: IWatchable<number> = new Loadable(0, async () => {
    const response = await fetch("/api/number");
    const number = await response.json();
    return number;
});
```

Additionally, we would like to combine this with watchable dependencies:

```ts
const smth: IWatchable<number> = new Loadable(0, async watch => {
    const response = await fetch(watch(apiPath));
    const number = await response.json();
    return number * watch(multiplier);
});
```

The implementation code sketch:

```ts
class Derived<X> implements IWatchable<X> {
    protected dependencies: IDependency[] = [];
    protected valueID: number = 0;
    ...

    protected updateValue() {
        this.dirty = false;
        if(!this.requiresRecompute()) return;

        for(const {remove} of this.dependencies) remove();

        const valueID = ++this.valueID;
        const foundDependencies = new Set<IWatchable<unknown>>();
        this.value = this.compute(dependency => {
            const value = dependency.get();

            // In case the dependency is registered after a new value is computed, don't register it
            const outdated = valueID != this.valueID;
            if(outdated) return;

            // If we already registered this dependency, continue
            if(foundDependencies.contains(dependency)) return value;

            // Subscribe to the new dependency
            foundDependencies.add(dependency);
            const unsubDirty = dependency.onDirty(this.dirtyListener);
            const unsubChange = dependency.onChange(this.changeListener);
            this.dependencies.push({
                watchable: dependency,
                value,
                remove: ()=>{
                    unsubDirty();
                    unsubChange();
                }
            });
            return value;
        });
    }

    ...
}
class Loadable<X> extends Derived<X> {
    protected valueID: number = 0;
    public constructor(
        init: X,
        compute: (watch: <T>(dependency: IWatchable<T>) => T) => Promise<X>
    ) {
        const field = new SimpleField<X>();
        const asyncResult = new Derived<Promise<X>>(compute);
        super(watch=>{
            const result = watch(field);

            // Perform the computation, which can update the result in the future
            const valueID = ++this.valueID;
            watch(asyncResult).then(result=>{
                const outdated = valueID != this.valueID;
                if(outdated) return;

                const isOld = result==field.get();
                if(isOld) return;

                field.set(result);
            });

            return result;
        });
        this.value.set(init);
    }
}
```

We simplified the dependency management system of derived values here, which now also allows for new dependencies to be registered until a new computation is started. We then used this to define a loadable value, whose result is defined by the result of a field. Meanwhile reading the value of the field also reads the async computation, and if the result of this is different from the old field value, and no new async computation has started yet, the field's value is updated.

## Challenge: dealing with metadata

In the case of loadable values, it would be nice to be able to check whether the value of a watchable is still being loaded and is expected to update in the future. Similarly it would be nice to be able to define derived values which may throw an error in which case they fallback to a default value, while still being able to read such errors.

We would like to be able to add metadata to watchables, in a way that derived-values do not have to concern themselves with propagating this. This should be similar to the idea of monads in functional programming.

### Solution

We let users define metadata types which specify how data of a collection of values can be combined implicitly, such that derived value computations don't have to manually do this.

```ts
interface IMetadataType<X> {
    /**
     * Specifies how to obtain a watchable value of this type, given a list of watchable values from dependencies
     * @param values The watchable values obtained from dependencies
     * @returns The combined watchable value
     */
    combine(values: IWatchable<IWatchable<X>[]>): IWatchable<X>;
}
interface IMetadata {
    /**
     * Retrieves the metadata of the given type
     * @param type The metadata type to get the value for
     * @returns The watchable value specifying the metadata that may change over time
     *
     * @remark This is a pure function, the same watchable is returned every time for a given type
     */
    get<T>(type: IMetadataType<T>): IWatchable<T>;
}
interface IWatchable<X> {
    /** The current value */
    get(): X;
    /** The collection of metadata properties of this watchable */
    readonly metadata: IMetadata;
    /** A function to register listeners for value dirtying, returns the unsubscribe function */
    onDirty(listener: () => void): () => void;
    /** A function to register listeners for value changes, returns the unsubscribe function */
    onChange(listener: () => void): () => void;
}
```

Then, we can rename the `Derived<X>` value of earlier that does not specify metadata to `SimpleDerived<X>`, and use it in defining our higher level derived value.

Code sketch:

```ts
class Derived<X> extends SimpleDerived<X> implements IWatchable<X> {
    public readonly metadata = new DerivedMetadata(
        new SimpleDerived<IMetadata[]>(watch => {
            const value = watch(this); // We don't care about the value, but need to force the dependencies to be calculated

            return this.dependencies.map(({watchable}) => watchable.metadata);
        })
    );
}
class DerivedMetadata implements IMetadata {
    protected cache = new Map<IMetadataType<unknown>, IWatchable<any>>();
    protected sources: IWatchable<IMetadata[]>;

    /**
     * Creates a new metadata instance, whose values will be computed by combining the values of the provided sources
     * @param sources The list of sources. Note that you do not have to check deep equivalence on the list, that is done automatically by this metadata to prevent unnecessary recomputations
     */
    public constructor(sources: IWatchable<IMetadata[]>) {
        this.sources = new Checked(
            sources,
            (vOld, vNew) =>
                vOld.length == vNew.length && vOld.every((v, i) => v == vNew[i])
        );
    }

    /** @override */
    public get<T>(type: IMetadataType<T>): IWatchable<T> {
        if (this.cache.has(type)) return this.cache.get(type);

        const sourceValues = new SimpleDerived<IWatchable<T>[]>(watch => {
            const sources: IMetadata[] = watch(this.sources);
            const sourceValues: IWatchable<T>[] = sources.map(metadata =>
                metadata.get(type)
            );
            return sourceValues;
        });
        const combined = type.combine(sourceValues);
        this.cache.set(type, combined);
        return combined;
    }
}
```

And for other watchables, we can create mutable metadata:

```ts
class MetadataMap implements IMetadata {
    protected map = new Map<
        IMetadataType<unknown>,
        {
            source: SimpleField<{value: unknown} | {source: unknown} | undefined>;
            result: IWatchable<unknown>;
        }
    >();
    protected fallback: IMetadata;

    public constructor(fallback: IMetadata = new DerivedMetadata(new Derived(() => []))) {
        this.fallback = fallback;
    }

    /** @override */
    public get<T>(type: IMetadataType<T>): IWatchable<T> {
        this.initType(type);
        return this.map.get(type).result;
    }

    /**
     * Registers a new metadata value
     * @param type The type to set the value for
     * @param value The value to be used by this type
     * @returns The mutator that can be used to dispatch the change
     */
    public set<T>(type: IMetadataType<T>, value: T): IMutator {
        this.initType(type);
        return this.map.get(type).source.set({value});
    }

    /**
     * Registers a new metadata value
     * @param type The type to set the value for
     * @param source The value to be used by this type
     * @returns The mutator that can be used to dispatch the change
     */
    public setSource<T>(type: IMetadataType<T>, source: IWatchable<T>): IMutator {
        this.initType(type);
        return this.map.get(type).source.set({source});
    }

    /**
     * Removes the type, such that we fallback to the parent value
     * @param type The type for which to delete the value
     * @returns The mutator that can be used to dispatch the change
     */
    public delete(type: IMetadataType<unknown>): Mutator {
        if (this.map.has(type)) return this.map.get(type).source.set(undefined);
        return dummyMutator();
    }

    /**
     * Initializes the stored value for the given type
     * @param type The type to initialize
     */
    protected initType(type: IMetadataType<unknown>): void {
        if (this.map.has(type)) return;

        const source = new SimpleField(undefined);
        const parent = this.fallback.get(type);
        const resultSources = new Derived<IWatchable<unknown>[]>(watch => {
            const local = watch(source);
            if (!local) return [parent];
            if ("value" in local) return [new Derived(() => local.value), parent];
            return [local.source, parent];
        });
        this.map.set(type, {
            source,
            result: type.combine(resultSources),
        });
    }
}
```

## Feature: debounce/throttle

Sometimes we do not want results to instantly update. If there's a heavy computation, and its dependencies change very frequently, computing the derived value on every change might be too much.

### Solution (failed)

We create a primitive that tracks when the watchable that it wraps changes, but only updates its own value after no change has occurred for at least some minimal delay. It could be that the stream of updates is continuous, in which case this would never update, so we also add a configurable maximum delay after which at least some value update is made.

This has high-potential of causing invalid intermediate state, if a derived value both depends on a debounced value, and a source of that debounced value. To deal with this, we can use some metadata type that tracks whether any data is currently being debounced. This way the derived value can check whether the data is being debounced, and return an old value until it no longer is.

Code sketch:

```ts
class Debounced<X> extends Derived<X> {
    protected debounceTime: number;
    protected maxDebounceTime: number;

    protected readonly metadataMap = new MetadataMap(new DerivedMetadata(
        new SimpleDerived<IMetadata[]>(watch => {
            const value = watch(this); // We don't care about the value, but need to force the dependencies to be calculated

            return this.dependencies.map(({watchable})=>watchable.metadata);
        });
    ));
    public readonly metadata: IMetadata = metadataMap;

    protected debounce: undefined | {
        maxDebounceTimeoutID: string,
        debounceTimeoutID: string,
        type: "dirty"|"change"
    };

    public constructor(source: IWatchable<X>, debounce: number, maxDebounce: number) {
        super(watch=>watch(source));
        this.debounceTime = debounce;
        this.maxDebounceTime = maxDebounce;
    }


    protected dirtyListener = () => {
        this.scheduleSignal();
    };
    protected changeListener = () => {
        this.scheduleSignal();
        this.debounce.type = "change";
    };

    protected scheduleSignal() {
        if(!this.debounce) {
            this.debounce = {
                type: "dirty",
                maxDebounceTimeoutID: setTimeout(this.signal, this.maxDebounceTime),
                debounceTimeoutID: setTimeout(this.signal, this.debounceTime)
            };
        } else {
            clearTimeout(this.debounce.debounceTimeoutID);
            this.debounce.debounceTimeoutID = setTimeout(this.signal, this.debounceTime);
        }
    }

    protected signal = ()=>{
        if(!this.debounce) return;
        clearTimeout(this.debounce.maxDebounceTimeoutID);
        clearTimeout(this.debounce.debounceTimeoutID);

        this.callDirtyListeners();
        this.dirty = true;
        if(this.debounce.type=="change")
             this.callChangeListeners();

        this.debounce = undefined;
    }
}
```

#### Failed

This turns out not to work, because no new dirty events are dispatched unless the source value is read. The value is not read until we dispatch a dirty event, hence we can't guarantee that the timeout is reset when the source value changes again.

### Simplified solution

We create a primitive that tracks when the watchable that it wraps changes, but only updates its own value after some delay since it last changed itself. It ensures that an update occurs no more often than once every `x` amount of milliseconds. It could be that the stream of updates is continuous, in which case the delay essentially encodes an update frequency.

Code sketch:

```ts
class Throttled<X> extends Derived<X> {
    protected throttleTime: number;

    protected throttle: undefined | {
        timeoutID: string,
        type: ISignal,
        signal?: ISignal,
    };

    public constructor(source: IWatchable<X>, throttle: number) {
        super(watch=>watch(source));
        this.throttleTime = throttle;
        this.source = source;
    }


    protected dirtyListener = () => {
        if(this.throttle) {
            if(!this.throttle.signal) this.throttle.signal = "dirty";
        } else {
            this.throttle = {
                timeoutID: setTimeout(this.throttleEnd, this.throttleTime)
                type: "dirty",
            };
            this.callDirtyListeners();
        }
    };
    protected changeListener = () => {
        if(this.throttle?.type=="change") {
            this.throttle.signal = "change";
        } else {
            if(this.throttle)
                clearTimeout(this.throttle.timeoutID);
            this.throttle = {
                timeoutID: setTimeout(this.throttleEnd, this.throttleTime),
                type: "change",
            };
            this.callChangeListeners();
        }
    };

    protected throttleEnd = ()=>{
        if(!this.throttle) return;

        if(this.throttle.signal!=undefined)
            this.callDirtyListeners();
        if(this.throttle.signal=="change")
             this.callChangeListeners();
        this.throttle = undefined;
    }
}
type ISignal = "dirty"|"change"
```

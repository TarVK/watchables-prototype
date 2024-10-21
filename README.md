# Watchables

Watchables are primitive datatypes, similar to observables, except optimized for usage in "derived" values. Derived values can be seen as "pure" functions that rely on foreign watchables rather than parameters. They compute their value lazily and cache their results to prevent unnecessary recomputes. They ensure that at any point in time, accessing their value is equivalent to executing the compute function directly:

```ts
const compute = watch => watch(someField) * 2;
const derived = new Derived(compute);
```

=>

```ts
compute(w => w.get()) == derived.get();
```

This equivalence holds at any point in time (except for when dirty events are dispatched), and for any compute function (which is pure when not considering usage of other watched watchables).

## Status
This repository contains a prototype implementation that I'm also using in a private project. Eventually I hope to develop my primitives into a dedicated package with some nice features, but I want to experience what it's like to use the current system before committing to it and turning it into a public package.  If you want to play around with this idea yourself, you can copy the source code to any project, and leave any feedback in the issue tracker. Any new ideas are welcome. 

## Example

```tsx
import {Field} from "./Field";
import {useWatch} from "./react/useWatch";

const firstName = new Field("Bob");
const lastName = new Field("Johnson");
const fullName = new Derived(watch=>`${watch(firstName)} ${watch(lastName)}`);

const NameInput: FC<{firstName: Field<String>, lastName: Field<String>}> = ({firstName, lastName})=>{
    const watch = useWatch(); // Use the 'watch' hook to subscribe to value changes
    return <>
        <input value={watch(firstName)} onChange={e=>firstName.set(e.target!.value).commit()} />
        <input value={watch(lastName)} onChange={e=>lastName.set(e.target!.value).commit()} />
    </>;
};
const NameDisplay: FC<{name: IWatchable<String>}> = ({name})=>
    // or use the watcher component for fine-grained watching in a single component
    <Watcher> 
        {watch=>watch(name)}
    </Watcher>
const NameChanger: FC<{firstName: Field<String>, lastName: Field<String>}> = ({firstName, lastName})=>
    <button onClick={
        // Chain multiple mutations together, and commit them as an atomic event
        ()=>firstName.set("John").chain(lastName.set("Smith")).commit()
    }>
        change
    </button>

const App = ()=><>
        <NameDisplay name={fullName}/>
        <NameInput firstName={firstName} lastName={lastName}/>
        <NameChanger firstName={firstName} lastName={lastName}/>
    </>;
```

Note that `.set` on a field does not immediately change the value. Instead it returns a mutator that has to be committed. These mutations can be combined together, to commit them atomically (e.g. no change event is dispatched until all mutations are preformed). This allows developers to ensure that no illegal intermediate state is communicated to listeners of watchables. The idea is that API (/model) functions should return mutations, so that these can be combined in extended APIs/models, and that these mutations are only committed by event handlers. 
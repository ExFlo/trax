# Trax API

<!-- Table of content -->
  * [Data object declaration](#data-object-declaration)
    + [@Data](#@data)
    + [@ref](#@ref)
    + [@computed](#@computed)
  * [Creation, disposal and JSON conversion](#creation,-disposal-and-json-conversion)
    + [new operator](#new-operator)
    + [getter auto-creation](#getter-auto-creation)
    + [create<T>(c: Constructor<T> | Factory<T>, json?: Object): T](#create)
    + [convertToJson(d: any, converter?: JSConverter): Object](#converttojson)
    + [dispose(traxObject: any, recursive = false): void](#dispose)
  * [Watchers and Trackers](#watchers-and-trackers)
    + [watch(traxObject: any, fn: WatchFunction): WatchFunction | null](#watch)
    + [unwatch(traxObject: any, watchFn: WatchFunction | null)](#unwatch)
    + [numberOfWatchers(traxObject: any): number](#numberofwatchers)
    + [track(traxObject: any, fn: TrackFunction): TrackFunction | null](#track)
    + [untrack(o: any, trackFn: TrackFunction): void](#untrack)
    + [numberOfTrackers(o: any): number](#numberOfTrackers)
  * [Versions and mutations](#versions-and-mutations)
    + [version(traxObject: any): number](#version)
    + [isMutating(traxObject: any): boolean](#ismutating)
    + [changeComplete(traxObject: any): Promise<void>](#changecomplete)
    + [commitChanges(traxObject: any, forceNewRefreshContext = false): void](#commitchanges)
  * [Trax objects relationships](#trax-objects-relationships)
    + [isDataObject(traxObject: any): boolean](#isdataobject)
    + [hasParents(traxObject: any): boolean](#hasparents)
    + [getParents(traxObject: any): any[] | null](#getparents)
  * [Property helpers](#property-helpers)
    + [hasProperty(traxObject: any, propName: string): boolean](#hasproperty)
    + [createProperty(traxObject: any, propName: string | number): any](#createproperty)
    + [resetProperty(traxObject: any, propName: string): any](#resetproperty)
    + [forEachProperty(traxObject: any, processor: (propName: string, internalPropValue: any) => void): void](#foreachproperty)


## Data object declaration

### @Data

Trax objects are identified thanks to the @Data class decorator that tell the trax pre-processor which classes should be processed (i.e. rewritten) before the typescript compilation runs.

```js
import { Data } from 'trax';

@Data class User {
    id: number;
    pseudo: string;
    email: string;
    preferences: UserPref; // UserPref is another trax object
}
```

Note: the typescript code generated by the trax pre-processor can be visualized in the console output but adding a ```// log``` comment in the @Data class:

```js
// the trax code generated for this class will be visible on the build console
@Data class User {
    // log
    id: number;
}
```

### @ref

Trax objects can only track properties of the following types:
- base types: *string*, *number*, *boolean*
- function types: *Function* or inline function type, like *(arg:string)=>void*
- any reference: *any*
- trax object class: e.g. *User* where User is a trax class
- Array collections of the previous types: e.g. *number[]*, or *User[]* or *User[][]*
- Dictionary collections of the previous types: e.g. *{[k:string]: User}* or *{[k:string]: User[]}*

As you will note, it is not possible to use a type interface. This is because the current trax pre-processor doesn't look deep in the typescript dependency graph and doesn't know if a type refers to a trax class, a normal class or a type interface.

This is where the **@ref** decorator comes into play as it allows to
- declare a property as a *reference* - i.e. trax will only track the reference of the object and not the internal object changes (that would be tracked for a trax object used as property without the @ref decorator)
- declare up to which level a collection should be tracked (e.g. it gives the possibility to only track an array, and not its items)

```js
interface NameHolder {
    name: string;
}

@Data class User {
    // the following property will be tracked in depth
    supervisor: User;

    // the following properties will be tracked by reference only
    @ref supervisor2: User;
    @ref person: NameHolder;
}
```

In case of collections, **@ref.depth** must be used to tell how deep the collection (or sub-collection) should be tracked:

```js
@Data class RefCol {
    // here only 'values' reference is tracked, not the list - equivalent to @ref.depth(1)
    @ref values: string[] = [];
    // here the list is tracked (e.g. on item add/remove) and the item references are tracked
    @ref.depth(2) people: NameHolder[];
    // here the dictionary, the list and the item references are tracked
    @ref.depth(3) names: { [name: string]: NameHolder[] }; 
}
```

Note: **@ref.depth can only be used on Arrays and Dictionaries** - in other word, it will not work on normal trax object properties.

### @computed

Properties expressed through getters that only depend on their trax object can be flagged with the **@computed** decorator to have their content *[memoized][memoization]* :

```js
// example from the TodoMVC use case
@Data export class TodoApp {
    @ref filter: "ALL" | "ACTIVE" | "COMPLETED" = "ALL";
    list: Todo[];

    @computed get listView(): Todo[] {
        if (this.filter === "ALL") {
            return this.list;
        } else {
            const isComplete = (this.filter === "COMPLETED");
            return this.list.filter(item => item.completed === isComplete);
        }
    }
```

## Creation, disposal and JSON conversion

### new operator
The simplest - and most natural - way of creating a trax object is to create it through the new operator:

```js
@Data class User {
    id: number;
}
let u123 = new User();
u123.id = 123;
```

Note: trax objects currently don't accept constructors.

### getter auto-creation
The second way to instantiate a trax object is to have it automatically created when retrieved through a getter from another trax object (as a form of dependency injection).

```js
@Data class Name {
    firstName = "";
    lastName = "[no name]";
}
@Data class User {
    name: Name;         // will be auto-created (as not marked as optional)
    birthName?: Name;   // will not be auto-created (optional)
}

const u = new User();
console.log(u.name.lastName); // prints "[no name]"
console.log(u.birthName);     // prints "undefined"
```

### create
The third way to instantiate a trax object is to call the **create** factory:
```js
function create<T>(c: Constructor<T> | Factory<T>, json?: Object): T {...}
```
The purpose of the create factory is to create and initialize a trax object from data passed as a json structure.
```js
let u123 = create( User, {id:123} );
```
Note: when passing a deep JSON structure, trax will lazily instantiate the sub-objects - i.e. the instantiation will be done when they are retrieved for the 1st time. So if a part of the object graph is not accessed, it will never be instantiated.

```js
@Data class Node {
    name: string;
    next?: Node;
}
let nd = create( Node, {name:"first", next: {name:"second", next: {name:"third"}}} );

console.log(nd.name); // "first"
// nd.next is not fully instantiated, nd.next.next is not even created at this stage
console.log(nd.next.name); // "second"
// nd.next.next is created by not fully initialized
console.log(nd.next.next.name); // "third"
// all nodes are created & initialized
```

### convertToJson

```js
function convertToJson(d: any, converter?: JSConverter): Object {...}
```
As the name implies, the goal of this function is to convert a trax object (and its child objects) to a JSON structure - so that it can be stored or sent to a server for instance:

```js
const nd = create( Node, {name:"first", next: {name:"second", next: {name:"third"}}} );

nd.next.name = "new_second";

const json = convertToJson(nd);
console.log(json);
// output: {name:"first", next: {name:"new_second", next: {name:"third"}}}
```

In case of special conversion needs (e.g. to avoid converting sub-objects), a second *converter* argument can be passed to *convertToJson*. The converter is a function that will be called for each object property and that should return the value to set in the JSON result.

On top of the object to convert, the converter function is passed with a *JSConversionContext* argument that provides contextual information about the current conversion.

```js
interface JSConversionContext {
    getDefaultConversion(): any;   // call the default converter (that may call back the custom converter on sub-objects)
    getPreviousConversion(): any;  // return a JSON object if the current object has already been converted (e.g. in the case of a diamond graph) - return undefined otherwise
}
```

Example:
```js
const nd = create( Node, {name:"first", next: {name:"second", next: {name:"third"}}} );

function cc(o: any, cc: JSConversionContext) {
    if (o.constructor === Node && o.name==="third") {
        return "THIRD"; // should return undefined to remove this node from the output
    }
    return cc.getDefaultConversion(); // use the default converter by default
}

const json = convertToJson(nd, cc);
console.log(json);
// output: {name:"first", next: {name:"new_second", next: "THIRD"}}
```

### dispose

```js
function dispose(traxObject: any, recursive = false): void {...}
```
Last but not least, when a graph of trax objects has been created, it should be explicitly *disposed* when not used any longer. Indeed trax objects keep internal references to their parent objects (that also reference them) and that can create memory leaks if not properly cleaned-up.

In the case of an object A referencing an object B that also references an object C (i.e. A->B->C), calling dispose(B) will:
- remove B reference on A (forward reference)
- remove internal A reference on B (backward reference)
- remove internal B reference on C (backward reference on children)

Note: A and C will not be disposed. If you wish B and all its children to be disposed, then the second *recursive* argument should be set to true.


## Watchers and Trackers

### watch
```js
function watch(traxObject: any, fn: WatchFunction): WatchFunction | null {...}
type WatchFunction = (o: TraxObject) => void;
```
*watch* allows to register a callback to be asynchronously notified of a trax object changes. When the callback is called, the object being watched is *clean* again but will have a different version number (cf. [introduction][] or [version](#version) api below);

Any trax object can be watched, even though in practice applications will mostly focus on root objects.

```js
@Data class User {
    id: number;
    name: string;
}

const u = new User();
const cb = watch(u, ()=> {
    console.log(`user changed: ${u.name}[${u.id}]`);
});

u.name = "Donald Duck";
u.id = 42;

// as the watch callback is called asynchronously when all JS operations are complete
// the callback will only be called once (even though 2 changes were performed)
// and the console will show:
// user changed: Donald Duck[42]
```

*watch* will also be notified of changes that occurred in sub-objects:
```js
@Data class Node {
    name: string;
    next?: Node;
}
const n1 = create( Node, {name:"first", next: {name:"second", next: {name:"third"}}} );

const cb = watch(n1, ()=> {
    console.log(`node changed: ${u.name}`);
});

const n3 = n1.next.next;
n3.name = "THIRD"; // -> will trigger "node changed: first" in the console
```

Note: if watch is called multiple times with the same watch function, then the watch function will be called multiple times for the same change notification.

### unwatch
```js
function unwatch(traxObject: any, watchFn: WatchFunction | null) {...}
```
As the name implies, *unwatch* allows to un-register a watch callback.

Note: unwatch needs to be called for each watch callbacks. If the same callback has been registered multiple times, unwatch needs to be called multiple times as well.

### numberOfWatchers
```js
function numberOfWatchers(traxObject: any): number {...}
```

*numberOfWatchers* tells how many watch callbacks are attached to a trax object.

```js
const u = new User();
console.log(numberOfWatchers(u)); // prints 0

const cb = watch(u, ()=> {
    console.log(`user changed: ${u.name}[${u.id}]`);
});
console.log(numberOfWatchers(u)); // prints 1

unwatch(u, cb);
console.log(numberOfWatchers(u)); // prints 0
```

### track
```js
function track(traxObject: any, fn: TrackFunction): TrackFunction | null {...}
type TrackFunction = (o: TraxObject, operation: string, property?: string | number, previousValue?: any, newValue?: any) => void;
```

On the contrary to *watch*, the *track* function allows to be **synchronously** notified of a trax object changes. Using this function can lead to performance issues and should be handled with special attention.

As compared to watch, the track callback receives multiple arguments:
1. the reference to the object being tracked
2. a string indicating which operation was performed. It is usually "set" for classical value objects, but can also be Array mutation function names (such as "push" or "pop") on Array collections.
3. (optional) the name or index of the property being changed
4. (optional) the previous value of the property being changed
5. (optional) the new value set to the property

```js
const u = new User();
let count=0;
const cb = track(u, () => {
    count++;
    console.log(`track #${count} ${u.name}[${u.id}]`);
});

u.name = "Donald Duck";
// console: track #1 Donald Duck[0]
u.id = 42;
// console: track #2 Donald Duck[42]

untrack(u, cb);
```

### untrack
```js
function untrack(o: any, trackFn: TrackFunction): void {...}
```
*untrack* allows to un-register a track callback associated to a trax object (cf. previous example);

### numberOfTrackers
```js
function numberOfTrackers(o: any): number {...}
```
*numberOfTrackers* tells how many track callbacks have been registered on a trax object (similar to [numberOfWatchers](#numberOfWatchers)).



## Versions and mutations

As explained in the [introduction][], trax uses an internal versioning system to be able to tell watch functions if an object has changed (or which sub-part of an object changed, compared to a previous version).

### version
```js
function version(traxObject: any): number {...}
```

*version* returns the internal version associated to a trax object (0 will be returned for newly created objects or for non-trax objects);

Version numbers follow the following pattern:
- even numbers (e.g. 2) denote **clean** objects - i.e. objects that haven't been changed since last *watch()* callback notifications
- odd numbers (e.g. 3) denote **dirty** objects - that is objects that have changed since last *watch()* callback notifications

Note: version is useless for *track()* notifications as *track()* callbacks are called synchronously (cf. above).

```js
@Data class User {
    id: number;
    email: string;
}

const u = new User();
console.log(version(u)); // prints 0

u.id = 123;
console.log(version(u)); // prints 1
u.email = "donald@duck.com";
console.log(version(u)); // prints 1

await changeComplete(u); // cf. below
console.log(version(u)); // prints 2
```

### isMutating
```js
function isMutating(traxObject: any): boolean {...}
```

*isMutating* tells if a trax object is being changed (i.e. if its dirty, that is if its version number is odd);

```js
const u = new User();
console.log(isMutating(u)); // prints false

u.id = 123;
console.log(isMutating(u)); // prints true
u.email = "donald@duck.com";
console.log(isMutating(u)); // prints true

await changeComplete(u);    // cf. below
console.log(isMutating(u)); // prints false
```

### changeComplete
```js
function changeComplete(traxObject: any): Promise<void> {...}
```

*changeComplete* is a function that allows to *wait* until an object becomes *clean* again. Technically it registers a one-time callback that fulfills the promise returned by the function. In practice this function is mostly used for tests purposes (cf. above examples).

### commitChanges
```js
function commitChanges(traxObject: any, forceNewRefreshContext = false): void {...}
```

*commitChanges* allows to synchronously run the operations that are normally run in the trax micro-task and that lead to the *watch()* callbacks notifications. The function returns true if some changes were committed. The second argument allows to create a new refresh context in case no changes were committed. The refresh context is an object that stores all the changes to be committed for a given transaction. A new refresh context is systematically created when changes are committed.


## Trax objects relationships

### isDataObject
```js
function isDataObject(traxObject: any): boolean {...}
```

*isDataObject* tells if an object is a trax object or not.

```js
@Data class User {
    id: number;
    email: string;
}

console.log(isDataObject(new User()));      // prints true
console.log(isDataObject({hello:"world"})); // prints false
```

### hasParents
```js
function hasParents(traxObject: any): boolean {...}
```

*hasParents* tells if a trax object is referenced by other trax objects.

```js
@Data class Node {
    name: string;
    next?: Node;
}
const nd = create( Node, {name:"first", next: {name:"second", next: {name:"third"}}} );

console.log(hasParents(nd));           // prints false
console.log(hasParents(nd.next));      // prints true
console.log(hasParents(nd.next.next)); // prints true
```

### getParents
```js
function getParents(traxObject: any): any[] | null {...}
```

*getParents* returns the trax objects that reference a trax object;

```js
const ndA = create( Node, {name:"A"}),
    ndB = create( Node, {name:"B"}),
    ndC = create( Node, {name:"C"});

getParents(ndC); // returns null
ndA.next = ndC;
getParents(ndC); // returns [ndA]
ndB.next = ndC;
getParents(ndC); // returns [ndA, ndB]
```

Note: if an X object references a Y object several times (through several properties) it will appear several times in the list returned by getParents(Y).

## Property helpers
### hasProperty
```js
function hasProperty(traxObject: any, propName: string): boolean {...}
```

*hasProperty* tells if a property is supported by a trax object.

```js
@Data class User {
    id: number;
    email: string;
}
const u = new User();
console.log(hasProperty(u, "email"));   // prints true
console.log(hasProperty(u, "pseudo"));  // prints false
```

### createProperty
```js
function createProperty(traxObject: any, propName: string | number): any {...}
```

*createProperty* allows to force the creation of a property, even it it is marked as optional.

```js
@Data class BTreeNode {
    data: any = "[empty]";
    left: BTreeNode;
    right?: BTreeNode; 
}

const n = create(BTreeNode, {data: "root"});
// left is created at first get
console.log(n.left.data);  // prints "[empty]"
// but right is optional
console.log(n.right);      // prints undefined
// createProperty will force the creation:
createProperty(n, "right");
console.log(n.right.data); // prints "[empty]"
```

### resetProperty
```js
function resetProperty(traxObject: any, propName: string): any {...}
```

*resetProperty* allows to reset a property in the same state as when the object was created.

```js
// cf. BTreeNode in previous example

const n = new BTreeNode();
console.log(n.data);  // prints "[empty]"
n.data = "root";
console.log(n.data);  // prints "root"
n.data = "root2";
console.log(n.data);  // prints "root2"
resetProperty(n, "data");
console.log(n.data);  // prints "[empty]"
```

### forEachProperty
```js
function forEachProperty(traxObject: any, processor: (propName: string, internalPropValue: any) => void): void {...}
```

Last, *forEachProperty* allows to iterate overall all the properties defined on a trax object. The iteration function passed as argument will be called with 2 arguments:
1. the property name
2. the internal value stored for the property. This internal value may be different from the value retrieved when accessing the property, as it will be undefined until the first getter or setter is called:

```js
@Data class User {
    id: number;
    email: string;
    supervisor: User;
};

const u = new User();

forEachProperty(u, (name, internalValue) => {
    console.log(`${name}: ${internalValue}`);
});
// prints:
// id: undefined
// email: undefined
// supervisor: undefined

u.id = 123;
console.log(u.email); // prints "" (but will force email initialization)

forEachProperty(u, (name, internalValue) => {
    console.log(`${name}: ${internalValue}`);
});
// prints:
// id: 123
// email: 
// supervisor: undefined
```

[memoization]: https://en.wikipedia.org/wiki/Memoization
[introduction]: ../readme.md#How-does-it-work?

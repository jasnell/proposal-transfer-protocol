# Transfer Protocol

**Stage:** 0

**Champions:** TBD

**Authors:** TBD

## Status

This proposal is at Stage 0 and is seeking feedback on the core concept and
API design.

## Motivation

`ArrayBuffer.prototype.transfer()` (ES2024) introduced ownership transfer to
JavaScript: moving exclusive access from one holder to another, rendering the
original unusable. But the concept is hardcoded to `ArrayBuffer`. This proposal
generalizes it:

| Concept | Description |
|-|-|
| `Symbol.transfer` | A well-known symbol for the transfer protocol |
| `Object.transfer(obj)` | Static method to trigger transfer |
| `Reflect.transfer(obj)` | Reflect method for use in Proxy traps |
| Proxy `transfer` trap | Intercept transfer on Proxy objects |

### An Existing Pattern Without a Common Protocol

Multiple language, web, runtime, and user-land APIs already implement ownership transfer independently, each with its own method name and mechanism:

| API | Transfer Mechanism |
|-|-|
| `ArrayBuffer.prototype.transfer()` | moves buffer data, detached the original |
| `ReadableStream` / `WritableStream` locking | `getReader()` locks a stream to a single consumer; `releaseLock()` releases it |
| `DisposableStack.prototype.move()` | moves registered resources to a new stack, marks the original disposed |
| `postMessage()` transfer lists | `ArrayBuffer`, `MessagePort`, `OffscreenCanvas`, `ImageBitmap`, and others can be transferred across realms, detaching the source |
| Runtime-specific types | Node.js adds `FileHandle` as a transferable type (detached on send) |

These all exist as at least an attempt to either enforce, or allow for, exclusive
ownership of a stateful resource, but share no common protocol. There is no way to
write generic code that transfers an arbitrary resource, and no way for
user-defined types to participate. Proxy can intercept the underlying method calls
via existing traps (`get`, `apply`), but cannot distinguish a transfer operation
from an ordinary property access without inspecting the property key — a dedicated
trap makes the intent explicit and easier for engines to optimize.

### The Problem with Iterators

Iterators are a primary motivating case. An iterator is stateful and
single-consumer -- `.next()` advances state that cannot be rewound. Yet
nothing prevents multiple consumers from sharing a reference and interleaving
`.next()` calls:

```js
const iter = [1, 2, 3, 4].values();

// Two consumers unknowingly share the same iterator
consumeEvens(iter);
consumeOdds(iter);
// Each consumer sees an unpredictable subset of values
```

Same for async iterators:

```js
async function processStream(stream) {
  const iter = stream[Symbol.asyncIterator]();
  pipeline1(iter);  // starts consuming
  pipeline2(iter);  // races with pipeline1 -- data is split unpredictably
}
```

`ReadableStream.from(iter)` wraps an async iterator in a `ReadableStream` that has
locking — but the locking protects the *stream*, not the *iterator*. Any code that
retains a reference to `iter` can call `.next()` and pull values out from under
the stream:

```js
const iter = generateData();
const stream = ReadableStream.from(iter);
const reader = stream.getReader(); // stream is locked

await iter.next(); // bypasses the stream -- silently steals data
```

The stream has no way to prevent this because iterators have no ownership
primitive.

### Not All Iterables Are Transferable

The iteration protocol is duck-typed. Many iterators are plain objects with
no shared prototype:

```js
// A perfectly valid iterator -- but has no prototype-level transfer support
const myIter = {
  i: 0,
  next() { return { value: this.i++, done: this.i > 3 }; }
};
```

**Transfer is opt-in.** Calling `Object.transfer()` on an object without
`[Symbol.transfer]()` throws a `TypeError`:

```js
const plain = { next() { return { value: 1, done: false }; } };
Object.transfer(plain); // throws TypeError: Object is not transferable
```

Built-in iterators (Array, Map/Set, generators, Iterator Helpers) implement
`[Symbol.transfer]()` on their shared prototypes. User-land iterators must
implement it explicitly.

### Why a Protocol?

Adding `.transfer()` directly to `Iterator.prototype` risks collision with
existing user-land code. And transfer is not unique to iterators — built-in
APIs already implement it under different method names (`.transfer()`,
`.move()`, `getReader()`), and user-land types (database cursors, file
handles, resource wrappers) need it too. A Symbol-based protocol avoids
the collision while covering all of these. Every other MOP operation is
interceptable via Proxy; transfer should be too.

## Proposal

### `Symbol.transfer`

A new well-known symbol. An object implements the protocol by providing a
`[Symbol.transfer]()` method that:

1. Returns a new object that takes ownership of the receiver's state.
2. Detaches the receiver permanently (see below).

#### The Protocol Defines Shape, Not Behavior

The design follows `Symbol.dispose` / `Symbol.asyncDispose`: the spec
defines that the method exists, when it is called, and minimal constraints
on its return value (must be an object). Nothing requires that
`[Symbol.transfer]()` actually detaches the source or moves state, just as
nothing requires that `[Symbol.dispose]()` actually releases resources.
Detaching the source, throwing on post-transfer access, making disposal a
no-op on detached objects — these are contracts that implementations are
expected to follow but the engine cannot verify.

#### Transfer Does Not Ensure Exclusive Access

Transfer is a primitive for *moving* state, not a mechanism that *enforces*
exclusive ownership. `Object.transfer(obj)` detaches the source and returns
a new owner, but nothing prevents the caller from passing that new owner to
multiple consumers. `ArrayBuffer.prototype.transfer()` has the same property: the resulting
buffer is a fresh object, but the language does not prevent sharing it.

Transfer makes it *easier* to implement single-ownership patterns — the
source is rendered unusable, so one reference is eliminated — but the
discipline of not sharing the result is the caller's responsibility.

#### Custom Implementations Must Enforce Their Own Detach Semantics

**The engine does not enforce detachment for user-defined types.** `[[Transfer]]`
calls `[Symbol.transfer]()` and validates the return is an object -- nothing
more. The **implementor** must:

- Detach the original (clear state, set a flag).
- Throw on subsequent operational access (`.next()`, `.read()`, etc.).
- Throw if `[Symbol.transfer]()` is called again on a detached object.
- **Not** throw when `[Symbol.dispose]()` is called on a detached object --
  disposal must be a no-op. See "Interaction with Explicit Resource
  Management" below.

For built-in types, the engine enforces detachment internally. For
user-defined types, the pattern is:

1. Check a `#detached` flag; throw if already detached.
2. Move internal state to a new instance.
3. Clear internal state on the original and set `#detached = true`.
4. Return the new instance.

#### Example: Correctly Implementing the Protocol

```js
class OwnedResource {
  #data;
  #detached = false;

  constructor(data) {
    this.#data = data;
  }

  get detached() {
    return this.#detached;
  }

  use() {
    if (this.#detached) throw new TypeError('Resource has been detached');
    return this.#data;
  }

  [Symbol.transfer]() {
    if (this.#detached) {
      throw new TypeError('Cannot transfer a detached resource');
    }
    const data = this.#data;
    this.#data = undefined;
    this.#detached = true;
    return new OwnedResource(data);
  }

  [Symbol.dispose]() {
    if (this.#detached) return; // no-op if already transferred
    this.#data = undefined;
    this.#detached = true;
  }
}
```

### The `[[Transfer]]` Internal Method

A new essential internal method. For **ordinary objects**, `[[Transfer]]()`
performs:

1. If `! IsExtensible(O)` is `false`, throw a **`TypeError`** ("cannot
   transfer a frozen or sealed object").
2. Let `transferFn` be `? GetMethod(O, @@transfer)`. Returns `undefined` if
   absent; throws `TypeError` if present but not callable.
3. If `transferFn` is `undefined`, throw a **`TypeError`** ("object is not
   transferable").
4. Let `result` be `? Call(transferFn, O)`.
5. If `result` is not an Object, throw a `TypeError`
   ("`[Symbol.transfer]()` must return an object").
6. Return `result`.

Step 1 rejects frozen and sealed objects. Transfer mutates the object's
observable behavior (from functional to throwing on all operations), which
contradicts the immutability contract that `Object.freeze()` /
`Object.seal()` establishes. This check is only enforced by `[[Transfer]]`
-- a direct `obj[Symbol.transfer]()` call bypasses it.

Step 5 requires only that the return value be an object. Returning the same
type is convention, not enforced (same as `Symbol.iterator`).

Step 2 uses `GetMethod`, so if `[Symbol.transfer]` is a getter, it will be
invoked. Other Symbol protocols work the same way, but transfer is
irreversible -- a getter with side effects could cause surprising behavior.

Many iterators in the ecosystem are plain objects without
`[Symbol.transfer]()`. `Object.transfer()` will correctly throw on these
rather than silently misbehave.

For **Proxy exotic objects**, `[[Transfer]]()` is intercepted by the
`transfer` handler trap (see below).

### `Object.transfer(obj)`

Triggers `[[Transfer]]` on the argument.

```js
const a = new OwnedResource({ value: 42 });
const b = Object.transfer(a);

a.use();   // throws TypeError: Resource has been detached
b.use();   // { value: 42 }
```

**Steps:**

1. If `obj` is not an Object, throw a `TypeError`.
2. Return `? obj.[[Transfer]]()`.

Calling `obj[Symbol.transfer]()` directly is valid, but `Object.transfer()`
is preferred because it routes through `[[Transfer]]`, which provides:

- Argument type checking (must be an object)
- Protocol validation (`[Symbol.transfer]` must exist and be callable,
  return value must be an object)
- Proxy `transfer` trap dispatch

A direct `obj[Symbol.transfer]()` call bypasses all of this — no validation,
no Proxy interception. The static method is the safe default; the raw symbol
call is available for cases where the caller has already validated the object
and wants to avoid the overhead.

### `Reflect.transfer(target)`

The `Reflect` counterpart:

| Operation   | `Object` method        | `Reflect` method          | Proxy trap    |
|-------------|------------------------|---------------------------|---------------|
| Get         | —                      | `Reflect.get()`           | `get`         |
| Set         | —                      | `Reflect.set()`           | `set`         |
| Delete      | —                      | `Reflect.deleteProperty()`| `deleteProperty`|
| **Transfer**| **`Object.transfer()`**| **`Reflect.transfer()`**  | **`transfer`**|

**Steps:**

1. If `target` is not an Object, throw a `TypeError`.
2. Return `? target.[[Transfer]]()`.

Used from within Proxy traps to forward to default behavior:

```js
const proxy = new Proxy(target, {
  transfer(target) {
    console.log('transfer intercepted');
    return Reflect.transfer(target); // forward to default behavior
  }
});
```

### Proxy `transfer` Handler Trap

Invoked when `[[Transfer]]()` is called on a Proxy exotic object.

**Signature:**

```js
handler.transfer(target)
```

- `target` -- the original target object of the Proxy.

No `receiver` parameter -- unlike `get`/`set`, transfer has no prototype
chain delegation. There is no meaningful "receiver" distinct from the target.

**Returns:** An Object (the transferred result).

**Steps:**

1. Let `handler` be `[[ProxyHandler]]`.
2. If `handler` is `null`, throw `TypeError` (revoked Proxy).
3. Let `trap` be `? GetMethod(handler, "transfer")`.
4. If `trap` is `undefined`, return `? target.[[Transfer]]()`.
5. Let `trapResult` be `? Call(trap, handler, « target »)`.
6. Perform `? ValidateTransferTrapResult(target, trapResult)`.
7. Return `trapResult`.

### No syntax changes

Unlike `using` with `Symbol.dispose` and `await using` with `Symbol.asyncDispose',
the transfer protocol does not require new syntax. `Object.transfer()` is sufficient to trigger transfer semantics, and the `transfer` trap allows Proxies to intercept without syntax changes.

#### Trap Invariants

The `ValidateTransferTrapResult` operation enforces:

1. **Must return an object.** If the trap returns a non-object, throw a
   `TypeError`.
2. **Non-configurable transferability.** If the target has a non-configurable,
   non-writable `[Symbol.transfer]` property that is `undefined`, the
   trap must throw (cannot fabricate transferability for a non-transferable
    target). The `has` and `get` traps have analogous invariants for
   non-configurable properties.

No additional invariants (e.g., verifying target detachment) are enforced.
The trap must produce a valid result object; what happens to the target is
the trap's responsibility.

#### Use Cases for the Proxy Trap

**Logging and instrumentation:**

```js
function withTransferLogging(obj) {
  return new Proxy(obj, {
    transfer(target) {
      console.log(`Transferring ${target.constructor.name}`);
      return Reflect.transfer(target);
    }
  });
}
```

**Access control:**

```js
function transferGuard(obj, allowTransfer) {
  return new Proxy(obj, {
    transfer(target) {
      if (!allowTransfer()) {
        throw new TypeError('Transfer not permitted at this time');
      }
      return Reflect.transfer(target);
    }
  });
}
```

**Transfer interception for membranes:**

Membranes (security boundaries between object graphs) need to intercept all
MOP operations to wrap/unwrap objects crossing the boundary. Without a
`transfer` trap, transfer would bypass the membrane:

```js
function createMembrane(target) {
  const handler = {
    // ... other traps for get, set, etc.

    transfer(target) {
      const transferred = Reflect.transfer(target);
      // Wrap the transferred object in the membrane too
      return new Proxy(transferred, handler);
    }
  };
  return new Proxy(target, handler);
}
```

**Double transfer through a Proxy:**

`Object.transfer(proxy)` fires the trap, which typically calls
`Reflect.transfer(target)`, detaching the target. A second call throws
because the target is already detached. A trap that does *not* forward
(e.g., virtualized transfer) can allow multiple transfers -- the invariants
do not require target detachment.

**Virtualized transfer (synthetic objects):**

```js
const virtualResource = new Proxy({}, {
  transfer(target) {
    // This proxy doesn't wrap a real transferable --
    // it synthesizes transfer behavior entirely in the trap
    return { value: 'synthesized resource' };
  }
});

const owned = Object.transfer(virtualResource); // works
```

## Built-in Implementations

The following built-in types would implement `[Symbol.transfer]()`.
`Object.transfer()` throws `TypeError` on anything else.

### What Is and Is Not Transferable

| Object type                              | Transferable? | Mechanism                          |
|------------------------------------------|---------------|------------------------------------|
| Array iterators (`.values()`, etc.)      | Yes | `Iterator.prototype[@@transfer]` |
| Map/Set iterators                        | Yes | `Iterator.prototype[@@transfer]` |
| Generator objects                        | Yes | `Iterator.prototype[@@transfer]` |
| Iterator Helpers results (`.map()`, etc.)| Yes | `Iterator.prototype[@@transfer]` |
| Async generator objects                  | Yes | `AsyncIterator.prototype[@@transfer]` |
| `DisposableStack`                        | Yes | `[@@transfer]` delegates to `.move()` |
| `AsyncDisposableStack`                   | Yes | `[@@transfer]` delegates to `.move()` |
| `ArrayBuffer`                            | Yes | `[@@transfer]` delegates to `.transfer()` |
| Plain `{ next() {} }` objects            | No  | No `[Symbol.transfer]` -- throws `TypeError` |
| Arrays, Maps, Sets (the collections)     | No  | They are iterab*les*, not iterators |
| Arbitrary user objects                   | No  | Unless they implement the protocol  |

Plain-object iterators with no shared prototype will not gain transfer
automatically. Custom iterators that need it must implement
`[Symbol.transfer]()`:

```js
class MyIterator {
  #state;
  #detached = false;

  // ... iteration methods ...

  [Symbol.transfer]() {
    if (this.#detached) throw new TypeError('Already detached');
    const iter = new MyIterator(/* transfer state */);
    this.#detached = true;
    return iter;
  }
}
```

### Iterators

#### Iterator.prototype[Symbol.transfer]()

After transfer, the original iterator is detached:

```js
const iter = [1, 2, 3].values();
const owned = Object.transfer(iter);

iter.next();    // throws TypeError: Iterator has been detached
owned.next();   // { value: 1, done: false }
owned.next();   // { value: 2, done: false }
```

##### The `[[Detached]]` Internal Slot

Built-in iterators need a way to distinguish "detached" from "exhausted."
An exhausted iterator returns `{ value: undefined, done: true }` from
`.next()`. A detached iterator throws `TypeError`. These are different
states — an exhausted iterator completed normally; a detached iterator had
its state moved elsewhere.

`ArrayBuffer` does not have an explicit `[[Detached]]` slot — it derives
detached status from `[[ArrayBufferData]]` being `null`. Built-in iterators
could do something similar (e.g., Array iterators could check
`[[IteratedObject]]` being a sentinel value). But iterators have
heterogeneous internal slots across types (`[[IteratedObject]]`,
`[[GeneratorState]]`, `[[AsyncGeneratorState]]`, etc.), so a uniform
`[[Detached]]` boolean slot on each built-in iterator type is the cleaner
approach. When `[[Detached]]` is `true`, `.next()`, `.return()`, `.throw()`,
`[Symbol.iterator]()`, and `[Symbol.transfer]()` all throw `TypeError`.

The new iterator inherits all of the original's iteration-specific internal
slots (e.g., `[[IteratedObject]]`, `[[ArrayIteratorNextIndex]]`). The
original has those slots cleared and `[[Detached]]` set to `true`.

#### AsyncIterator.prototype[Symbol.transfer]()

Same semantics for async iterators (with a `[[Detached]]` slot on async
iterator types):

```js
async function* generate() {
  yield 1; yield 2; yield 3;
}

const iter = generate();
const owned = Object.transfer(iter);

await iter.next();    // throws TypeError: AsyncIterator has been detached
await owned.next();   // { value: 1, done: false }
```

#### Changes to `%IteratorPrototype% [ @@dispose ]`

The existing `%IteratorPrototype%[@@dispose]()` algorithm (ERM §12.1.1.1)
calls `.return()` on the iterator. Because `.return()` throws `TypeError` on
a detached iterator, disposal of a transferred iterator would throw --
violating the rule that cleanup methods no-op after transfer.

This proposal requires modifying the normative definition of
`%IteratorPrototype%[@@dispose]()` to suppress errors from `.return()` when
the iterator has been detached. The modified algorithm:

1. Let _O_ be the **this** value.
2. If _O_ has a `[[Detached]]` slot and _O_.[[Detached]] is **true**, return
   **undefined**.
3. Let _return_ be ? GetMethod(_O_, "return").
4. If _return_ is not **undefined**, then
   a. Perform ? Call(_return_, _O_).
5. Return **undefined**.

The same change applies to `%AsyncIteratorPrototype%[@@asyncDispose]()`.

This ensures that `using` / `await using` bindings that hold a
now-transferred iterator silently no-op at block exit, consistent with
`DisposableStack` and the general post-transfer cleanup contract.

#### Transfer Mid-Iteration

Transfer itself always succeeds immediately -- it detaches the source and
returns a new owner. Consequences depend on timing:

**Sync iterators in a `for...of` loop.** The loop's next `.next()` call
throws `TypeError` (detached). The loop's implicit `try/finally` then calls
`.return()`, which also throws:

```js
const iter = [1, 2, 3].values();
for (const x of iter) {
  if (x === 2) {
    const owned = Object.transfer(iter);
    // Next iteration: iter.next() throws TypeError
    // Loop finally: iter.return() also throws TypeError
    doWork(owned);
  }
}
```

**Async iterators with pending operations.** An async generator may have
an in-flight `.next()` whose promise has not yet settled:

```js
const iter = asyncGen();
const pending = iter.next();  // in-flight
Object.transfer(iter);        // what happens?
```

`[Symbol.transfer]()` should throw `TypeError` if the async generator has
pending operations (`[[AsyncGeneratorState]]` is `"executing"` or the queue
is non-empty). The caller must `await` pending operations before
transferring. The alternative -- implicitly cancelling pending operations
(à la `ReadableStreamDefaultReader.releaseLock()`) -- adds complexity
without clear benefit.

### DisposableStack / AsyncDisposableStack

A `DisposableStack` collects resources for group disposal — whoever holds
the stack is responsible for calling `dispose()`. This is inherently
single-owner. Transfer decouples a stack from a `using` binding without
triggering disposal, enabling a setup-or-rollback pattern:

```js
function setupResources() {
  using stack = new DisposableStack();
  stack.use(openFile('a.txt'));
  stack.use(openFile('b.txt'));
  doValidation(); // if this throws, stack disposes both files
  return Object.transfer(stack);
  // stack is detached -- the `using` cleanup at block exit is a no-op
  // caller owns all registered resources via the returned stack
  // Equivalent to calling stack.move() but using the generalized protocol
}

{
  using owned = setupResources();
  // ... use resources ...
  // owned[Symbol.dispose]() disposes both files at block exit
}
```

`DisposableStack` already has a `.move()` method that does exactly this —
`[Symbol.transfer]()` delegates to it, the same way
`ArrayBuffer.prototype[Symbol.transfer]()` delegates to `.transfer()`.
No additional `[[Detached]]` slot or machinery is needed.

The `.move()` algorithm (spec §12.3.3.6):

1. Check `[[DisposableState]]` — if already `disposed`, throw `ReferenceError`.
2. Create a new `DisposableStack` with `[[DisposableState]]` set to `pending`.
3. Move `[[DisposeCapability]]` (the resource list) from the original to the
   new stack.
4. Replace the original's `[[DisposeCapability]]` with an empty one.
5. Set the original's `[[DisposableState]]` to `disposed`.
6. Return the new stack.

After `.move()`, the original stack is in the `disposed` state. Existing
methods already check this: `.use()`, `.adopt()`, `.defer()`, and `.move()`
throw `ReferenceError`; `.dispose()` / `[Symbol.dispose]()` returns
`undefined` (no-op). This matches the transfer contract (operational methods
throw, cleanup no-ops) with no changes to `DisposableStack` itself.

`AsyncDisposableStack.prototype.move()` (spec §12.4.3.6) is identical in
structure, using `[[AsyncDisposableState]]` instead.

One wrinkle: `.move()` throws **`ReferenceError`** on disposed stacks, while
detached iterators throw **`TypeError`**. `[Symbol.transfer]()` could either
preserve `.move()`'s error type (simple delegation) or normalize to
`TypeError` (consistency with iterators and the protocol's own validation
errors). The current proposal defers to `.move()`'s existing behavior.

### Post-Transfer Behavior

The general rule: operational methods throw, cleanup methods no-op.

Iterators are an exception in that all methods (including `.return()`) throw,
since cleanup responsibility has transferred to the new owner and a silent no-op
could mask ownership bugs.

#### Iterators

All methods on a detached iterator throw `TypeError`: `.next()`, `.return()`,
`.throw()`, `[Symbol.iterator]()`, `[Symbol.asyncIterator]()`.

`[Symbol.iterator]()` must also throw -- otherwise `for...of` would silently
enter iteration on a detached iterator before `.next()` fails:

```js
const iter = [1, 2, 3].values();
const owned = Object.transfer(iter);

// Without the guard, this would enter the loop body and only
// throw on the first .next() call. With the guard, it throws
// immediately at the [Symbol.iterator]() call.
for (const x of iter) { /* ... */ } // throws TypeError
```

#### DisposableStack / AsyncDisposableStack

| Method               | Behavior after transfer |
|----------------------|-------------------------|
| `.use()`             | Throws `ReferenceError` |
| `.adopt()`           | Throws `ReferenceError` |
| `.defer()`           | Throws `ReferenceError` |
| `.move()`            | Throws `ReferenceError` |
| `[Symbol.dispose]()` | No-op (returns `undefined`) |
| `[Symbol.asyncDispose]()` | No-op             |
| `.disposed`          | `true`                  |

These are the *existing* behaviors of a disposed `DisposableStack` — no
spec changes are needed for these types. Adding resources to a disposed
stack is an error. Disposing a disposed stack is a no-op — the resources
have been moved, there is nothing to clean up.

#### ArrayBuffer

Accessing a detached buffer's contents throws `TypeError`.

#### Interaction with Explicit Resource Management (`using`)

If the original is held by `using` / `await using`, dispose **will still be
called** at block exit. A detached object **must not throw** from dispose --
it must be a silent no-op:

```js
async function example(source) {
  await using iter = source[Symbol.asyncIterator]();

  const owned = Object.transfer(iter);
  // iter is now detached

  doWork(owned);

  // At block exit, iter[Symbol.asyncDispose]() is called.
  // iter is detached, but this MUST NOT throw.
  // It should silently do nothing -- there is nothing to clean up.
}
```

The detach guard applies to **operational methods** (`.next()`, `.return()`,
`.throw()`) -- not to disposal. For iterators, where `[Symbol.dispose]()` /
`[Symbol.asyncDispose]()` delegates to `.return()`, the dispose
implementation must catch the `TypeError` from a detached `.return()` and
suppress it, preserving the no-op invariant. Pattern for user-defined types:

```js
class MyResource {
  #detached = false;

  [Symbol.transfer]() {
    // ... transfer logic, sets #detached = true ...
  }

  doWork() {
    if (this.#detached) throw new TypeError('Resource has been detached');
    // ...
  }

  [Symbol.dispose]() {
    if (this.#detached) return; // no-op, not an error
    // ... actual cleanup ...
  }
}
```

### Edge Cases

#### `Symbol.transfer` on `Object.prototype`

If `Symbol.transfer` is defined on `Object.prototype`, every object becomes
transferable:

```js
Object.prototype[Symbol.transfer] = function() { return {}; };
Object.transfer({});  // succeeds -- any object is now "transferable"
```

This is the same class of prototype pollution that affects `then` on
`Object.prototype` (see [proposal-thenable-curtailment][curtailment]).
Following the model proposed there, `Object.prototype` would exotically
reject `Symbol.transfer` as an own property -- `Object.defineProperty` and
`[[Set]]` silently no-op when the key is `Symbol.transfer` and the target
is `Object.prototype`. This prevents prototype pollution without affecting
any other object.

[curtailment]: https://github.com/tc39/proposal-thenable-curtailment

#### Partial Detach on Exception

A user-defined `[Symbol.transfer]()` may throw after partially clearing
internal state, leaving the object in an inconsistent state:

```js
[Symbol.transfer]() {
  this.#connectionA = undefined;  // cleared
  this.#connectionB.validate();   // throws!
  // #connectionA is gone, #connectionB still exists
  // object is partially detached
}
```

The `[[Transfer]]` algorithm uses `? Call()`, so the exception propagates
to the caller. The protocol does not attempt to roll back partial state changes.
Implementations should structure `[Symbol.transfer]()` to perform all
fallible operations before mutating state when possible.

#### Re-entrancy

If `[Symbol.transfer]` is defined as a getter, the getter may call
`Object.transfer()` on the same object, causing re-entrant invocation:

```js
const obj = {
  get [Symbol.transfer]() {
    Object.transfer(this);  // re-entrant!
    return () => ({});
  }
};
```

The behavior is not explicitly constrained. The inner `Object.transfer()`
call will invoke `GetMethod`, which triggers the getter again, leading to
infinite recursion and a stack overflow. For built-in types, re-entrancy is
not possible. For user-defined types, this is the implementor's problem.

#### Frozen and Sealed Objects

`Object.transfer()` and `Reflect.transfer()` throw `TypeError` on
non-extensible (frozen or sealed) objects. Transfer changes an object's
observable behavior from functional to throwing, which contradicts the
immutability guarantee of `Object.freeze()`:

```js
const iter = [1, 2, 3].values();
Object.freeze(iter);
Object.transfer(iter);  // throws TypeError: cannot transfer frozen object
```

This is enforced by the `[[Transfer]]` algorithm's `IsExtensible` check
(step 1). A direct `obj[Symbol.transfer]()` call bypasses this check --
the protocol cannot prevent it, but `Object.transfer()` can.

This creates an inconsistency with `ArrayBuffer`: `buf.transfer()` succeeds
on frozen buffers (the `ArrayBufferCopyAndDetach` algorithm has no
`IsExtensible` check), but `Object.transfer(buf)` would throw. See
Anticipated Objections for discussion of the options.

## Use Cases

### Use Case 1: Concurrent Safety

When async iterators are passed between functions, nothing enforces
single-consumer access:

```js
async function processStream(source) {
  const owned = Object.transfer(source);
  // Any further use of `source` throws TypeError
  for await (const chunk of owned) {
    process(chunk);
  }
}
```

### Use Case 2: Ownership in APIs

APIs that accept stateful objects often implicitly expect ownership:

```js
class DataProcessor {
  #source;

  constructor(source) {
    this.#source = Object.transfer(source); // enforce the contract
  }

  process() {
    for (const item of this.#source) {
      // safe -- no other consumer can interfere
    }
  }
}
```

### Use Case 3: Resource Management

With [Explicit Resource Management][erm] (`using` / `await using`), transfer
moves cleanup responsibility with ownership:

```js
async function handleConnection(stream) {
  await using owned = Object.transfer(stream);
  // `stream` is detached -- only `owned` can be consumed or disposed
  for await (const msg of owned) {
    respond(msg);
  }
  // owned[Symbol.asyncDispose]() is called at block exit
}
```

Without transfer, it is ambiguous who is responsible for calling `.return()`.

[erm]: https://github.com/tc39/proposal-explicit-resource-management

### Use Case 4: `ReadableStream.from()` and Unprotected Iterators

`ReadableStream.from(iter)` wraps an async iterator in a `ReadableStream`.
The stream has locking — `getReader()` ensures single-consumer access to
the stream. But the underlying iterator has no such protection:

```js
async function* generateData() {
  yield 'chunk1';
  yield 'chunk2';
  yield 'chunk3';
}

const iter = generateData();
const stream = ReadableStream.from(iter);

// The stream can be locked to a single reader...
const reader = stream.getReader();

// ...but nothing prevents consuming the iterator directly,
// bypassing the stream entirely and corrupting both consumers:
await iter.next(); // silently steals 'chunk1' from the stream
```

The stream's locking mechanism protects the *stream*, not the *iterator
underneath it*. Any code that retains a reference to `iter` can call
`.next()` and pull values out from under the stream, causing data loss or
interleaving. The stream has no way to know this happened.

With transfer, `ReadableStream.from()` could take ownership of the
iterator, preventing this class of bug:

```js
const iter = generateData();
const stream = ReadableStream.from(Object.transfer(iter));
// iter is now detached -- any direct .next() call throws TypeError

await iter.next(); // throws TypeError: Iterator has been detached

// The stream is the sole consumer of the iterator's data
const reader = stream.getReader();
const { value } = await reader.read(); // 'chunk1' -- guaranteed
```

The same pattern applies to any API that wraps an iterator for internal
consumption. Without transfer, the API must trust that callers discard
their reference to the iterator. With transfer, the API can enforce it.

### Use Case 5: User-Defined Transferable Resources

**Implementors enforce detach semantics** -- the engine does not (see above).

Correct:

```js
class DatabaseCursor {
  #connection;
  #query;
  #detached = false;

  constructor(connection, query) {
    this.#connection = connection;
    this.#query = query;
  }

  get detached() { return this.#detached; }

  fetch() {
    if (this.#detached) throw new TypeError('Cursor has been detached');
    return this.#connection.fetchNext(this.#query);
  }

  [Symbol.transfer]() {
    if (this.#detached) throw new TypeError('Cannot transfer a detached cursor');
    const cursor = new DatabaseCursor(this.#connection, this.#query);
    this.#connection = undefined;
    this.#query = undefined;
    this.#detached = true;
    return cursor;
  }
}

// API that takes ownership
function processCursor(cursor) {
  const owned = Object.transfer(cursor);
  while (true) {
    const row = owned.fetch();
    if (!row) break;
    handle(row);
  }
}
```

Incorrect -- the engine will not catch this:

```js
class BrokenResource {
  #data;

  constructor(data) {
    this.#data = data;
  }

  use() { return this.#data; }

  [Symbol.transfer]() {
    // BUG: returns a new object with the moved data, but does NOT
    // detach the original. The original can still access #data.
    return new BrokenResource(this.#data);
    // Missing: this.#data = undefined;
  }
}

const a = new BrokenResource([1, 2, 3]);
const b = Object.transfer(a);
a.use();  // [1, 2, 3] -- still accessible! Protocol violated, but no engine error.
b.use();  // [1, 2, 3] -- both hold a reference to the same data.
```

The engine validates mechanics (callable, returns object) but not semantics
(original rendered unusable).

### Use Case 6: Cross-Realm Iterator Transfer

With `Symbol.transfer` as a common protocol, iterators can participate in
`postMessage()` transfer lists -- moving an iterator across worker boundaries
using a `MessagePort` pair, the same way `ReadableStream` transfer works
today. A sync `Iterator` becomes an `AsyncIterator` on the receiving side
(since `MessagePort` communication is inherently asynchronous). See
[CROSS-REALM-ITERATOR-TRANSFER.md](./CROSS-REALM-ITERATOR-TRANSFER.md) for
the full design.

### Use Case 7: Proxy Membranes and Security Boundaries

Without a `transfer` trap, transfer bypasses Proxy membranes:

```js
function createMembrane(wetTarget) {
  const wet2dry = new WeakMap();
  const dry2wet = new WeakMap();

  function wrap(wetObj) {
    if (wet2dry.has(wetObj)) return wet2dry.get(wetObj);
    const dryProxy = new Proxy(wetObj, {
      get(target, prop, receiver) {
        const val = Reflect.get(target, prop);
        return typeof val === 'object' && val !== null ? wrap(val) : val;
      },
      transfer(target) {
        const wetTransferred = Reflect.transfer(target);
        return wrap(wetTransferred); // keep it inside the membrane
      },
      // ... other traps
    });
    wet2dry.set(wetObj, dryProxy);
    dry2wet.set(dryProxy, wetObj);
    return dryProxy;
  }

  return wrap(wetTarget);
}
```

## Prior Art

- **`ArrayBuffer.prototype.transfer()`** (ES2024) -- ownership transfer for
  binary data buffers, with detach semantics.
- **Web platform `Transferable` interface** -- `postMessage()` supports
  transferring `ArrayBuffer`, `MessagePort`, `ReadableStream`, etc. between
  contexts, detaching the original.
- **Proxy handler traps** -- every fundamental MOP operation (`[[Get]]`,
  `[[Set]]`, `[[Delete]]`, `[[Call]]`, etc.) has a corresponding Proxy trap
  and `Reflect` method. This proposal extends that pattern to transfer.
- **Rust's ownership model** -- move semantics ensure single ownership of
  resources at compile time.
- **`ReadableStream` locking** -- prevents concurrent access to a stream
  by locking it to a single reader.
- **C++ `std::move`** -- casts to an rvalue reference, enabling move
  constructors/assignment operators to transfer resources. The source is
  left in a valid but unspecified state.
- **`Symbol.iterator` / `Symbol.dispose`** -- precedent for Symbol-based
  protocols in the language for iteration and resource management.
- **Piscina `Transferable` interface** -- the [Piscina][piscina] worker pool
  defines a symbol-based transfer protocol: objects implement
  `[Piscina.transferableSymbol]` (returns the transfer list) and
  `[Piscina.valueSymbol]` (returns the value to transmit) to opt into
  `postMessage` transfer. This is a userland precursor to `Symbol.transfer`,
  limited to cross-thread transfer.
- **Comlink `transfer()` / `releaseProxy()`** -- [Comlink][comlink] wraps
  `postMessage` transfer with `Comlink.transfer(value, transferables)` and
  provides `proxy[Comlink.releaseProxy]()` which sets a flag and throws on
  subsequent access — a manual same-realm invalidation pattern.

[piscina]: https://github.com/piscinajs/piscina
[comlink]: https://github.com/GoogleChromeLabs/comlink

## Appendix: Web Platform Integration

Not part of the TC39 proposal. Discusses potential web platform integration.

### Relationship to structuredClone and postMessage

The web platform has [`Transferable`][transferable] objects via `postMessage()`
/ `structuredClone()`. `Symbol.transfer` is complementary but distinct:

| Aspect                  | Web platform `Transferable`             | `Symbol.transfer`                    |
|-------------------------|-----------------------------------------|------------------------------------------|
| **Scope**               | Cross-realm (workers, iframes)          | Same-realm (in-process ownership move)   |
| **Mechanism**           | Structured clone algorithm              | `[[Transfer]]` internal method           |
| **Defined by**          | WHATWG HTML spec                        | ECMAScript (this proposal)               |
| **What moves**          | Underlying data across thread boundary  | Logical ownership within same thread     |
| **Objects supported**   | Hardcoded list in the spec              | Any object implementing the protocol     |
| **User-extensible**     | No                                      | Yes                                      |

`postMessage()` moves data across realms; `Object.transfer()` moves
ownership within the same realm. Both detach the source.

A possible WHATWG/W3C follow-on: let the structured clone algorithm recognize
`Symbol.transfer` for user-defined objects in `postMessage()` transfer lists:

```js
// Hypothetical future integration
const cursor = new DatabaseCursor(connection, query);
worker.postMessage({ cursor }, { transfer: [cursor] });
// cursor is detached in this realm; worker receives a transferred copy
```

[transferable]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects

### Web API Candidates for Symbol.transfer

#### Strong Candidates

Already transferable via `postMessage()` or already have locking semantics:

**ReadableStream** -- Has `getReader()` locking. Already `postMessage()`
transferable. Same-realm transfer complements locking with full ownership move.

```js
const stream = new ReadableStream(/* ... */);
const owned = Object.transfer(stream);
// stream is detached; owned is the sole consumer
// No need to acquire a reader lock -- ownership is exclusive
```

**WritableStream** -- Same locking model via `getWriter()`.

**TransformStream** -- Readable + writable sides, both locked. Transfer
moves both ends atomically.

**MessagePort** -- Already `postMessage()` transferable. Single-consumer.

**ImageBitmap** -- Already `postMessage()` transferable. Has `.close()`.

**OffscreenCanvas** -- Already `postMessage()` transferable. Single context.

**VideoFrame / AudioData** (WebCodecs) -- Already `postMessage()`
transferable. Have `.close()` for lifetime management.

#### Moderate Candidates

Single-consumer or exclusive-access, but not currently `postMessage()`
transferable:

**FileSystemFileHandle / FileSystemDirectoryHandle** -- Represents file/dir
access. Transfer useful for handing off to another component.

**FileSystemWritableFileStream** -- Single-writer file I/O.

**ReadableStreamDefaultReader / ReadableStreamBYOBReader** -- Represent the
read lock on a `ReadableStream`. Transferring a reader would be equivalent to
releasing the lock and acquiring a new one from the stream:

```js
const stream = new ReadableStream(/* ... */);
const reader = stream.getReader();

// Transfer is equivalent to:
//   reader.releaseLock();
//   const newReader = stream.getReader();
// but expressed as a single ownership move.
const newReader = Object.transfer(reader);

// reader is detached -- cannot read or release
// newReader holds the lock
```

This has observable side effects: `releaseLock()` cancels any pending
`reader.read()` calls, resolving them with `{ value: undefined, done: true }`.
A `[Symbol.transfer]()` implementation would need the same behavior --
transferring a reader with in-flight reads cancels them.

**WritableStreamDefaultWriter** -- Same locking model as the readers above,
via `WritableStream.getWriter()`. Transfer would release and reacquire the
write lock with the same cancellation semantics for pending `writer.write()`
and `writer.close()` calls.

**IDBCursor** -- Stateful sequential-access cursor tied to a transaction.

**RTCDataChannel** -- Single `onmessage` handler, though event-based
delivery is less natural for transfer than streams.

#### Poor Fit

Multi-consumer by design:

- **AbortController / AbortSignal** -- Multiple listeners, broadcast model.
- **EventTarget** -- Multi-consumer event dispatch.
- **BroadcastChannel** -- Multi-consumer messaging.
- **WebSocket** -- Event-based delivery with multiple listeners.
- **MediaStreamTrack** -- Clonable, multiple simultaneous sinks.
- **Request / Response** -- Body is single-use, but `.clone()` exists and
  objects are not typically transferred between owners.

### Pattern: Same-Realm vs. Cross-Realm Transfer

The combination of `Symbol.transfer` (same-realm) and `postMessage()`
transfer (cross-realm) forms a two-level ownership model:

```
Same realm                            Cross realm
─────────────                         ─────────────
Object.transfer(obj)                  worker.postMessage(msg, { transfer: [obj] })
  │                                     │
  ├─ Invokes [[Transfer]]               ├─ Structured clone algorithm
  ├─ Calls [Symbol.transfer]()          ├─ Checks [[Transferable]] internal
  ├─ Detaches source                    ├─ Detaches source
  └─ Returns new owner (same realm)     └─ Reconstructs in target realm
```

Both detach the source. The difference is whether the new owner lives in
the same realm or a different one.

## Open Questions

1. **`ReferenceError` vs `TypeError` on post-transfer access?**
   `DisposableStack` methods throw `ReferenceError` on disposed stacks (the
   existing spec behavior). Detached iterators would throw `TypeError` (new
   behavior). Detached `ArrayBuffer` access also throws `TypeError`. Should
   `DisposableStack`'s `[Symbol.transfer]()` normalize to `TypeError` for
   consistency with the rest of the protocol, or preserve the existing
   `ReferenceError` by delegating directly to `.move()`? The simpler path
   is direct delegation (preserving `ReferenceError`), but the inconsistency
   between "transferred stack throws ReferenceError" and "transferred
   iterator throws TypeError" may confuse users.

2. **`Iterator.from()` as a bridge for non-transferable iterators?**
   `Iterator.from(plain)` wraps a duck-typed iterator in an object that
   inherits `[Symbol.transfer]()` from `Iterator.prototype`:
   ```js
   const plain = { i: 0, next() { return { value: this.i++, done: this.i > 3 }; } };
   Object.transfer(plain);                // TypeError: not transferable
   Object.transfer(Iterator.from(plain)); // OK -- wrapper is transferable
   ```
   Problem: transferring the wrapper detaches the wrapper but the underlying
   plain object remains accessible. **Leaning no** -- this leaky abstraction
   undermines the ownership guarantee.

## Anticipated Objections

### "Too much machinery for the problem"

The proposal adds five new language surface areas: a well-known symbol, an
internal method, two static methods, and a Proxy trap. The iterator
interleaving bug from the motivation is real but may be uncommon in
practice. The MOP integration (Proxy trap, `Reflect` method) is what
distinguishes this from a userland convention — without it,
`Object.transfer()` is just a utility function calling a symbol.

### "Why not userland?"

A library could define a symbol, a transfer function, and a convention.
The things userland *cannot* do: (1) add `[Symbol.transfer]()` to built-in
iterator prototypes, (2) add a Proxy trap, (3) add a `[[Detached]]`
internal slot to built-in iterators. If the committee doesn't find built-in
iterator transfer compelling, the proposal loses its primary justification
for being in the spec.

### "The Proxy trap is premature"

Adding a new MOP internal method is a significant spec change. Every engine
must implement it. Every exotic object type must account for it (even if the
default is "delegate to ordinary"). The argument that Proxy's `get` trap
can't distinguish a transfer-related symbol lookup from a regular property
access is valid — but someone will ask whether anyone actually needs to
distinguish them. If the Proxy trap is dropped, the proposal shrinks to
`Symbol.transfer` + `Object.transfer()` with no MOP changes — much smaller,
much easier to accept, but loses the membrane use case.

### "`[[Transfer]]` is not essential"

`[[Get]]`, `[[Set]]`, `[[Delete]]` are operations the entire language
depends on. `[[Transfer]]` is used by most objects never. Adding it to the
essential internal methods table elevates it to the same status as property
access. `[[Call]]` and `[[Construct]]` are essential internal methods that most
objects don't have either.

### "The frozen/sealed check is inconsistent with ArrayBuffer"

`ArrayBuffer.prototype.transfer()` works on frozen ArrayBuffers today.
The `ArrayBufferCopyAndDetach` abstract operation checks
`RequireInternalSlot`, `IsSharedArrayBuffer`, `IsDetachedBuffer`, and
`[[ArrayBufferDetachKey]]` — but never `IsExtensible`. `Object.freeze()`
prevents property changes but does not prevent internal slot mutation
(`[[ArrayBufferData]]` is set to `null` during detach). So:

```js
const buf = new ArrayBuffer(8);
Object.freeze(buf);
buf.transfer();          // succeeds — internal slots are not properties
Object.transfer(buf);    // throws TypeError — IsExtensible check fails
```

The `IsExtensible` check in `[[Transfer]]` step 1 creates this
inconsistency. Three options: (1) drop the check, allowing frozen objects
to be transferred via `Object.transfer()` — transfer mutates internal
state, not properties, so `Object.freeze()` is arguably irrelevant;
(2) keep the check and accept the inconsistency — `Object.transfer()` is
stricter than the raw `.transfer()` call, which is a reasonable posture;
(3) propose updating `ArrayBuffer.prototype.transfer()` to also reject
frozen buffers — but this is a breaking change to ES2024 behavior.

### "Transfer doesn't actually solve the safety problem"

Transfer eliminates one reference (the source) but doesn't prevent the
caller from passing the result to multiple consumers. The "concurrent
safety" use case requires every API that consumes an iterator to adopt the
`Object.transfer()` pattern. Most won't, so the ecosystem benefit is
incremental.

### "What about `Symbol.clone` / structured clone?"

Clone and transfer are different operations, but the question will come up.
WHATWG may want a concrete `structuredClone` integration story — saying
"out of scope" may not satisfy if they see this as the path to
user-extensible structured clone.

### "The `Object.prototype` exotic rejection depends on proposal-thenable-curtailment"

That proposal is Stage 1 with an unsettled mechanism. Depending on it ties
this proposal's outcome to another proposal's progress. If thenable
curtailment stalls or changes approach, this proposal needs a different
answer to prototype pollution. Making `Object.prototype` increasingly
exotic also sets a precedent — every new well-known symbol with pollution
concerns could justify another exotic rejection.

### "The `using` interaction is a footgun"

```js
await using iter = source[Symbol.asyncIterator]();
const owned = Object.transfer(iter);
doWork(owned); // if this throws, owned is not disposed -- resource leak
```

The safe pattern requires two `using` bindings for one logical resource:

```js
await using iter = source[Symbol.asyncIterator]();
await using owned = Object.transfer(iter);
```

This is arguably confusing and error-prone compared to not transferring
at all.

### "Generator transfer is non-trivial engine work"

Generator objects have complex internal state (`[[GeneratorState]]`,
`[[GeneratorContext]]` including the suspended execution context and stack
frame). Moving this to a new object is implementable but non-trivial.
Async generators are harder — there's a job queue involved.

### "This overlaps with the Structs / shared structs proposal"

The shared structs proposal addresses cross-thread data sharing with its
own ownership semantics. If that proposal lands, there's potential overlap
or conflict. The structs champion group will want to understand the
relationship.

### "Just use ReadableStream"

`ReadableStream` already has locking, single-consumer semantics, and is
transferable via `postMessage()`. If the primary motivation is safe
single-consumer iteration, wrapping an iterator in a `ReadableStream`
solves it today. `ReadableStream` is now available in Node.js, Deno, and Bun, but may
be absent in embedded engines — and it's heavy for simple iteration.

### "Reduced proposal without MOP changes"

If the Proxy trap and `[[Transfer]]` internal method face strong resistance,
a fallback is: `Symbol.transfer` + `Object.transfer()` only. No Proxy trap,
no `Reflect.transfer()`, no new internal method. In this reduced version,
`Object.transfer()` would look up and call the symbol via ordinary
`[[Get]]` + `[[Call]]` (with return-value validation), rather than routing
through a dedicated `[[Transfer]]` internal method. This loses Proxy
interceptability but covers the primary use cases (iterators, user-defined
resources, `ArrayBuffer` unification) with far less spec complexity.

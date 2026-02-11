---
theme: default
title: "Transfer Protocol"
info: |
  ## TC39 Proposal: Transfer Protocol
  A generalized ownership-transfer protocol for JavaScript.

  Stage 0
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
---

# Transfer Protocol

A generalized ownership-transfer protocol for JavaScript

<div class="pt-12">
  <span class="px-2 py-1 rounded text-sm" style="background: #f0f0f0; color: #333;">
    Stage 0
  </span>
</div>

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/nicolo-ribaudo/proposal-transfer-protocol" target="_blank" class="text-xl slidev-icon-btn">
    <carbon-logo-github />
  </a>
</div>

---
transition: slide-left
---

# The Problem

Ownership transfer exists in the platform -- but there's no common protocol

<v-clicks>

- **`ArrayBuffer.prototype.transfer()`** -- moves buffer data, detaches the original (ES2024)
- **`DisposableStack.prototype.move()`** -- moves resources to a new stack, marks original disposed
- **`ReadableStream` / `WritableStream` locking** -- `getReader()` locks a stream to a single consumer
- **`postMessage()` transfer lists** -- `ArrayBuffer`, `MessagePort`, `OffscreenCanvas`, etc.
- **Runtime-specific types** -- Node.js adds `FileHandle` as a transferable type (detached on send)

</v-clicks>

<div v-click class="mt-6 p-4 rounded" style="background: #fff3cd; color: #856404;">
  Same problem -- exclusive ownership of a stateful resource -- but no shared protocol. No way to write generic transfer code. No way for user-defined types to participate.
</div>

---
transition: slide-left
---

# Motivating Example: Iterators

Iterators are stateful and single-consumer, but nothing enforces it

```js {all|1|3-5|none}
const iter = [1, 2, 3, 4].values();

// Two consumers unknowingly share the same iterator
consumeEvens(iter);
consumeOdds(iter);
// Each consumer sees an unpredictable subset of values
```

<v-click>

Same for async iterators:

```js
async function processStream(stream) {
  const iter = stream[Symbol.asyncIterator]();
  pipeline1(iter);  // starts consuming
  pipeline2(iter);  // races with pipeline1 -- data split unpredictably
}
```

</v-click>

<div v-click class="mt-4 p-3 rounded" style="background: #f8d7da; color: #721c24;">
  Nothing in the language prevents this. Transfer makes ownership explicit.
</div>

---
layout: section
---

# The Proposal

Four pieces that generalize ownership transfer

---
transition: slide-left
---

# Piece 1: `Symbol.transfer`

A new well-known symbol for opting into transfer

An object implements the protocol by providing a `[Symbol.transfer]()` method that:

<v-clicks>

1. Returns a **new object** that takes ownership of the receiver's state
2. **Detaches** the receiver permanently

</v-clicks>

<div v-click>

```js
class OwnedResource {
  #data;
  #detached = false;

  [Symbol.transfer]() {
    if (this.#detached) throw new TypeError('Cannot transfer a detached resource');
    const data = this.#data;
    this.#data = undefined;
    this.#detached = true;
    return new OwnedResource(data);
  }

  [Symbol.dispose]() {
    if (this.#detached) return; // no-op -- nothing to clean up
  }
}
```

</div>

---
transition: slide-left
---

# Piece 2: `Object.transfer(obj)`

Triggers `[[Transfer]]` on the argument

```js
const a = new OwnedResource({ value: 42 });
const b = Object.transfer(a);

a.use();   // throws TypeError: Resource has been detached
b.use();   // { value: 42 }
```

<v-click>

**Why not call `[Symbol.transfer]()` directly?**

`Object.transfer()` routes through `[[Transfer]]`, which provides:

- Argument type checking (must be an object)
- Protocol validation (`[Symbol.transfer]` must exist, be callable, return object)
- Proxy `transfer` trap dispatch
- Frozen/sealed object rejection

</v-click>

<v-click>

Direct `obj[Symbol.transfer]()` bypasses all of this -- available for perf-sensitive paths where the caller has already validated.

</v-click>

---
transition: slide-left
---

# Piece 3: `Reflect.transfer(target)`

The low-level MOP counterpart for Proxy trap forwarding

| Operation    | `Object` method         | `Reflect` method           | Proxy trap       |
|-------------|-------------------------|----------------------------|------------------|
| Get         | --                      | `Reflect.get()`            | `get`            |
| Set         | --                      | `Reflect.set()`            | `set`            |
| Delete      | --                      | `Reflect.deleteProperty()` | `deleteProperty` |
| **Transfer** | **`Object.transfer()`** | **`Reflect.transfer()`**   | **`transfer`**   |

<v-click>

```js
const proxy = new Proxy(target, {
  transfer(target) {
    console.log('transfer intercepted');
    return Reflect.transfer(target); // forward to default behavior
  }
});
```

</v-click>

---
transition: slide-left
---

# Piece 4: Proxy `transfer` Trap

Intercepts `[[Transfer]]()` on Proxy exotic objects

<div class="grid grid-cols-2 gap-4">
<div>

**Logging:**

```js
new Proxy(obj, {
  transfer(target) {
    console.log('Transferring...');
    return Reflect.transfer(target);
  }
});
```

</div>
<div>

**Access control:**

```js
new Proxy(obj, {
  transfer(target) {
    if (!allowed())
      throw new TypeError('Not permitted');
    return Reflect.transfer(target);
  }
});
```

</div>
</div>

<v-click>

**Membranes** -- security boundaries that must intercept *all* MOP operations:

```js
transfer(target) {
  return new Proxy(Reflect.transfer(target), handler); // keep inside the membrane
}
```

</v-click>

---
transition: slide-left
---

# The `[[Transfer]]` Internal Method

New essential internal method -- the engine-level mechanism

For **ordinary objects**:

<v-clicks>

1. If `IsExtensible(O)` is **false** -- throw `TypeError` (cannot transfer frozen/sealed)
2. Let `transferFn` = `GetMethod(O, @@transfer)` -- throws if present but not callable
3. If `transferFn` is **undefined** -- throw `TypeError` (not transferable)
4. Let `result` = `Call(transferFn, O)`
5. If `result` is not an Object -- throw `TypeError`
6. Return `result`

</v-clicks>

<div v-click class="mt-4 p-3 rounded" style="background: #d4edda; color: #155724;">
  For <strong>Proxy exotic objects</strong>, <code>[[Transfer]]()</code> is intercepted by the <code>transfer</code> handler trap instead.
</div>

---
transition: slide-left
---

# Built-in Transferables

What gets `[Symbol.transfer]()` out of the box

| Object type | Transferable? | Mechanism |
|-------------|:---:|-----------|
| Array, Map/Set, Generator, Iterator Helper iterators | Yes | `Iterator.prototype[@@transfer]` |
| Async generators | Yes | `AsyncIterator.prototype[@@transfer]` |
| `DisposableStack` / `AsyncDisposableStack` | Yes | Delegates to `.move()` |
| `ArrayBuffer` | Yes | Delegates to `.transfer()` |
| Plain `{ next() {} }` objects | **No** | No `[Symbol.transfer]` |
| Arrays, Maps, Sets (the collections) | **No** | Iterab*les*, not iterators |

---
transition: slide-left
---

# Iterator Transfer in Action

After transfer, the original is detached

```js {all|1-2|4-5|6-7|all}
const iter = [1, 2, 3].values();
const owned = Object.transfer(iter);

iter.next();    // throws TypeError: Iterator has been detached
owned.next();   // { value: 1, done: false }
owned.next();   // { value: 2, done: false }
owned.next();   // { value: 3, done: false }
```

<v-click>

Async iterators work the same way:

```js
async function* generate() { yield 1; yield 2; yield 3; }

const iter = generate();
const owned = Object.transfer(iter);

await iter.next();    // throws TypeError: AsyncIterator has been detached
await owned.next();   // { value: 1, done: false }
```

</v-click>

---
transition: slide-left
---

# Design Principles

<v-clicks>

- **Shape, not behavior** -- spec defines *when* the method is called, not *what* it does. Engine validates mechanics (callable, returns object) not semantics (source detached).
- **Enables but does not enforce exclusive ownership** -- source is rendered unusable, but caller can still share the result. Not sharing is the caller's responsibility.
- **Operational methods throw, cleanup methods no-op** -- `.next()`, `.read()`, etc. throw `TypeError` after transfer. `[Symbol.dispose]()` must silently no-op for safe `using` interaction.
- **Opt-in, not universal** -- only objects implementing `[Symbol.transfer]()` are transferable. Without the symbol, `Object.transfer()` throws `TypeError`.
- **Full MOP integration** -- `[[Transfer]]` internal method, `Object.transfer()`, `Reflect.transfer()`, Proxy `transfer` trap. Every fundamental operation has this triad.
- **Frozen/sealed objects are non-transferable** -- transfer mutates observable behavior, contradicting `Object.freeze()`. Enforced by `[[Transfer]]`; direct `obj[Symbol.transfer]()` bypasses this.

</v-clicks>

---
transition: slide-left
---

# Interaction with `using`

Detached objects must not throw from dispose

```js {all|2|4-5|8-9|all}
async function example(source) {
  await using iter = source[Symbol.asyncIterator]();

  const owned = Object.transfer(iter);
  // iter is now detached

  doWork(owned);

  // At block exit, iter[Symbol.asyncDispose]() is called.
  // iter is detached, but this MUST NOT throw -- silent no-op.
}
```

<v-click>

The **correct** pattern for both bindings:

```js
await using iter = source[Symbol.asyncIterator]();
await using owned = Object.transfer(iter);
// now `owned` will be disposed at block exit
```

</v-click>

---
transition: slide-left
---

# Use Case: Concurrent Safety

Make ownership explicit when passing async iterators between functions

```js
async function processStream(source) {
  const owned = Object.transfer(source);
  // Any further use of `source` throws TypeError
  for await (const chunk of owned) {
    process(chunk);
  }
}
```

---
transition: slide-left
---

# Use Case: `ReadableStream.from()` and Unprotected Iterators

The stream is locked -- but the iterator underneath it isn't

```js
const iter = generateData(); // async generator
const stream = ReadableStream.from(iter);
const reader = stream.getReader(); // stream is locked

// ...but nothing prevents consuming the iterator directly:
await iter.next(); // silently steals data from the stream
```

<v-click>

With transfer, `ReadableStream.from()` takes ownership of the iterator:

```js
const iter = generateData();
const stream = ReadableStream.from(Object.transfer(iter));

await iter.next();             // throws TypeError: Iterator has been detached
await reader.read();           // 'chunk1' -- guaranteed
```

</v-click>

<div v-click class="mt-4 p-2 rounded text-sm" style="background: #fff3cd; color: #856404;">
  Applies to any API that wraps an iterator internally. Without transfer, the API must <em>trust</em> callers discard their reference. With transfer, it <em>enforces</em> it.
</div>

---
transition: slide-left
---

# Use Case: Ownership in APIs

APIs that accept stateful objects can enforce the ownership contract

```js
class DataProcessor {
  #source;

  constructor(source) {
    this.#source = Object.transfer(source); // take ownership
  }

  process() {
    for (const item of this.#source) {
      // safe -- no other consumer can interfere
    }
  }
}
```

---
transition: slide-left
---

# Use Case: User-Defined Transferable Resources

Any class can participate in the protocol

```js
class DatabaseCursor {
  #connection; #query; #detached = false;

  fetch() {
    if (this.#detached) throw new TypeError('Cursor has been detached');
    return this.#connection.fetchNext(this.#query);
  }

  [Symbol.transfer]() {
    if (this.#detached) throw new TypeError('Cannot transfer detached cursor');
    const cursor = new DatabaseCursor(this.#connection, this.#query);
    this.#connection = undefined;
    this.#query = undefined;
    this.#detached = true;
    return cursor;
  }
}

function processCursor(cursor) {
  const owned = Object.transfer(cursor); // enforce the contract
  while (true) { const row = owned.fetch(); if (!row) break; handle(row); }
}
```

---
layout: center
class: text-center
---

# Thank You

Transfer Protocol -- Stage 0

<div class="mt-6">

`Symbol.transfer` | `Object.transfer()` | `Reflect.transfer()` | Proxy `transfer` trap

</div>

<div class="mt-8 text-sm" style="color: #6c757d;">
  Seeking feedback on the core concept and API design
</div>

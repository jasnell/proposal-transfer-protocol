---
theme: default
title: "Transferable Protocol"
info: |
  ## TC39 Proposal: Transferable Protocol
  A generalized ownership-transfer protocol for JavaScript.

  Stage 0
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
---

# Transferable Protocol

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
- **`ReadableStream` / `WritableStream` locking** -- `getReader()` locks a stream to a single consumer
- **`DisposableStack.prototype.move()`** -- moves resources to a new stack, marks original disposed
- **`postMessage()` transfer lists** -- `ArrayBuffer`, `MessagePort`, `OffscreenCanvas`, etc.
- **Node.js transferables** -- `X509Certificate`, `FileHandle`, `KeyObject`, etc.

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
| Delete      | `Object.delete...`      | `Reflect.deleteProperty()` | `deleteProperty` |
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
| Array iterators (`.values()`, etc.) | Yes | `Iterator.prototype[@@transfer]` |
| Map/Set iterators | Yes | `Iterator.prototype[@@transfer]` |
| Generator objects | Yes | `Iterator.prototype[@@transfer]` |
| Iterator Helper results (`.map()`, etc.) | Yes | `Iterator.prototype[@@transfer]` |
| Async generators | Yes | `AsyncIterator.prototype[@@transfer]` |
| `DisposableStack` | Yes | Delegates to `.move()` |
| `AsyncDisposableStack` | Yes | Delegates to `.move()` |
| `ArrayBuffer` | Yes | Delegates to `.transfer()` |
| Plain `{ next() {} }` objects | **No** | No `[Symbol.transfer]` |
| Arrays, Maps, Sets | **No** | Iterab*les*, not iterators |

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

# Post-Transfer Behavior

Operational methods throw, cleanup methods no-op

<div class="grid grid-cols-2 gap-6">
<div>

**Iterators:**

All methods throw `TypeError`:
- `.next()`
- `.return()`
- `.throw()`
- `[Symbol.iterator]()`
- `[Symbol.asyncIterator]()`

<div class="mt-2 text-sm" style="color: #6c757d;">
  <code>[Symbol.iterator]()</code> must throw too -- otherwise <code>for...of</code> would silently enter the loop before failing on <code>.next()</code>.
</div>

</div>
<div>

**DisposableStack:**

| Method | After transfer |
|--------|---------------|
| `.use()` | Throws `ReferenceError` |
| `.adopt()` | Throws `ReferenceError` |
| `.defer()` | Throws `ReferenceError` |
| `.move()` | Throws `ReferenceError` |
| `[Symbol.dispose]()` | No-op |
| `.disposed` | `true` |

<div class="mt-2 text-sm" style="color: #6c757d;">
  These are the <em>existing</em> behaviors -- no spec changes needed.
</div>

</div>
</div>

---
transition: slide-left
---

# Design Principle: Shape, Not Behavior

The protocol defines *when* the method is called -- not *what* it does

<v-clicks>

- Follows the `Symbol.dispose` model: spec doesn't enforce that `[Symbol.dispose]()` actually releases resources
- `[Symbol.transfer]()` isn't required to actually detach the source or move state
- The engine validates **mechanics** (callable, returns object) but not **semantics** (original rendered unusable)
- Detaching the source, throwing on post-transfer access -- these are contracts implementations are **expected** to follow

</v-clicks>

<div v-click class="mt-6 p-3 rounded" style="background: #cce5ff; color: #004085;">
  Transfer does not <em>ensure</em> exclusive access. It makes it <em>easier</em> to implement single-ownership patterns. The discipline of not sharing the result is the caller's responsibility.
</div>

---
transition: slide-left
---

# Transfer Is Not Copy, and Not Cleanup

Three distinct operations

<div class="grid grid-cols-3 gap-4 mt-4">
<div class="p-4 rounded" style="background: #e2e3f1; color: #1a1a2e;">

**Transfer (move)**

State moves to new owner. Source is detached.

```js
const b = Object.transfer(a);
// a is dead, b has the state
```

</div>
<div class="p-4 rounded" style="background: #e2e3f1; color: #1a1a2e;">

**Clone (copy)**

State is duplicated. Both remain valid.

```js
const b = structuredClone(a);
// both a and b are alive
```

</div>
<div class="p-4 rounded" style="background: #e2e3f1; color: #1a1a2e;">

**Dispose (cleanup)**

Resources are released. Object is done.

```js
a[Symbol.dispose]();
// a has cleaned up
```

</div>
</div>

<v-click>

<div class="mt-6 p-3 rounded" style="background: #fff3cd; color: #856404;">

**Key:** Transfer does <strong>not</strong> call `.return()` or `[Symbol.dispose]()` on the original. The new owner assumes responsibility for cleanup.

</div>

</v-click>

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

# Use Case: DisposableStack Handoff

Transfer decouples a stack from a `using` binding without triggering disposal

```js
function setupResources() {
  using stack = new DisposableStack();
  stack.use(openFile('a.txt'));
  stack.use(openFile('b.txt'));
  doValidation(); // if this throws, stack disposes both files
  return Object.transfer(stack);
  // stack is detached -- the `using` cleanup at block exit is a no-op
  // caller owns all registered resources via the returned stack
}

{
  using owned = setupResources();
  // ... use resources ...
  // owned[Symbol.dispose]() disposes both files at block exit
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
transition: slide-left
---

# Edge Cases

Things to be aware of

<v-clicks>

- **Transfer mid-iteration** -- next `.next()` call throws, loop's `finally` calls `.return()` which also throws
- **Frozen/sealed objects** -- `Object.transfer()` rejects them (`IsExtensible` check). Direct `[Symbol.transfer]()` bypasses this.
- **Iterator helper chains** -- only the outermost iterator is detached; inner iterators remain accessible if references are held
- **Chained transfers** -- transferring a detached object throws; transferring the current owner is valid
- **Async generators mid-execution** -- recommended: throw if `[[AsyncGeneratorState]]` is `"executing"` or pending `.next()` calls exist
- **`Symbol.transfer` on `Object.prototype`** -- exotic rejection prevents prototype pollution (follows `proposal-thenable-curtailment` model)

</v-clicks>

---
transition: slide-left
---

# Design Decisions

Why these choices?

| Decision | Rationale |
|----------|-----------|
| `Object.transfer()` not prototype method | Web compat -- `.transfer()` on `Iterator.prototype` risks breakage |
| `Symbol.transfer` not a string key | Follows `Symbol.dispose`, `Symbol.iterator` convention |
| No `Object.isTransferable()` | `Symbol.transfer in obj` is sufficient |
| Proxy trap + `Reflect` method | Every MOP op has this triad; transfer should too |
| `[[Transfer]]` as essential internal method | Same status as `[[Call]]`/`[[Construct]]` -- most objects don't have them either |
| No built-in `.detached` on iterators | Web compat concern; `ArrayBuffer.detached` and `DisposableStack.disposed` predate this |

---
transition: slide-left
---

# Relationship to Existing APIs

Additive, not replacement

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

**`ArrayBuffer.prototype.transfer()`**

`[Symbol.transfer]()` delegates to existing `.transfer()`. Existing code is unaffected.

```js
const buf = new ArrayBuffer(1024);
const owned = Object.transfer(buf);
buf.detached; // true
```

</div>
<div>

**`structuredClone` / `postMessage`**

Out of scope for TC39. Web platform can independently recognize `Symbol.transfer` in transfer lists as a WHATWG/W3C follow-on.

| | Web `Transferable` | `Symbol.transfer` |
|-|---|---|
| Scope | Cross-realm | Same-realm |
| Extensible | No | Yes |

</div>
</div>

<div v-click class="mt-4 text-sm" style="color: #6c757d;">

**Ecosystem prior art:** [Piscina](https://github.com/piscinajs/piscina) already defines a symbol-based transfer protocol (`[Piscina.transferableSymbol]` / `[Piscina.valueSymbol]`) for opting user-defined objects into `postMessage` transfer. [Comlink](https://github.com/GoogleChromeLabs/comlink) provides `Comlink.transfer()` and a `releaseProxy()` invalidation pattern.

</div>

---
transition: slide-left
---

# Anticipated Objections

<v-clicks>

- **"Too much machinery"** -- 5 new surface areas (symbol, internal method, 2 static methods, Proxy trap). Counter: MOP integration distinguishes this from a userland convention.

- **"Why not userland?"** -- Userland can't: add `[Symbol.transfer]` to built-in prototypes, add a Proxy trap, or add `[[Detached]]` slots to built-in iterators.

- **"Proxy trap is premature"** -- Fallback position: `Symbol.transfer` + `Object.transfer()` only. Smaller, but loses membranes.

- **"Frozen/sealed inconsistency"** -- `buf.transfer()` works on frozen buffers; `Object.transfer(buf)` would throw. Three options under consideration.

- **"Doesn't actually solve safety"** -- Transfer eliminates one reference but doesn't prevent sharing the result. Ecosystem benefit is incremental.

- **"Just use ReadableStream"** -- Heavy for simple iteration, not available in all JS environments.

</v-clicks>

---
transition: slide-left
---

# Open Questions

<div class="mt-4">

### 1. `ReferenceError` vs `TypeError` on post-transfer access?

`DisposableStack` throws `ReferenceError` (existing behavior). Iterators would throw `TypeError`. Should `[Symbol.transfer]()` normalize to `TypeError` for consistency, or preserve `.move()`'s existing error?

### 2. `Iterator.from()` as a bridge for non-transferable iterators?

```js
const plain = { i: 0, next() { return { value: this.i++, done: this.i > 3 }; } };
Object.transfer(plain);                // TypeError: not transferable
Object.transfer(Iterator.from(plain)); // OK -- wrapper is transferable
```

**Leaning no** -- transferring the wrapper detaches it, but the underlying plain object remains accessible. Leaky abstraction undermines the ownership guarantee.

</div>

---
layout: center
class: text-center
---

# Thank You

Transferable Protocol -- Stage 0

<div class="mt-6">

`Symbol.transfer` | `Object.transfer()` | `Reflect.transfer()` | Proxy `transfer` trap

</div>

<div class="mt-8 text-sm" style="color: #6c757d;">
  Seeking feedback on the core concept and API design
</div>

# Test Cases: Transfer Protocol

> Auto-generated from `spec.emu`. This file describes testable assertions
> extracted from the proposal spec text. Each entry provides sufficient
> detail for `/tc39-generate-test262` to produce a complete test262 test file.
>
> **Do not edit auto-generated entries by hand.** Instead, update spec.emu
> and re-run `/tc39-generate-tests`. You may add manual entries at the end
> under a `## Manual Test Cases` section — these will be preserved across
> regenerations.

**Spec version**: `d3a9dcb`
**Generated**: 2026-03-07
**Total assertions**: 150

---

## EnsureNotDetached (`sec-ensurenotdetached`)

### TC-0001: Throws TypeError when [[Detached]] is true
- **Clause**: `sec-ensurenotdetached` — EnsureNotDetached
- **Step**: 1
- **Assertion**: If _O_ has a [[Detached]] internal slot and _O_.[[Detached]] is *true*, throw a *TypeError* exception.
- **Category**: negative
- **Preconditions**: An iterator that has been transferred (so [[Detached]] is true). Create an array iterator via `[].values()`, transfer it with `Object.transfer()`, then attempt an operation on the original.
- **Test Description**: Create `const it = [1,2,3].values(); const it2 = Object.transfer(it);`. Then call `it.next()`. The original iterator's [[Detached]] is now true, so `EnsureNotDetached` (called from `%ArrayIteratorPrototype%.next`) should throw TypeError.
- **Expected**: TypeError thrown.

### TC-0002: Does not throw when [[Detached]] is false
- **Clause**: `sec-ensurenotdetached` — EnsureNotDetached
- **Step**: 1–2
- **Assertion**: If _O_ has [[Detached]] and it is *false*, EnsureNotDetached returns ~unused~ without throwing.
- **Category**: positive
- **Preconditions**: A freshly created array iterator (not transferred).
- **Test Description**: Create `const it = [1,2,3].values();`. Call `it.next()`. Since [[Detached]] is false (never transferred), the call succeeds normally.
- **Expected**: `{ value: 1, done: false }` returned; no exception.

### TC-0003: Does not throw for objects without [[Detached]] slot
- **Clause**: `sec-ensurenotdetached` — EnsureNotDetached
- **Step**: 1
- **Assertion**: The condition requires both that [[Detached]] exists AND is true. An object without [[Detached]] passes through without error.
- **Category**: positive
- **Preconditions**: A plain object used as a custom iterator (no [[Detached]] slot). This is tested indirectly: `Iterator.prototype[Symbol.iterator]` calls `EnsureNotDetached` when `_O_` is an Object, but a plain object with no [[Detached]] slot should not throw.
- **Test Description**: Create a plain object `const obj = { next() { return { value: 1, done: false }; } };`. Manually set its prototype to `Iterator.prototype`. Call `obj[Symbol.iterator]()`. Since `obj` has no [[Detached]] slot, `EnsureNotDetached` should not throw.
- **Expected**: Returns `obj` without throwing.

## OrdinaryTransfer (`sec-ordinarytransfer`)

### TC-0004: Throws TypeError on frozen object
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 1–2
- **Assertion**: If `IsExtensible(_O_)` returns *false*, throw a *TypeError*. `Object.freeze()` makes objects non-extensible.
- **Category**: negative
- **Preconditions**: An object with `[Symbol.transfer]()` method that has been frozen.
- **Test Description**: Create `const obj = { [Symbol.transfer]() { return {}; } }; Object.freeze(obj);`. Call `Object.transfer(obj)`. The `IsExtensible` check in `OrdinaryTransfer` should fail.
- **Expected**: TypeError thrown.

### TC-0005: Throws TypeError on sealed object
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 1–2
- **Assertion**: `Object.seal()` also makes objects non-extensible, so the same check triggers.
- **Category**: negative
- **Preconditions**: An object with `[Symbol.transfer]()` method that has been sealed.
- **Test Description**: Create `const obj = { [Symbol.transfer]() { return {}; } }; Object.seal(obj);`. Call `Object.transfer(obj)`.
- **Expected**: TypeError thrown.

### TC-0006: Throws TypeError on Object.preventExtensions object
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 1–2
- **Assertion**: `Object.preventExtensions()` makes objects non-extensible.
- **Category**: negative
- **Preconditions**: An object with `[Symbol.transfer]()` that has had `preventExtensions` called.
- **Test Description**: Create `const obj = { [Symbol.transfer]() { return {}; } }; Object.preventExtensions(obj);`. Call `Object.transfer(obj)`.
- **Expected**: TypeError thrown.

### TC-0007: Throws TypeError when Symbol.transfer is undefined (missing)
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 3–4
- **Assertion**: If `GetMethod(_O_, %Symbol.transfer%)` returns *undefined* (the method doesn't exist), throw a *TypeError*.
- **Category**: negative
- **Preconditions**: A plain extensible object with no `[Symbol.transfer]` property.
- **Test Description**: Call `Object.transfer({})`.
- **Expected**: TypeError thrown.

### TC-0008: Throws TypeError when Symbol.transfer is not callable
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 3
- **Assertion**: `GetMethod` throws *TypeError* if the property value is neither *undefined*, *null*, nor callable.
- **Category**: type-check
- **Preconditions**: An extensible object with `[Symbol.transfer]` set to a non-callable value.
- **Test Description**: Call `Object.transfer({ [Symbol.transfer]: 42 })`.
- **Expected**: TypeError thrown (from GetMethod).

### TC-0009: Calls Symbol.transfer with the object as this value
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 5
- **Assertion**: `Call(_transferFn_, _O_)` invokes the transfer function with `_O_` as the *this* value.
- **Category**: observable-ordering
- **Preconditions**: An object with a `[Symbol.transfer]` that records its `this` value.
- **Test Description**: Create `let thisVal; const obj = { [Symbol.transfer]() { thisVal = this; return {}; } };`. Call `Object.transfer(obj)`. Verify `thisVal === obj`.
- **Expected**: `thisVal` is the original object `obj`.

### TC-0010: Throws TypeError when Symbol.transfer returns non-object
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 6
- **Assertion**: If _result_ is not an Object, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: An object whose `[Symbol.transfer]` returns a primitive.
- **Test Description**: Test with each primitive return type: `undefined`, `null`, `true`, `42`, `"string"`, `Symbol()`, `0n`. For each, create `{ [Symbol.transfer]() { return <primitive>; } }` and call `Object.transfer()`.
- **Expected**: TypeError thrown for each primitive return value.

### TC-0011: Returns the object returned by Symbol.transfer
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 7
- **Assertion**: The return value of OrdinaryTransfer is the object returned by `[Symbol.transfer]()`.
- **Category**: positive
- **Preconditions**: An object whose `[Symbol.transfer]` returns a specific object.
- **Test Description**: Create `const result = {}; const obj = { [Symbol.transfer]() { return result; } };`. Call `const r = Object.transfer(obj);`. Verify `r === result`.
- **Expected**: `r` is the exact same object as `result`.

### TC-0012: IsExtensible check happens before GetMethod
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 1–3
- **Assertion**: The `IsExtensible` check (step 1–2) happens before `GetMethod` (step 3). A frozen object should throw TypeError without ever accessing `[Symbol.transfer]`.
- **Category**: observable-ordering
- **Preconditions**: A frozen object with a `[Symbol.transfer]` getter that records access.
- **Test Description**: Create `let accessed = false; const obj = Object.freeze(Object.defineProperty({}, Symbol.transfer, { get() { accessed = true; return () => ({}); }, configurable: true }));`. Note: `Object.freeze` after `defineProperty` won't work since freeze makes it non-configurable. Instead: `const obj = {}; Object.defineProperty(obj, Symbol.transfer, { get() { accessed = true; return () => ({}); }, configurable: true, enumerable: true }); Object.preventExtensions(obj);`. Call `Object.transfer(obj)` and expect TypeError. Verify `accessed` is still false.
- **Expected**: TypeError thrown; `accessed` is `false` (Symbol.transfer was never read).

### TC-0013: GetMethod for Symbol.transfer is the only observable property access
- **Clause**: `sec-ordinarytransfer` — OrdinaryTransfer
- **Step**: 1–7
- **Assertion**: Beyond `IsExtensible` and `GetMethod` for `Symbol.transfer`, OrdinaryTransfer performs no other observable operations on the object before calling the transfer function.
- **Category**: observable-ordering
- **Preconditions**: A Proxy wrapping an object with `[Symbol.transfer]`.
- **Test Description**: Create a Proxy around `{ [Symbol.transfer]() { return {}; } }` that logs all trap calls. Call `Object.transfer(proxy)`. Verify the only traps called are: `isExtensible` (from step 1), `get` for `Symbol.transfer` (from GetMethod step 3), then whatever `Call` does. No `has`, `getOwnPropertyDescriptor`, `ownKeys`, etc. should be called as part of OrdinaryTransfer itself.
- **Expected**: Trap log shows `isExtensible`, `get` (for Symbol.transfer), then the call to the transfer function.

## ValidateTransferTrapResult (`sec-validatetransfertrapresult`)

### TC-0014: Throws TypeError if trapResult is not an Object
- **Clause**: `sec-validatetransfertrapresult` — ValidateTransferTrapResult
- **Step**: 1
- **Assertion**: If _trapResult_ is not an Object, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: A Proxy with a `transfer` trap that returns a primitive.
- **Test Description**: For each primitive type (`undefined`, `null`, `42`, `"str"`, `true`, `Symbol()`, `0n`): create `const p = new Proxy({ [Symbol.transfer]() { return {}; } }, { transfer(target) { return <primitive>; } });`. Call `Object.transfer(p)`.
- **Expected**: TypeError thrown for each primitive.

### TC-0015: Throws TypeError when target has non-configurable non-writable Symbol.transfer = undefined
- **Clause**: `sec-validatetransfertrapresult` — ValidateTransferTrapResult
- **Step**: 2–3.a.i.a
- **Assertion**: If the target has an own property keyed by `Symbol.transfer` that is non-configurable, is a data descriptor, is non-writable, and has value *undefined*, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: A target object with `Symbol.transfer` defined as `{ value: undefined, writable: false, configurable: false }`, wrapped in a Proxy whose transfer trap returns an object.
- **Test Description**: Create `const target = {}; Object.defineProperty(target, Symbol.transfer, { value: undefined, writable: false, configurable: false });`. Create `const p = new Proxy(target, { transfer() { return {}; } });`. Call `Object.transfer(p)`.
- **Expected**: TypeError thrown (target is explicitly non-transferable).

### TC-0016: Succeeds when target has no Symbol.transfer property
- **Clause**: `sec-validatetransfertrapresult` — ValidateTransferTrapResult
- **Step**: 2–3
- **Assertion**: If `_target_.[[GetOwnProperty]](%Symbol.transfer%)` returns *undefined* (no such property), the validation succeeds.
- **Category**: positive
- **Preconditions**: A target with no `Symbol.transfer`, wrapped in a Proxy whose transfer trap returns an object.
- **Test Description**: Create `const p = new Proxy({}, { transfer() { return { transferred: true }; } });`. Call `Object.transfer(p)`.
- **Expected**: Returns `{ transferred: true }` without error.

### TC-0017: Succeeds when target has configurable Symbol.transfer
- **Clause**: `sec-validatetransfertrapresult` — ValidateTransferTrapResult
- **Step**: 3.a
- **Assertion**: If `_targetDesc_.[[Configurable]]` is *true*, the invariant check is skipped.
- **Category**: positive
- **Preconditions**: A target with `Symbol.transfer` as configurable (even if value is undefined).
- **Test Description**: Create `const target = {}; Object.defineProperty(target, Symbol.transfer, { value: undefined, writable: false, configurable: true });`. Wrap in Proxy with `transfer` trap returning `{}`. Call `Object.transfer(p)`.
- **Expected**: Succeeds; returns the trap result.

### TC-0018: Succeeds when target has non-configurable but writable Symbol.transfer
- **Clause**: `sec-validatetransfertrapresult` — ValidateTransferTrapResult
- **Step**: 3.a.i
- **Assertion**: If `_targetDesc_.[[Writable]]` is *true*, even though `[[Configurable]]` is *false*, the invariant check is skipped.
- **Category**: positive
- **Preconditions**: A target with `Symbol.transfer` as `{ value: undefined, writable: true, configurable: false }`.
- **Test Description**: Create `const target = {}; Object.defineProperty(target, Symbol.transfer, { value: undefined, writable: true, configurable: false });`. Wrap in Proxy with `transfer` trap returning `{}`. Call `Object.transfer(p)`.
- **Expected**: Succeeds; returns the trap result.

### TC-0019: Succeeds when target has non-configurable non-writable Symbol.transfer with non-undefined value
- **Clause**: `sec-validatetransfertrapresult` — ValidateTransferTrapResult
- **Step**: 3.a.i.a
- **Assertion**: The check only triggers when `[[Value]]` is *undefined*. A non-undefined value in a non-configurable, non-writable data property does not trigger the invariant.
- **Category**: positive
- **Preconditions**: A target with `Symbol.transfer` as `{ value: function(){}, writable: false, configurable: false }`.
- **Test Description**: Create `const target = {}; Object.defineProperty(target, Symbol.transfer, { value: () => ({}), writable: false, configurable: false });`. Wrap in Proxy with `transfer` trap returning `{}`. Call `Object.transfer(p)`.
- **Expected**: Succeeds; returns the trap result.

### TC-0020: Succeeds when target has non-configurable accessor property for Symbol.transfer
- **Clause**: `sec-validatetransfertrapresult` — ValidateTransferTrapResult
- **Step**: 3.a
- **Assertion**: The check at step 3.a requires `IsDataDescriptor(_targetDesc_)` to be *true*. An accessor property (getter/setter) is not a data descriptor, so the check is skipped.
- **Category**: positive
- **Preconditions**: A target with `Symbol.transfer` as a non-configurable accessor.
- **Test Description**: Create `const target = {}; Object.defineProperty(target, Symbol.transfer, { get() { return () => ({}); }, configurable: false });`. Wrap in Proxy with `transfer` trap returning `{}`. Call `Object.transfer(p)`.
- **Expected**: Succeeds; returns the trap result.

## CreateTransferredIterator (`sec-createtransferrediterator`)

### TC-0021: Transfers array iterator slots to new object
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 2–2.g
- **Assertion**: When _O_ has [[IteratedArrayLike]], [[ArrayLikeNextIndex]], and [[ArrayLikeIterationKind]], the new iterator receives copies of those slots and can continue iteration from where the original left off.
- **Category**: positive
- **Preconditions**: An array iterator that has been partially consumed.
- **Test Description**: Create `const it = [10, 20, 30].values(); it.next();` (consumes first element). Transfer: `const it2 = Object.transfer(it);`. Call `it2.next()`.
- **Expected**: `it2.next()` returns `{ value: 20, done: false }` (continues from index 1).

### TC-0022: Clears original array iterator slots after transfer
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 2.f–2.g
- **Assertion**: After transfer, `_O_.[[IteratedArrayLike]]` is set to *undefined* and `_O_.[[ArrayLikeNextIndex]]` is set to 0.
- **Category**: positive
- **Preconditions**: An array iterator that has been transferred.
- **Test Description**: Create `const it = [10, 20, 30].values(); Object.transfer(it);`. Call `it.next()`. Since [[IteratedArrayLike]] is now undefined, the iterator should return a done result (this is how the existing %ArrayIteratorPrototype%.next algorithm handles `[[IteratedArrayLike]]` being undefined).
- **Expected**: TypeError thrown (detached guard fires before reaching the undefined-array check).

### TC-0023: New array iterator has same prototype as original
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 1, 2.b
- **Assertion**: `_proto_` is obtained from `_O_.[[GetPrototypeOf]]()` and passed to `OrdinaryObjectCreate`. The new iterator has the same prototype.
- **Category**: positive
- **Preconditions**: An array iterator.
- **Test Description**: Create `const it = [].values(); const it2 = Object.transfer(it);`. Check `Object.getPrototypeOf(it2) === Object.getPrototypeOf([].values())`.
- **Expected**: `true` — prototypes match.

### TC-0024: Sets original [[Detached]] to true, new [[Detached]] to false
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 7–8
- **Assertion**: `_newIterator_.[[Detached]]` is *false*, `_O_.[[Detached]]` is *true*.
- **Category**: positive
- **Preconditions**: Any transferable iterator.
- **Test Description**: Create `const it = [1].values(); const it2 = Object.transfer(it);`. Verify `it2.next()` succeeds (not detached) and `it.next()` throws TypeError (detached).
- **Expected**: `it2.next()` returns `{ value: 1, done: false }`; `it.next()` throws TypeError.

### TC-0025: Throws TypeError when transferring an executing generator
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 3.a
- **Assertion**: If `_O_.[[GeneratorState]]` is ~executing~, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: A generator that attempts to transfer itself mid-execution.
- **Test Description**: Create `function* g() { yield Object.transfer(this_gen); } const this_gen = g(); this_gen.next();` — when `.next()` resumes, the generator is in executing state and tries to transfer itself. More precisely: `let gen; function* g() { yield Object.transfer(gen); } gen = g(); gen.next(); gen.next();` — the second `.next()` resumes into the yield which calls `Object.transfer(gen)` while `gen` is executing.
- **Expected**: TypeError thrown from inside the generator.

### TC-0026: Transfers generator in suspended-start state
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 3–3.h
- **Assertion**: A generator in ~suspended-start~ state can be transferred. The new generator receives [[GeneratorState]], [[GeneratorContext]], [[GeneratorBrand]].
- **Category**: positive
- **Preconditions**: A generator that has been created but never iterated.
- **Test Description**: Create `function* g() { yield 1; yield 2; } const it = g(); const it2 = Object.transfer(it); it2.next();`.
- **Expected**: `it2.next()` returns `{ value: 1, done: false }`.

### TC-0027: Transfers generator in suspended-yield state
- **Clause**: `sec-createtransferrediterator` �� CreateTransferredIterator
- **Step**: 3–3.h
- **Assertion**: A generator paused at a yield point can be transferred. The new generator continues from where the original left off.
- **Category**: positive
- **Preconditions**: A generator that has yielded once.
- **Test Description**: Create `function* g() { yield 1; yield 2; yield 3; } const it = g(); it.next(); const it2 = Object.transfer(it); it2.next();`.
- **Expected**: `it2.next()` returns `{ value: 2, done: false }`.

### TC-0028: Original generator state set to completed after transfer
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 3.g
- **Assertion**: `_O_.[[GeneratorState]]` is set to ~completed~ after transfer.
- **Category**: positive
- **Preconditions**: A generator that has been transferred.
- **Test Description**: Create `function* g() { yield 1; } const it = g(); Object.transfer(it);`. Call `it.next()` — the detach guard fires first, but if we could bypass it, the generator would be completed. Test via: transfer succeeds, and calling `.next()` on original throws TypeError (from detach guard, not completed state — but both prevent use).
- **Expected**: `it.next()` throws TypeError (detached).

### TC-0029: Transfers iterator helper with [[UnderlyingIterators]]
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 3.b
- **Assertion**: If _O_ has an [[UnderlyingIterators]] internal slot (Iterator Helper), that slot is transferred and cleared on the original.
- **Category**: positive
- **Preconditions**: An iterator helper (e.g., from `.map()`, `.filter()`).
- **Test Description**: Create `const it = [1, 2, 3].values().map(x => x * 2); it.next(); const it2 = Object.transfer(it); it2.next();`.
- **Expected**: `it2.next()` returns `{ value: 4, done: false }` (continues from where original left off; 1*2=2 was already consumed, so next is 2*2=4).

### TC-0030: Clears original [[UnderlyingIterators]] after transfer
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 3.b.d
- **Assertion**: `_O_.[[UnderlyingIterators]]` is set to *undefined* after transfer.
- **Category**: positive
- **Preconditions**: An iterator helper that has been transferred.
- **Test Description**: Create `const it = [1, 2, 3].values().map(x => x * 2); Object.transfer(it);`. Attempt `it.next()`.
- **Expected**: TypeError thrown (detached).

### TC-0031: Throws TypeError when transferring executing async generator
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 4.a
- **Assertion**: If `_O_.[[AsyncGeneratorState]]` is ~executing~, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: An async generator that attempts to transfer itself mid-execution.
- **Test Description**: Create `let agen; async function* ag() { yield Object.transfer(agen); } agen = ag(); await agen.next(); await agen.next();` — the second `.next()` resumes into the yield, which calls `Object.transfer(agen)` while it is executing.
- **Expected**: The promise from `.next()` rejects with TypeError.

### TC-0032: Throws TypeError when transferring draining-queue async generator
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 4.a
- **Assertion**: If `_O_.[[AsyncGeneratorState]]` is ~draining-queue~, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: An async generator in draining-queue state. This state occurs when the generator body has returned/thrown but there are still pending requests in the queue.
- **Test Description**: This is difficult to observe directly in user code since draining-queue is a transient internal state. A test would need to exploit the timing of async generator completion with multiple pending `.next()` calls. The assertion is primarily for engine-internal correctness.
- **Expected**: TypeError thrown (if this state is reachable during a transfer attempt).

### TC-0033: Throws TypeError when async generator queue is non-empty
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 4.b
- **Assertion**: If the number of elements in `_O_.[[AsyncGeneratorQueue]]` is not 0, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: An async generator with pending `.next()` requests in its queue.
- **Test Description**: Create `async function* ag() { yield 1; yield 2; } const it = ag(); it.next(); it.next();` — calling `.next()` twice enqueues two requests. Before any resolve, attempt `Object.transfer(it)`. However, since the first `.next()` will start executing the generator (making it ~executing~), the executing check fires first. A more precise test: `async function* ag() { await new Promise(r => setTimeout(r, 100)); yield 1; } const it = ag(); it.next(); /* it is now executing */`. The queue check is a belt-and-suspenders safety net.
- **Expected**: TypeError thrown (from either executing state check or queue length check).

### TC-0034: Transfers async generator in suspended-start state
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 4–4.j
- **Assertion**: An async generator in ~suspended-start~ can be transferred.
- **Category**: positive
- **Preconditions**: A freshly created async generator (never iterated).
- **Test Description**: Create `async function* ag() { yield 1; yield 2; } const it = ag(); const it2 = Object.transfer(it); const r = await it2.next();`.
- **Expected**: `r` is `{ value: 1, done: false }`.

### TC-0035: New async generator queue is always a new empty List
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 4.f
- **Assertion**: `_newIterator_.[[AsyncGeneratorQueue]]` is set to a new empty List, not copied from the original.
- **Category**: positive
- **Preconditions**: An async generator that has been transferred.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); const it2 = Object.transfer(it); const r = await it2.next();`. Verify the new iterator works correctly (its queue starts fresh).
- **Expected**: `r` is `{ value: 1, done: false }`.

### TC-0036: Original async generator state set to completed after transfer
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 4.h
- **Assertion**: `_O_.[[AsyncGeneratorState]]` is set to ~completed~ after transfer.
- **Category**: positive
- **Preconditions**: An async generator that has been transferred.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); Object.transfer(it);`. Call `it.next()`.
- **Expected**: TypeError thrown (detached guard prevents reaching the completed state logic).

### TC-0037: Transfers RegExp string iterator slots
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 5–5.h
- **Assertion**: When _O_ has [[IteratingRegExp]], [[IteratedString]], [[Global]], [[Unicode]], [[Done]], all are transferred to the new iterator.
- **Category**: positive
- **Preconditions**: A RegExp string iterator created via `String.prototype.matchAll()`.
- **Test Description**: Create `const it = "aaa".matchAll(/a/g); it.next();` (consumes first match). Transfer: `const it2 = Object.transfer(it); it2.next();`.
- **Expected**: `it2.next()` returns the second match result (value is a match array for the second "a").

### TC-0038: Clears original RegExp string iterator slots
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 5.i–5.k
- **Assertion**: After transfer, `_O_.[[IteratingRegExp]]` and `_O_.[[IteratedString]]` are set to *undefined*, `_O_.[[Done]]` is set to *true*.
- **Category**: positive
- **Preconditions**: A RegExp string iterator that has been transferred.
- **Test Description**: Create `const it = "aaa".matchAll(/a/g); Object.transfer(it);`. Call `it.next()`.
- **Expected**: TypeError thrown (detached).

### TC-0039: Transfers WrapForValidIterator [[Iterated]] slot
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 6–6.c
- **Assertion**: When _O_ has [[Iterated]] (from `Iterator.from()` wrapping a non-Iterator), that slot is transferred.
- **Category**: positive
- **Preconditions**: A WrapForValidIterator created by `Iterator.from()` on a plain iterator object.
- **Test Description**: Create `const raw = { next() { return { value: 42, done: false }; } }; const it = Iterator.from(raw); const it2 = Object.transfer(it); it2.next();`.
- **Expected**: `it2.next()` returns `{ value: 42, done: false }`.

### TC-0040: Clears original WrapForValidIterator [[Iterated]]
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 6.d
- **Assertion**: After transfer, `_O_.[[Iterated]]` is set to *undefined*.
- **Category**: positive
- **Preconditions**: A WrapForValidIterator that has been transferred.
- **Test Description**: Create `const it = Iterator.from({ next() { return { value: 1, done: false }; } }); Object.transfer(it);`. Call `it.next()`.
- **Expected**: TypeError thrown (detached).

### TC-0041: Throws TypeError for unrecognized iterator type
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 7
- **Assertion**: If _O_ does not match any of the recognized internal slot patterns (array iterator, generator, async generator, regexp string iterator, WrapForValidIterator), throw a *TypeError*.
- **Category**: negative
- **Preconditions**: An object with [[Detached]] but none of the recognized iterator slot sets. This would be an unusual edge case — perhaps a custom exotic object.
- **Test Description**: This case cannot be triggered with standard built-in iterators since all paths that set [[Detached]] also set one of the recognized slot patterns. It would require engine-internal manipulation. The assertion ensures future iterator types that add [[Detached]] but forget to add a branch will fail explicitly rather than silently.
- **Expected**: TypeError thrown.

### TC-0042: Preserves prototype across transfer for all iterator types
- **Clause**: `sec-createtransferrediterator` — CreateTransferredIterator
- **Step**: 1
- **Assertion**: The prototype of the new iterator matches the prototype of the original for every iterator type.
- **Category**: positive
- **Preconditions**: Iterators of each type.
- **Test Description**: For each iterator type (array, generator, Map, Set, String matchAll, Iterator.from wrapper), create an iterator, transfer it, and verify `Object.getPrototypeOf(transferred) === Object.getPrototypeOf(original_before_transfer)`.
- **Expected**: Prototypes match for all types.

## [[Transfer]] for Ordinary Objects (`sec-ordinary-object-transfer`)

### TC-0043: Delegates to OrdinaryTransfer
- **Clause**: `sec-ordinary-object-transfer` — [[Transfer]] ( )
- **Step**: 1
- **Assertion**: The [[Transfer]] internal method of an ordinary object delegates to OrdinaryTransfer.
- **Category**: positive
- **Preconditions**: An ordinary object with a `[Symbol.transfer]()` method.
- **Test Description**: Create `const result = {}; const obj = { [Symbol.transfer]() { return result; } };`. Call `Object.transfer(obj)`. This invokes `obj.[[Transfer]]()` which calls `OrdinaryTransfer(obj)`.
- **Expected**: Returns `result`.

## [[Transfer]] for Proxy Objects (`sec-proxy-object-transfer`)

### TC-0044: Throws TypeError on revoked proxy
- **Clause**: `sec-proxy-object-transfer` — [[Transfer]] ( )
- **Step**: 1
- **Assertion**: `ValidateNonRevokedProxy(_O_)` throws *TypeError* if the proxy has been revoked.
- **Category**: negative
- **Preconditions**: A revoked Proxy.
- **Test Description**: Create `const { proxy, revoke } = Proxy.revocable({ [Symbol.transfer]() { return {}; } }, {}); revoke();`. Call `Object.transfer(proxy)`.
- **Expected**: TypeError thrown.

### TC-0045: Falls through to target [[Transfer]] when no trap
- **Clause**: `sec-proxy-object-transfer` — [[Transfer]] ( )
- **Step**: 5–6
- **Assertion**: If `GetMethod(_handler_, "transfer")` returns *undefined*, the operation falls through to `_target_.[[Transfer]]()`.
- **Category**: positive
- **Preconditions**: A Proxy with no `transfer` trap, wrapping a transferable target.
- **Test Description**: Create `const result = {}; const target = { [Symbol.transfer]() { return result; } }; const p = new Proxy(target, {});`. Call `Object.transfer(p)`.
- **Expected**: Returns `result` (the target's `[Symbol.transfer]()` was invoked).

### TC-0046: Calls trap with handler as this and target as argument
- **Clause**: `sec-proxy-object-transfer` — [[Transfer]] ( )
- **Step**: 7
- **Assertion**: `Call(_trap_, _handler_, « _target_ »)` calls the trap with the handler as `this` and the target as the sole argument.
- **Category**: observable-ordering
- **Preconditions**: A Proxy with a `transfer` trap that records its `this` and arguments.
- **Test Description**: Create `let trapThis, trapArgs; const target = { [Symbol.transfer]() { return {}; } }; const handler = { transfer(...args) { trapThis = this; trapArgs = args; return {}; } }; const p = new Proxy(target, handler);`. Call `Object.transfer(p)`. Verify `trapThis === handler` and `trapArgs[0] === target` and `trapArgs.length === 1`.
- **Expected**: `trapThis` is the handler; `trapArgs` is `[target]`.

### TC-0047: Returns trap result
- **Clause**: `sec-proxy-object-transfer` — [[Transfer]] ( )
- **Step**: 9
- **Assertion**: The return value of [[Transfer]] on a Proxy is the trap result (after validation).
- **Category**: positive
- **Preconditions**: A Proxy with a `transfer` trap returning a specific object.
- **Test Description**: Create `const expected = { id: 123 }; const p = new Proxy({}, { transfer() { return expected; } });`. Call `const r = Object.transfer(p);`.
- **Expected**: `r === expected`.

### TC-0048: Validates trap result via ValidateTransferTrapResult
- **Clause**: `sec-proxy-object-transfer` — [[Transfer]] ( )
- **Step**: 8
- **Assertion**: After calling the trap, `ValidateTransferTrapResult(_target_, _trapResult_)` is invoked. If the target has a non-configurable, non-writable `Symbol.transfer` data property with value *undefined*, the validation throws even though the trap returned an object.
- **Category**: negative
- **Preconditions**: A target with non-configurable non-writable `Symbol.transfer = undefined`, wrapped in a Proxy with a transfer trap.
- **Test Description**: Create `const target = {}; Object.defineProperty(target, Symbol.transfer, { value: undefined, writable: false, configurable: false }); const p = new Proxy(target, { transfer() { return {}; } });`. Call `Object.transfer(p)`.
- **Expected**: TypeError thrown (from ValidateTransferTrapResult).

### TC-0049: Proxy transfer trap receives target, not proxy
- **Clause**: `sec-proxy-object-transfer` — [[Transfer]] ( )
- **Step**: 7
- **Assertion**: The trap receives the underlying `_target_`, not the proxy itself. This is stated in the note: "there is no _receiver_ parameter."
- **Category**: observable-ordering
- **Preconditions**: A Proxy with a transfer trap.
- **Test Description**: Create `const target = {}; let received; const p = new Proxy(target, { transfer(t) { received = t; return {}; } });`. Call `Object.transfer(p)`. Verify `received === target` and `received !== p`.
- **Expected**: `received` is `target`, not `p`.

### TC-0050: Nested proxies — trap on outer, falls through on inner
- **Clause**: `sec-proxy-object-transfer` — [[Transfer]] ( )
- **Step**: 5–7
- **Assertion**: When a Proxy wraps another Proxy, each level's transfer trap is consulted independently. If the outer has no trap, it falls through to the inner proxy's [[Transfer]].
- **Category**: positive
- **Preconditions**: Two nested proxies; inner has a transfer trap, outer has none.
- **Test Description**: Create `const result = {}; const target = { [Symbol.transfer]() { return result; } }; const inner = new Proxy(target, { transfer(t) { return { inner: true }; } }); const outer = new Proxy(inner, {});`. Call `Object.transfer(outer)`.
- **Expected**: Returns `{ inner: true }` (outer fell through to inner's trap).

## Object.transfer (`sec-object.transfer`)

### TC-0051: Throws TypeError for undefined
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: 1
- **Assertion**: If _obj_ is not an Object, throw a *TypeError*. `undefined` is not an Object.
- **Category**: type-check
- **Preconditions**: None.
- **Test Description**: Call `Object.transfer(undefined)`.
- **Expected**: TypeError thrown.

### TC-0052: Throws TypeError for null
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: 1
- **Assertion**: `null` is not an Object.
- **Category**: type-check
- **Preconditions**: None.
- **Test Description**: Call `Object.transfer(null)`.
- **Expected**: TypeError thrown.

### TC-0053: Throws TypeError for boolean
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: 1
- **Assertion**: Booleans are not Objects.
- **Category**: type-check
- **Preconditions**: None.
- **Test Description**: Call `Object.transfer(true)` and `Object.transfer(false)`.
- **Expected**: TypeError thrown for both.

### TC-0054: Throws TypeError for number
- **Clause**: `sec-object.transfer` �� Object.transfer
- **Step**: 1
- **Assertion**: Numbers are not Objects.
- **Category**: type-check
- **Preconditions**: None.
- **Test Description**: Call `Object.transfer(42)`.
- **Expected**: TypeError thrown.

### TC-0055: Throws TypeError for string
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: 1
- **Assertion**: Strings are not Objects.
- **Category**: type-check
- **Preconditions**: None.
- **Test Description**: Call `Object.transfer("hello")`.
- **Expected**: TypeError thrown.

### TC-0056: Throws TypeError for symbol
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: 1
- **Assertion**: Symbols are not Objects.
- **Category**: type-check
- **Preconditions**: None.
- **Test Description**: Call `Object.transfer(Symbol())`.
- **Expected**: TypeError thrown.

### TC-0057: Throws TypeError for bigint
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: 1
- **Assertion**: BigInts are not Objects.
- **Category**: type-check
- **Preconditions**: None.
- **Test Description**: Call `Object.transfer(0n)`.
- **Expected**: TypeError thrown.

### TC-0058: Calls [[Transfer]] on argument and returns result
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: 2
- **Assertion**: Returns the result of `_obj_.[[Transfer]]()`.
- **Category**: positive
- **Preconditions**: A transferable object.
- **Test Description**: Create `const result = {}; const obj = { [Symbol.transfer]() { return result; } }; const r = Object.transfer(obj);`.
- **Expected**: `r === result`.

### TC-0059: Object.transfer is writable, non-enumerable, configurable
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: (property attributes)
- **Assertion**: As a built-in function property of the Object constructor, it has attributes `{ [[Writable]]: true, [[Enumerable]]: false, [[Configurable]]: true }`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Call `Object.getOwnPropertyDescriptor(Object, "transfer")`.
- **Expected**: `{ value: Object.transfer, writable: true, enumerable: false, configurable: true }`.

### TC-0060: Object.transfer name is "transfer"
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: (name property)
- **Assertion**: The `"name"` property is `"transfer"`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `Object.transfer.name`.
- **Expected**: `"transfer"`.

### TC-0061: Object.transfer length is 1
- **Clause**: `sec-object.transfer` — Object.transfer
- **Step**: (length property)
- **Assertion**: The `"length"` property is `1` (one formal parameter: _obj_).
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `Object.transfer.length`.
- **Expected**: `1`.

## Symbol.transfer (`sec-symbol.transfer`)

### TC-0062: Symbol.transfer is a symbol
- **Clause**: `sec-symbol.transfer` — Symbol.transfer
- **Step**: (value)
- **Assertion**: The value of `Symbol.transfer` is a well-known symbol.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `typeof Symbol.transfer`.
- **Expected**: `"symbol"`.

### TC-0063: Symbol.transfer has description "Symbol.transfer"
- **Clause**: `sec-symbol.transfer` — Symbol.transfer
- **Step**: (description)
- **Assertion**: The [[Description]] of %Symbol.transfer% is `"Symbol.transfer"`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `Symbol.transfer.description`.
- **Expected**: `"Symbol.transfer"`.

### TC-0064: Symbol.transfer is non-writable, non-enumerable, non-configurable
- **Clause**: `sec-symbol.transfer` — Symbol.transfer
- **Step**: (attributes)
- **Assertion**: The property has attributes `{ [[Writable]]: false, [[Enumerable]]: false, [[Configurable]]: false }`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Call `Object.getOwnPropertyDescriptor(Symbol, "transfer")`.
- **Expected**: `{ value: Symbol.transfer, writable: false, enumerable: false, configurable: false }`.

### TC-0065: Symbol.transfer identity is stable
- **Clause**: `sec-symbol.transfer` — Symbol.transfer
- **Step**: (well-known symbol identity)
- **Assertion**: Accessing `Symbol.transfer` multiple times returns the same symbol.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `Symbol.transfer === Symbol.transfer`.
- **Expected**: `true`.

## Object.prototype Transfer Rejection (`sec-object.prototype-transfer-rejection`)

### TC-0066: DefineOwnProperty rejects Symbol.transfer on Object.prototype
- **Clause**: `sec-object.prototype-transfer-rejection` — Modifications to the Object Prototype Object
- **Step**: (DefineOwnProperty modification)
- **Assertion**: `Object.defineProperty(Object.prototype, Symbol.transfer, { value: fn })` returns *false* (or throws in strict mode) without creating the property.
- **Category**: negative
- **Preconditions**: None.
- **Test Description**: Attempt `Object.defineProperty(Object.prototype, Symbol.transfer, { value: () => ({}), configurable: true });`.
- **Expected**: TypeError thrown (since `Object.defineProperty` calls `DefinePropertyOrThrow` which throws when [[DefineOwnProperty]] returns false).

### TC-0067: Set rejects Symbol.transfer on Object.prototype
- **Clause**: `sec-object.prototype-transfer-rejection` — Modifications to the Object Prototype Object
- **Step**: (Set modification)
- **Assertion**: When the receiver is %Object.prototype% and the property key is %Symbol.transfer%, [[Set]] returns *false*.
- **Category**: negative
- **Preconditions**: None.
- **Test Description**: In strict mode: `Object.prototype[Symbol.transfer] = () => ({};`. In sloppy mode, verify the assignment silently fails: `Object.prototype[Symbol.transfer] = () => ({}); assert(Object.prototype[Symbol.transfer] === undefined);`.
- **Expected**: Strict mode: TypeError thrown. Sloppy mode: assignment silently fails, property remains absent.

### TC-0068: Object.prototype does not have Symbol.transfer after rejection
- **Clause**: `sec-object.prototype-transfer-rejection` — Modifications to the Object Prototype Object
- **Step**: (consequence)
- **Assertion**: After a failed attempt to set `Symbol.transfer` on `Object.prototype`, the property does not exist.
- **Category**: positive
- **Preconditions**: Attempt to set `Symbol.transfer` on `Object.prototype` (will fail).
- **Test Description**: Try `try { Object.defineProperty(Object.prototype, Symbol.transfer, { value: 1 }); } catch(e) {}`. Then check `Symbol.transfer in Object.prototype`.
- **Expected**: `false`.

### TC-0069: Plain objects cannot inherit transferability from Object.prototype
- **Clause**: `sec-object.prototype-transfer-rejection` — Modifications to the Object Prototype Object
- **Step**: (consequence)
- **Assertion**: Because `Symbol.transfer` cannot be set on `Object.prototype`, plain objects do not inherit a `[Symbol.transfer]()` method, and `Object.transfer({})` throws.
- **Category**: negative
- **Preconditions**: None.
- **Test Description**: Call `Object.transfer({})`.
- **Expected**: TypeError thrown (no `[Symbol.transfer]` method found).

## Reflect.transfer (`sec-reflect.transfer`)

### TC-0070: Throws TypeError for non-object arguments
- **Clause**: `sec-reflect.transfer` — Reflect.transfer
- **Step**: 1
- **Assertion**: If _target_ is not an Object, throw a *TypeError*.
- **Category**: type-check
- **Preconditions**: None.
- **Test Description**: Call `Reflect.transfer(undefined)`, `Reflect.transfer(null)`, `Reflect.transfer(42)`, `Reflect.transfer("str")`, `Reflect.transfer(true)`, `Reflect.transfer(Symbol())`, `Reflect.transfer(0n)`.
- **Expected**: TypeError thrown for each.

### TC-0071: Calls [[Transfer]] on target and returns result
- **Clause**: `sec-reflect.transfer` — Reflect.transfer
- **Step**: 2
- **Assertion**: Returns the result of `_target_.[[Transfer]]()`.
- **Category**: positive
- **Preconditions**: A transferable object.
- **Test Description**: Create `const result = {}; const obj = { [Symbol.transfer]() { return result; } }; const r = Reflect.transfer(obj);`.
- **Expected**: `r === result`.

### TC-0072: Reflect.transfer invokes Proxy transfer trap
- **Clause**: `sec-reflect.transfer` — Reflect.transfer
- **Step**: 2
- **Assertion**: Since `Reflect.transfer` calls `_target_.[[Transfer]]()`, it dispatches through the Proxy [[Transfer]] internal method when the target is a Proxy.
- **Category**: positive
- **Preconditions**: A Proxy with a transfer trap.
- **Test Description**: Create `let called = false; const p = new Proxy({}, { transfer() { called = true; return {}; } }); Reflect.transfer(p);`.
- **Expected**: `called` is `true`.

### TC-0073: Reflect.transfer is writable, non-enumerable, configurable
- **Clause**: `sec-reflect.transfer` — Reflect.transfer
- **Step**: (property attributes)
- **Assertion**: As a property of the Reflect object, it has attributes `{ [[Writable]]: true, [[Enumerable]]: false, [[Configurable]]: true }`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Call `Object.getOwnPropertyDescriptor(Reflect, "transfer")`.
- **Expected**: `{ value: Reflect.transfer, writable: true, enumerable: false, configurable: true }`.

### TC-0074: Reflect.transfer name is "transfer"
- **Clause**: `sec-reflect.transfer` — Reflect.transfer
- **Step**: (name property)
- **Assertion**: The `"name"` property is `"transfer"`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `Reflect.transfer.name`.
- **Expected**: `"transfer"`.

### TC-0075: Reflect.transfer length is 1
- **Clause**: `sec-reflect.transfer` — Reflect.transfer
- **Step**: (length property)
- **Assertion**: The `"length"` property is `1`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `Reflect.transfer.length`.
- **Expected**: `1`.

## Iterator.prototype [ %Symbol.transfer% ] (`sec-iterator.prototype-%symbol.transfer%`)

### TC-0076: Throws TypeError if this is not an Object
- **Clause**: `sec-iterator.prototype-%symbol.transfer%` — Iterator.prototype [ %Symbol.transfer% ] ( )
- **Step**: 1–2
- **Assertion**: If _O_ (the *this* value) is not an Object, throw a *TypeError*.
- **Category**: type-check
- **Preconditions**: Call the method with a primitive `this`.
- **Test Description**: Get the transfer method: `const fn = Iterator.prototype[Symbol.transfer];`. Call `fn.call(42)`, `fn.call("str")`, `fn.call(undefined)`.
- **Expected**: TypeError thrown for each.

### TC-0077: Throws TypeError if this has no [[Detached]] slot
- **Clause**: `sec-iterator.prototype-%symbol.transfer%` — Iterator.prototype [ %Symbol.transfer% ] ( )
- **Step**: 3
- **Assertion**: If _O_ does not have a [[Detached]] internal slot, throw a *TypeError*. This prevents non-built-in objects from using this method.
- **Category**: negative
- **Preconditions**: A plain object with `Iterator.prototype` in its prototype chain but no [[Detached]] slot.
- **Test Description**: Create `const obj = Object.create(Iterator.prototype); obj.next = () => ({ value: 1, done: false });`. Call `obj[Symbol.transfer]()`.
- **Expected**: TypeError thrown (no [[Detached]] slot).

### TC-0078: Throws TypeError if [[Detached]] is true (double transfer)
- **Clause**: `sec-iterator.prototype-%symbol.transfer%` — Iterator.prototype [ %Symbol.transfer% ] ( )
- **Step**: 4
- **Assertion**: If `_O_.[[Detached]]` is *true*, throw a *TypeError*. Prevents transferring an already-transferred iterator.
- **Category**: negative
- **Preconditions**: An iterator that has already been transferred once.
- **Test Description**: Create `const it = [1,2,3].values(); Object.transfer(it);`. Now call `it[Symbol.transfer]()` directly.
- **Expected**: TypeError thrown.

### TC-0079: Returns new iterator from CreateTransferredIterator
- **Clause**: `sec-iterator.prototype-%symbol.transfer%` — Iterator.prototype [ %Symbol.transfer% ] ( )
- **Step**: 5, 7
- **Assertion**: Returns the new iterator created by `CreateTransferredIterator(_O_)`.
- **Category**: positive
- **Preconditions**: A fresh array iterator.
- **Test Description**: Create `const it = [1,2,3].values(); const it2 = it[Symbol.transfer]();`. Call `it2.next()`.
- **Expected**: `it2.next()` returns `{ value: 1, done: false }`.

### TC-0080: Original is detached after [Symbol.transfer] call
- **Clause**: `sec-iterator.prototype-%symbol.transfer%` — Iterator.prototype [ %Symbol.transfer% ] ( )
- **Step**: 6 (NOTE)
- **Assertion**: After `[Symbol.transfer]()` returns, the original's [[Detached]] is *true*.
- **Category**: positive
- **Preconditions**: A transferred iterator.
- **Test Description**: Create `const it = [1,2,3].values(); it[Symbol.transfer]();`. Call `it.next()`.
- **Expected**: TypeError thrown (detached).

### TC-0081: Name property is "[Symbol.transfer]"
- **Clause**: `sec-iterator.prototype-%symbol.transfer%` — Iterator.prototype [ %Symbol.transfer% ] ( )
- **Step**: (name property)
- **Assertion**: The value of the `"name"` property is `"[Symbol.transfer]"`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `Iterator.prototype[Symbol.transfer].name`.
- **Expected**: `"[Symbol.transfer]"`.

### TC-0082: Length property is 0
- **Clause**: `sec-iterator.prototype-%symbol.transfer%` — Iterator.prototype [ %Symbol.transfer% ] ( )
- **Step**: (length property)
- **Assertion**: The function takes no arguments, so `"length"` is `0`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `Iterator.prototype[Symbol.transfer].length`.
- **Expected**: `0`.

## Iterator.prototype [ %Symbol.iterator% ] — Modified (`sec-iterator.prototype-%symbol.iterator%-modified`)

### TC-0083: Throws TypeError on detached iterator via [Symbol.iterator]
- **Clause**: `sec-iterator.prototype-%symbol.iterator%-modified` — Iterator.prototype [ %Symbol.iterator% ] ( )
- **Step**: 2
- **Assertion**: If _O_ is an Object, `EnsureNotDetached(_O_)` is called. A detached iterator throws TypeError.
- **Category**: negative
- **Preconditions**: A detached array iterator.
- **Test Description**: Create `const it = [1,2].values(); Object.transfer(it);`. Call `it[Symbol.iterator]()`.
- **Expected**: TypeError thrown.

### TC-0084: Returns this for non-detached iterator
- **Clause**: `sec-iterator.prototype-%symbol.iterator%-modified` — Iterator.prototype [ %Symbol.iterator% ] ( )
- **Step**: 2–3
- **Assertion**: If _O_ is an Object and not detached, returns _O_.
- **Category**: positive
- **Preconditions**: A fresh iterator.
- **Test Description**: Create `const it = [1,2].values(); const r = it[Symbol.iterator]();`.
- **Expected**: `r === it`.

### TC-0085: Does not throw for primitive this values
- **Clause**: `sec-iterator.prototype-%symbol.iterator%-modified` — Iterator.prototype [ %Symbol.iterator% ] ( )
- **Step**: 2–3
- **Assertion**: The detach check only runs when `_O_` is an Object. For primitives, it skips the check and returns _O_.
- **Category**: boundary
- **Preconditions**: None.
- **Test Description**: Call `Iterator.prototype[Symbol.iterator].call(42)`.
- **Expected**: Returns `42` (no throw).

### TC-0086: Detached iterator throws at for-of entry
- **Clause**: `sec-iterator.prototype-%symbol.iterator%-modified` — Iterator.prototype [ %Symbol.iterator% ] ( )
- **Step**: 2
- **Assertion**: A detached iterator throws at the start of `for...of` (which calls `[Symbol.iterator]()`) rather than on the first `.next()` call.
- **Category**: negative
- **Preconditions**: A detached iterator used in `for...of`.
- **Test Description**: Create `const it = [1,2,3].values(); Object.transfer(it);`. Try `for (const x of it) {}`.
- **Expected**: TypeError thrown immediately (before any loop body executes).

## %AsyncIteratorPrototype% [ %Symbol.transfer% ] (`sec-asynciterator.prototype-%symbol.transfer%`)

### TC-0087: Throws TypeError if this is not an Object
- **Clause**: `sec-asynciterator.prototype-%symbol.transfer%` — %AsyncIteratorPrototype% [ %Symbol.transfer% ] ( )
- **Step**: 1–2
- **Assertion**: If _O_ is not an Object, throw a *TypeError*.
- **Category**: type-check
- **Preconditions**: Call with a primitive this.
- **Test Description**: Get the async iterator prototype's transfer method. Call it with `this` = `42`.
- **Expected**: TypeError thrown.

### TC-0088: Throws TypeError if no [[Detached]] slot
- **Clause**: `sec-asynciterator.prototype-%symbol.transfer%` — %AsyncIteratorPrototype% [ %Symbol.transfer% ] ( )
- **Step**: 3
- **Assertion**: If _O_ does not have a [[Detached]] internal slot, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: An object inheriting from %AsyncIteratorPrototype% but without [[Detached]].
- **Test Description**: Create a plain async iterator object `const obj = { async next() { return { value: 1, done: false }; } };` with async iterator prototype. Call `obj[Symbol.transfer]()`.
- **Expected**: TypeError thrown.

### TC-0089: Throws TypeError if [[Detached]] is true
- **Clause**: `sec-asynciterator.prototype-%symbol.transfer%` — %AsyncIteratorPrototype% [ %Symbol.transfer% ] ( )
- **Step**: 4
- **Assertion**: If `_O_.[[Detached]]` is *true*, throw a *TypeError*.
- **Category**: negative
- **Preconditions**: A detached async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); Object.transfer(it);`. Call `it[Symbol.transfer]()`.
- **Expected**: TypeError thrown.

### TC-0090: Returns new async iterator
- **Clause**: `sec-asynciterator.prototype-%symbol.transfer%` — %AsyncIteratorPrototype% [ %Symbol.transfer% ] ( )
- **Step**: 5, 7
- **Assertion**: Returns the new iterator from CreateTransferredIterator.
- **Category**: positive
- **Preconditions**: A fresh async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); const it2 = Object.transfer(it); await it2.next();`.
- **Expected**: `{ value: 1, done: false }`.

### TC-0091: Name property is "[Symbol.transfer]"
- **Clause**: `sec-asynciterator.prototype-%symbol.transfer%` — %AsyncIteratorPrototype% [ %Symbol.transfer% ] ( )
- **Step**: (name property)
- **Assertion**: The `"name"` property is `"[Symbol.transfer]"`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Access the `[Symbol.transfer]` method on the async iterator prototype and check its `.name`.
- **Expected**: `"[Symbol.transfer]"`.

## %AsyncIteratorPrototype% [ %Symbol.asyncIterator% ] — Modified (`sec-asynciterator.prototype-%symbol.asynciterator%-modified`)

### TC-0092: Throws TypeError on detached async iterator via [Symbol.asyncIterator]
- **Clause**: `sec-asynciterator.prototype-%symbol.asynciterator%-modified` — %AsyncIteratorPrototype% [ %Symbol.asyncIterator% ] ( )
- **Step**: 2
- **Assertion**: If _O_ is an Object, `EnsureNotDetached(_O_)` is called. A detached async iterator throws TypeError.
- **Category**: negative
- **Preconditions**: A detached async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); Object.transfer(it);`. Call `it[Symbol.asyncIterator]()`.
- **Expected**: TypeError thrown.

### TC-0093: Returns this for non-detached async iterator
- **Clause**: `sec-asynciterator.prototype-%symbol.asynciterator%-modified` — %AsyncIteratorPrototype% [ %Symbol.asyncIterator% ] ( )
- **Step**: 2���3
- **Assertion**: If not detached, returns _O_.
- **Category**: positive
- **Preconditions**: A fresh async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); const r = it[Symbol.asyncIterator]();`.
- **Expected**: `r === it`.

### TC-0094: Detached async iterator throws at for-await-of entry
- **Clause**: `sec-asynciterator.prototype-%symbol.asynciterator%-modified` — %AsyncIteratorPrototype% [ %Symbol.asyncIterator% ] ( )
- **Step**: 2
- **Assertion**: A detached async iterator throws at the start of `for await...of` rather than on `.next()`.
- **Category**: negative
- **Preconditions**: A detached async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); Object.transfer(it);`. Try `for await (const x of it) {}`.
- **Expected**: TypeError thrown immediately.

## Iterator.prototype [ %Symbol.dispose% ] — Modified (`sec-iterator.prototype-%symbol.dispose%-modified`)

### TC-0095: Returns undefined for detached iterator (no-op)
- **Clause**: `sec-iterator.prototype-%symbol.dispose%-modified` — Iterator.prototype [ %Symbol.dispose% ] ( )
- **Step**: 1
- **Assertion**: If _O_ is an Object, has [[Detached]], and [[Detached]] is *true*, return *undefined* immediately.
- **Category**: positive
- **Preconditions**: A detached iterator.
- **Test Description**: Create `const it = [1,2].values(); Object.transfer(it);`. Call `it[Symbol.dispose]()`.
- **Expected**: Returns `undefined` without throwing.

### TC-0096: Does not call .return() on detached iterator
- **Clause**: `sec-iterator.prototype-%symbol.dispose%-modified` — Iterator.prototype [ %Symbol.dispose% ] ( )
- **Step**: 1
- **Assertion**: The early return for detached iterators skips the `GetMethod(_O_, "return")` and `Call` steps.
- **Category**: observable-ordering
- **Preconditions**: A detached iterator with a `.return()` method that records calls.
- **Test Description**: Create `let called = false; const it = [1,2].values(); it.return = function() { called = true; return { done: true }; }; Object.transfer(it);`. Call `it[Symbol.dispose]()`. Verify `called` is `false`.
- **Expected**: `called` is `false`; `.return()` was never invoked.

### TC-0097: Calls .return() on non-detached iterator
- **Clause**: `sec-iterator.prototype-%symbol.dispose%-modified` — Iterator.prototype [ %Symbol.dispose% ] ( )
- **Step**: 2–4
- **Assertion**: If the iterator is not detached and has a `.return()` method, it is called.
- **Category**: positive
- **Preconditions**: A non-detached iterator.
- **Test Description**: Create `function* g() { yield 1; } const it = g();`. Call `it[Symbol.dispose]()`. Verify the generator is now done: `it.next()` returns `{ value: undefined, done: true }`.
- **Expected**: Generator is completed after disposal.

### TC-0098: Disposal returns undefined for non-detached iterator
- **Clause**: `sec-iterator.prototype-%symbol.dispose%-modified` — Iterator.prototype [ %Symbol.dispose% ] ( )
- **Step**: 5
- **Assertion**: Returns *undefined* after calling `.return()`.
- **Category**: positive
- **Preconditions**: A non-detached generator.
- **Test Description**: Create `function* g() { yield 1; } const it = g(); const r = it[Symbol.dispose]();`.
- **Expected**: `r` is `undefined`.

### TC-0099: Disposal works for `using` with transferred iterator
- **Clause**: `sec-iterator.prototype-%symbol.dispose%-modified` — Iterator.prototype [ %Symbol.dispose% ] ( )
- **Step**: 1
- **Assertion**: A `using` binding holding a transferred iterator silently no-ops at block exit.
- **Category**: positive
- **Preconditions**: An iterator bound with `using`, then transferred within the block.
- **Test Description**: `{ using it = [1,2,3].values(); const it2 = Object.transfer(it); /* block exits, it[Symbol.dispose]() called on detached iterator */ }`. No error should be thrown.
- **Expected**: Block exits cleanly; no TypeError from disposal of detached iterator.

## %AsyncIteratorPrototype% [ %Symbol.asyncDispose% ] — Modified (`sec-asynciterator.prototype-%symbol.asyncdispose%-modified`)

### TC-0100: Returns undefined for detached async iterator (no-op)
- **Clause**: `sec-asynciterator.prototype-%symbol.asyncdispose%-modified` — %AsyncIteratorPrototype% [ %Symbol.asyncDispose% ] ( )
- **Step**: 1
- **Assertion**: If _O_ is an Object, has [[Detached]], and [[Detached]] is *true*, return *undefined*.
- **Category**: positive
- **Preconditions**: A detached async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); Object.transfer(it);`. Call `await it[Symbol.asyncDispose]()`.
- **Expected**: Returns `undefined` without throwing.

### TC-0101: Does not call .return() on detached async iterator
- **Clause**: `sec-asynciterator.prototype-%symbol.asyncdispose%-modified` — %AsyncIteratorPrototype% [ %Symbol.asyncDispose% ] ( )
- **Step**: 1
- **Assertion**: The early return skips `GetMethod` and `Call` steps.
- **Category**: observable-ordering
- **Preconditions**: A detached async generator.
- **Test Description**: Create `let called = false; async function* ag() { yield 1; } const it = ag(); const origReturn = it.return; it.return = async function(...args) { called = true; return origReturn.apply(this, args); }; Object.transfer(it);`. Call `await it[Symbol.asyncDispose]()`. Verify `called` is `false`.
- **Expected**: `called` is `false`.

### TC-0102: Calls .return() and awaits result for non-detached async iterator
- **Clause**: `sec-asynciterator.prototype-%symbol.asyncdispose%-modified` — %AsyncIteratorPrototype% [ %Symbol.asyncDispose% ] ( )
- **Step**: 2–5
- **Assertion**: If not detached, calls `.return()` and awaits the result.
- **Category**: positive
- **Preconditions**: A non-detached async generator.
- **Test Description**: Create `async function* ag() { yield 1; yield 2; } const it = ag(); await it.next();`. Call `await it[Symbol.asyncDispose]()`. Then `await it.next()`.
- **Expected**: After disposal, `it.next()` returns `{ value: undefined, done: true }`.

## Array Iterator Modifications (`sec-array-iterator-modifications`)

### TC-0103: CreateArrayIterator creates iterator with [[Detached]] false
- **Clause**: `sec-createarrayiterator-modified` — CreateArrayIterator
- **Step**: 1, 6
- **Assertion**: The array iterator created by `CreateArrayIterator` has [[Detached]] initialized to *false*.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Create `const it = [1].values();`. Verify `.next()` works (confirming [[Detached]] is false, since the detach guard would throw if it were true).
- **Expected**: `it.next()` returns `{ value: 1, done: false }`.

### TC-0104: Array iterator .next() throws TypeError when detached
- **Clause**: `sec-arrayiteratorprototype-next-modified` — %ArrayIteratorPrototype%.next ( )
- **Step**: 4
- **Assertion**: `EnsureNotDetached(_O_)` is called; throws if detached.
- **Category**: negative
- **Preconditions**: A detached array iterator.
- **Test Description**: Create `const it = [1,2,3].values(); Object.transfer(it);`. Call `it.next()`.
- **Expected**: TypeError thrown.

### TC-0105: Array iterator .next() detach check is after internal slot check
- **Clause**: `sec-arrayiteratorprototype-next-modified` — %ArrayIteratorPrototype%.next ( )
- **Step**: 3–4
- **Assertion**: The internal slot check (step 3) happens before the detach guard (step 4). Calling `.next()` on a non-array-iterator throws TypeError from the slot check, not the detach check.
- **Category**: observable-ordering
- **Preconditions**: A non-iterator object.
- **Test Description**: Create `const arrayIterProto = Object.getPrototypeOf([].values()); arrayIterProto.next.call({})`. This should throw from the internal slot check.
- **Expected**: TypeError thrown (from missing internal slots, not from detach).

## Generator-Based Iterator Modifications (`sec-generator-based-iterator-modifications`)

### TC-0106: CreateIteratorFromClosure creates with [[Detached]] false
- **Clause**: `sec-createiteratorfromclosure-modified` — CreateIteratorFromClosure
- **Step**: 2, 7
- **Assertion**: The iterator has [[Detached]] initialized to *false*.
- **Category**: positive
- **Preconditions**: A Map iterator (created via CreateIteratorFromClosure).
- **Test Description**: Create `const m = new Map([[1, "a"]]); const it = m.values(); it.next();`.
- **Expected**: `{ value: "a", done: false }` (confirms iterator works, [[Detached]] is false).

### TC-0107: GeneratorValidate throws TypeError when detached (Map iterator)
- **Clause**: `sec-generatorvalidate-modified` — GeneratorValidate
- **Step**: 4
- **Assertion**: `EnsureNotDetached(_generator_)` throws TypeError on a detached generator-based iterator.
- **Category**: negative
- **Preconditions**: A detached Map iterator.
- **Test Description**: Create `const m = new Map([[1, "a"], [2, "b"]]); const it = m.values(); Object.transfer(it);`. Call `it.next()`.
- **Expected**: TypeError thrown.

### TC-0108: GeneratorValidate detach check is after brand check
- **Clause**: `sec-generatorvalidate-modified` — GeneratorValidate
- **Step**: 1–4
- **Assertion**: The brand check (step 3) happens before the detach check (step 4). A wrong-brand generator throws from the brand check, not the detach check.
- **Category**: observable-ordering
- **Preconditions**: A Map iterator whose `.next()` is called with a Set iterator as `this`.
- **Test Description**: Create `const mapIt = new Map().values(); const setIt = new Set().values(); Object.transfer(setIt);`. Call `Map.prototype.values.call(setIt).next()` — actually, the brand check is on the iterator prototype's `.next()`. Call the Map iterator's `.next` with a detached Set iterator as `this`: `const mapNext = Object.getPrototypeOf(new Map().values()).next; mapNext.call(setIt);`.
- **Expected**: TypeError thrown (from brand mismatch, since Set iterators have a different brand than Map iterators).

### TC-0109: Set iterator transfer and detach
- **Clause**: `sec-generatorvalidate-modified` — GeneratorValidate
- **Step**: 4
- **Assertion**: Set iterators (also CreateIteratorFromClosure-based) support transfer and detach.
- **Category**: positive
- **Preconditions**: A Set iterator.
- **Test Description**: Create `const s = new Set([1, 2, 3]); const it = s.values(); it.next(); const it2 = Object.transfer(it); it2.next();`.
- **Expected**: `it2.next()` returns `{ value: 2, done: false }`. Calling `it.next()` throws TypeError.

### TC-0110: String iterator transfer and detach
- **Clause**: `sec-generatorvalidate-modified` — GeneratorValidate
- **Step**: 4
- **Assertion**: String iterators (CreateIteratorFromClosure-based) support transfer and detach.
- **Category**: positive
- **Preconditions**: A String iterator.
- **Test Description**: Create `const it = "abc"[Symbol.iterator](); it.next(); const it2 = Object.transfer(it); it2.next();`.
- **Expected**: `it2.next()` returns `{ value: "b", done: false }`. Calling `it.next()` throws TypeError.

## RegExp String Iterator Modifications (`sec-regexp-string-iterator-modifications`)

### TC-0111: CreateRegExpStringIterator creates with [[Detached]] false
- **Clause**: `sec-createregexpstringiterator-modified` ��� CreateRegExpStringIterator
- **Step**: 1, 7
- **Assertion**: The RegExp string iterator has [[Detached]] initialized to *false*.
- **Category**: positive
- **Preconditions**: A RegExp string iterator from `.matchAll()`.
- **Test Description**: Create `const it = "abc".matchAll(/./g); it.next();`.
- **Expected**: Returns a match result for "a" (works normally).

### TC-0112: RegExp string iterator .next() throws TypeError when detached
- **Clause**: `sec-regexpstringiteratorprototype-next-modified` — %RegExpStringIteratorPrototype%.next ( )
- **Step**: 4
- **Assertion**: `EnsureNotDetached(_O_)` throws TypeError on a detached RegExp string iterator.
- **Category**: negative
- **Preconditions**: A detached RegExp string iterator.
- **Test Description**: Create `const it = "abc".matchAll(/./g); Object.transfer(it);`. Call `it.next()`.
- **Expected**: TypeError thrown.

### TC-0113: RegExp string iterator detach check is after internal slot check
- **Clause**: `sec-regexpstringiteratorprototype-next-modified` — %RegExpStringIteratorPrototype%.next ( )
- **Step**: 3–4
- **Assertion**: The internal slot check (step 3) happens before the detach guard (step 4).
- **Category**: observable-ordering
- **Preconditions**: A non-RegExp-string-iterator object.
- **Test Description**: Call the `.next()` method of `%RegExpStringIteratorPrototype%` with a plain object as `this`.
- **Expected**: TypeError thrown (from missing internal slots).

## WrapForValidIterator Modifications (`sec-wrapforvaliditerator-modifications`)

### TC-0114: Iterator.from wrapper has [[Detached]] false
- **Clause**: `sec-iterator.from-modified` — Iterator.from ( _O_ )
- **Step**: 4–6
- **Assertion**: The wrapper created by `Iterator.from()` (when the argument is not an instance of %Iterator%) has [[Detached]] set to *false*.
- **Category**: positive
- **Preconditions**: A plain iterator object (not inheriting from Iterator.prototype).
- **Test Description**: Create `const raw = { next() { return { value: 1, done: false }; } }; const it = Iterator.from(raw); it.next();`.
- **Expected**: `{ value: 1, done: false }` (works normally).

### TC-0115: Iterator.from does not add [[Detached]] when argument is already an Iterator
- **Clause**: `sec-iterator.from-modified` — Iterator.from ( _O_ )
- **Step**: 2–3
- **Assertion**: If `OrdinaryHasInstance(%Iterator%, ...)` is *true*, the original iterator is returned directly without wrapping. No [[Detached]] is added by `Iterator.from` itself (the original already has it from its creation).
- **Category**: positive
- **Preconditions**: An array iterator (which is already an Iterator instance).
- **Test Description**: Create `const it = [1,2].values(); const it2 = Iterator.from(it);`.
- **Expected**: `it2 === it` (same object, no wrapping).

### TC-0116: Wrapper .next() throws TypeError when detached
- **Clause**: `sec-wrapforvaliditeratorprototype-next-modified` — %WrapForValidIteratorPrototype%.next ( )
- **Step**: 3
- **Assertion**: `EnsureNotDetached(_O_)` throws TypeError on a detached wrapper.
- **Category**: negative
- **Preconditions**: A transferred WrapForValidIterator.
- **Test Description**: Create `const it = Iterator.from({ next() { return { value: 1, done: false }; } }); Object.transfer(it);`. Call `it.next()`.
- **Expected**: TypeError thrown.

### TC-0117: Wrapper .return() throws TypeError when detached
- **Clause**: `sec-wrapforvaliditeratorprototype-return-modified` — %WrapForValidIteratorPrototype%.return ( )
- **Step**: 3
- **Assertion**: `EnsureNotDetached(_O_)` throws TypeError on a detached wrapper.
- **Category**: negative
- **Preconditions**: A transferred WrapForValidIterator.
- **Test Description**: Create `const it = Iterator.from({ next() { return { value: 1, done: false }; }, return() { return { done: true }; } }); Object.transfer(it);`. Call `it.return()`.
- **Expected**: TypeError thrown.

### TC-0118: Wrapper .return() works when not detached
- **Clause**: `sec-wrapforvaliditeratorprototype-return-modified` — %WrapForValidIteratorPrototype%.return ( )
- **Step**: 4–8
- **Assertion**: When not detached, `.return()` delegates to the underlying iterator's `.return()` method if present.
- **Category**: positive
- **Preconditions**: A non-detached WrapForValidIterator wrapping an iterator with `.return()`.
- **Test Description**: Create `let called = false; const it = Iterator.from({ next() { return { value: 1, done: false }; }, return() { called = true; return { value: undefined, done: true }; } }); it.return();`.
- **Expected**: `called` is `true`.

### TC-0119: Wrapper .return() returns done result when underlying has no .return()
- **Clause**: `sec-wrapforvaliditeratorprototype-return-modified` — %WrapForValidIteratorPrototype%.return ( )
- **Step**: 6–7
- **Assertion**: If `GetMethod(_iterator_, "return")` returns *undefined*, return `CreateIteratorResultObject(undefined, true)`.
- **Category**: positive
- **Preconditions**: A WrapForValidIterator wrapping an iterator without `.return()`.
- **Test Description**: Create `const it = Iterator.from({ next() { return { value: 1, done: false }; } }); const r = it.return();`.
- **Expected**: `r` is `{ value: undefined, done: true }`.

## Generator Objects (`sec-generator-object-modifications`)

### TC-0120: User generator has [[Detached]] slot initialized to false
- **Clause**: `sec-evaluategeneratorbody-modified` — RS: EvaluateGeneratorBody
- **Step**: (slot list modification)
- **Assertion**: Generator instances created by `EvaluateGeneratorBody` include [[Detached]] in their slot list, initialized to *false*.
- **Category**: positive
- **Preconditions**: A user-defined generator function.
- **Test Description**: Create `function* g() { yield 1; } const it = g(); it.next();`.
- **Expected**: `{ value: 1, done: false }` (works normally — [[Detached]] is false).

### TC-0121: User generator can be transferred
- **Clause**: `sec-generator-object-modifications` — Generator Objects
- **Step**: (transfer support)
- **Assertion**: User generators have [[Detached]], so they are transferable via the Iterator.prototype[Symbol.transfer] method.
- **Category**: positive
- **Preconditions**: A user-defined generator.
- **Test Description**: Create `function* g() { yield 1; yield 2; yield 3; } const it = g(); it.next(); const it2 = Object.transfer(it); it2.next();`.
- **Expected**: `it2.next()` returns `{ value: 2, done: false }`.

### TC-0122: User generator .next() throws TypeError when detached
- **Clause**: `sec-generatorvalidate-modified` — GeneratorValidate
- **Step**: 4
- **Assertion**: GeneratorValidate's detach guard applies to user generators.
- **Category**: negative
- **Preconditions**: A detached user generator.
- **Test Description**: Create `function* g() { yield 1; } const it = g(); Object.transfer(it); it.next();`.
- **Expected**: TypeError thrown.

### TC-0123: User generator .return() throws TypeError when detached
- **Clause**: `sec-generatorvalidate-modified` — GeneratorValidate
- **Step**: 4
- **Assertion**: GeneratorValidate is called by GeneratorResume, which is called by `.return()`. A detached generator's `.return()` throws.
- **Category**: negative
- **Preconditions**: A detached user generator.
- **Test Description**: Create `function* g() { yield 1; } const it = g(); Object.transfer(it); it.return();`.
- **Expected**: TypeError thrown.

### TC-0124: User generator .throw() throws TypeError when detached
- **Clause**: `sec-generatorvalidate-modified` — GeneratorValidate
- **Step**: 4
- **Assertion**: `.throw()` also goes through GeneratorValidate. A detached generator's `.throw()` throws TypeError (not the passed-in error).
- **Category**: negative
- **Preconditions**: A detached user generator.
- **Test Description**: Create `function* g() { yield 1; } const it = g(); Object.transfer(it); it.throw(new Error("test"));`.
- **Expected**: TypeError thrown (from detach guard, not the `Error("test")`).

## AsyncGenerator Objects (`sec-asyncgenerator-object-modifications`)

### TC-0125: User async generator has [[Detached]] slot initialized to false
- **Clause**: `sec-evaluateasyncgeneratorbody-modified` — RS: EvaluateAsyncGeneratorBody
- **Step**: (slot list modification)
- **Assertion**: AsyncGenerator instances include [[Detached]], initialized to *false*.
- **Category**: positive
- **Preconditions**: A user-defined async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); await it.next();`.
- **Expected**: `{ value: 1, done: false }`.

### TC-0126: User async generator can be transferred
- **Clause**: `sec-asyncgenerator-object-modifications` — AsyncGenerator Objects
- **Step**: (transfer support)
- **Assertion**: Async generators have [[Detached]] and are transferable via %AsyncIteratorPrototype%[Symbol.transfer].
- **Category**: positive
- **Preconditions**: A user-defined async generator.
- **Test Description**: Create `async function* ag() { yield 1; yield 2; } const it = ag(); await it.next(); const it2 = Object.transfer(it); await it2.next();`.
- **Expected**: `{ value: 2, done: false }`.

### TC-0127: AsyncGeneratorValidate throws TypeError when detached
- **Clause**: `sec-asyncgeneratorvalidate-modified` — AsyncGeneratorValidate
- **Step**: 5
- **Assertion**: `EnsureNotDetached(_generator_)` throws TypeError on a detached async generator.
- **Category**: negative
- **Preconditions**: A detached async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); Object.transfer(it);`. Call `it.next()`.
- **Expected**: The returned promise rejects with TypeError.

### TC-0128: AsyncGeneratorValidate detach check is after brand check
- **Clause**: `sec-asyncgeneratorvalidate-modified` — AsyncGeneratorValidate
- **Step**: 1–5
- **Assertion**: The slot checks (steps 1–3) and brand check (step 4) happen before the detach check (step 5).
- **Category**: observable-ordering
- **Preconditions**: An object without async generator internal slots.
- **Test Description**: Get the async generator prototype's `.next` method and call it with a plain object as `this`.
- **Expected**: TypeError thrown (from RequireInternalSlot, not from detach check).

### TC-0129: User async generator .return() rejects when detached
- **Clause**: `sec-asyncgeneratorvalidate-modified` — AsyncGeneratorValidate
- **Step**: 5
- **Assertion**: `.return()` also goes through AsyncGeneratorValidate.
- **Category**: negative
- **Preconditions**: A detached async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); Object.transfer(it); await it.return();`.
- **Expected**: Promise rejects with TypeError.

### TC-0130: User async generator .throw() rejects when detached
- **Clause**: `sec-asyncgeneratorvalidate-modified` — AsyncGeneratorValidate
- **Step**: 5
- **Assertion**: `.throw()` also goes through AsyncGeneratorValidate.
- **Category**: negative
- **Preconditions**: A detached async generator.
- **Test Description**: Create `async function* ag() { yield 1; } const it = ag(); Object.transfer(it); await it.throw(new Error("test"));`.
- **Expected**: Promise rejects with TypeError (from detach guard).

## ArrayBuffer.prototype [ %Symbol.transfer% ] (`sec-arraybuffer.prototype-%symbol.transfer%`)

### TC-0131: Transfers buffer data to new ArrayBuffer
- **Clause**: `sec-arraybuffer.prototype-%symbol.transfer%` — ArrayBuffer.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: Delegates to `ArrayBufferCopyAndDetach(_O_, undefined, ~preserve-resizability~)`. The new ArrayBuffer has the same data.
- **Category**: positive
- **Preconditions**: An ArrayBuffer with data.
- **Test Description**: Create `const buf = new ArrayBuffer(4); new Uint8Array(buf).set([1, 2, 3, 4]); const buf2 = Object.transfer(buf);`. Read `new Uint8Array(buf2)`.
- **Expected**: `Uint8Array [1, 2, 3, 4]`.

### TC-0132: Original buffer is detached after transfer
- **Clause**: `sec-arraybuffer.prototype-%symbol.transfer%` — ArrayBuffer.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: `ArrayBufferCopyAndDetach` detaches the original buffer.
- **Category**: positive
- **Preconditions**: An ArrayBuffer that has been transferred.
- **Test Description**: Create `const buf = new ArrayBuffer(4); Object.transfer(buf);`. Check `buf.byteLength`.
- **Expected**: `buf.byteLength` is `0` (detached).

### TC-0133: New buffer has same byte length
- **Clause**: `sec-arraybuffer.prototype-%symbol.transfer%` — ArrayBuffer.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: The new ArrayBuffer has the same byte length as the original.
- **Category**: positive
- **Preconditions**: An ArrayBuffer.
- **Test Description**: Create `const buf = new ArrayBuffer(16); const buf2 = Object.transfer(buf);`.
- **Expected**: `buf2.byteLength === 16`.

### TC-0134: Preserves resizability
- **Clause**: `sec-arraybuffer.prototype-%symbol.transfer%` — ArrayBuffer.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: The `~preserve-resizability~` argument to `ArrayBufferCopyAndDetach` preserves whether the buffer is resizable.
- **Category**: positive
- **Preconditions**: A resizable ArrayBuffer.
- **Test Description**: Create `const buf = new ArrayBuffer(4, { maxByteLength: 16 }); const buf2 = Object.transfer(buf);`.
- **Expected**: `buf2.resizable` is `true` and `buf2.maxByteLength === 16`.

### TC-0135: Matches behavior of ArrayBuffer.prototype.transfer()
- **Clause**: `sec-arraybuffer.prototype-%symbol.transfer%` — ArrayBuffer.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: `[Symbol.transfer]()` delegates to the same AO as `.transfer()`. They should produce equivalent results.
- **Category**: positive
- **Preconditions**: Two identical ArrayBuffers.
- **Test Description**: Create two identical buffers: `const buf1 = new ArrayBuffer(4); new Uint8Array(buf1).set([1,2,3,4]); const buf2 = new ArrayBuffer(4); new Uint8Array(buf2).set([1,2,3,4]);`. Transfer one with `.transfer()` and one with `Object.transfer()`. Compare results.
- **Expected**: Both new buffers have identical byte length and data.

### TC-0136: Name property is "[Symbol.transfer]"
- **Clause**: `sec-arraybuffer.prototype-%symbol.transfer%` — ArrayBuffer.prototype [ %Symbol.transfer% ] ( )
- **Step**: (name property)
- **Assertion**: The `"name"` property is `"[Symbol.transfer]"`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `ArrayBuffer.prototype[Symbol.transfer].name`.
- **Expected**: `"[Symbol.transfer]"`.

### TC-0137: Throws on detached ArrayBuffer
- **Clause**: `sec-arraybuffer.prototype-%symbol.transfer%` — ArrayBuffer.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: `ArrayBufferCopyAndDetach` throws TypeError when called on an already-detached buffer.
- **Category**: negative
- **Preconditions**: A detached ArrayBuffer.
- **Test Description**: Create `const buf = new ArrayBuffer(4); buf.transfer();`. Call `Object.transfer(buf)`.
- **Expected**: TypeError thrown (from ArrayBufferCopyAndDetach on detached buffer).

### TC-0138: Throws on SharedArrayBuffer
- **Clause**: `sec-arraybuffer.prototype-%symbol.transfer%` — ArrayBuffer.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: `ArrayBufferCopyAndDetach` rejects SharedArrayBuffers.
- **Category**: negative
- **Preconditions**: A SharedArrayBuffer.
- **Test Description**: Create `const sab = new SharedArrayBuffer(4);`. Try to call `ArrayBuffer.prototype[Symbol.transfer].call(sab)`.
- **Expected**: TypeError thrown.

## DisposableStack.prototype [ %Symbol.transfer% ] (`sec-disposablestack.prototype-%symbol.transfer%`)

### TC-0139: Throws TypeError if this lacks [[DisposableState]]
- **Clause**: `sec-disposablestack.prototype-%symbol.transfer%` — DisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: `RequireInternalSlot(_O_, [[DisposableState]])` throws TypeError for non-DisposableStack objects.
- **Category**: type-check
- **Preconditions**: A plain object.
- **Test Description**: Call `DisposableStack.prototype[Symbol.transfer].call({})`.
- **Expected**: TypeError thrown.

### TC-0140: Throws ReferenceError if disposed
- **Clause**: `sec-disposablestack.prototype-%symbol.transfer%` — DisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 3
- **Assertion**: If `_O_.[[DisposableState]]` is ~disposed~, throw a *ReferenceError*.
- **Category**: negative
- **Preconditions**: A DisposableStack that has been disposed.
- **Test Description**: Create `const stack = new DisposableStack(); stack[Symbol.dispose]();`. Call `Object.transfer(stack)`.
- **Expected**: ReferenceError thrown.

### TC-0141: Calls .move() on the stack
- **Clause**: `sec-disposablestack.prototype-%symbol.transfer%` — DisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 4–5
- **Assertion**: Gets the `"move"` method via `Get(_O_, "move")` and calls it.
- **Category**: observable-ordering
- **Preconditions**: A DisposableStack with a tracked resource.
- **Test Description**: Create `let moveCalled = false; const stack = new DisposableStack(); stack.use({ [Symbol.dispose]() {} }); const origMove = stack.move; stack.move = function() { moveCalled = true; return origMove.call(this); };`. Call `Object.transfer(stack)`. Verify `moveCalled` is `true`.
- **Expected**: `moveCalled` is `true`.

### TC-0142: Returns result of .move()
- **Clause**: `sec-disposablestack.prototype-%symbol.transfer%` — DisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 5
- **Assertion**: The return value is the result of `Call(_moveMethod_, _O_)`.
- **Category**: positive
- **Preconditions**: A DisposableStack.
- **Test Description**: Create `const stack = new DisposableStack(); let disposed = false; stack.defer(() => { disposed = true; }); const stack2 = Object.transfer(stack); stack2[Symbol.dispose]();`.
- **Expected**: `disposed` is `true` (the resource moved to the new stack).

### TC-0143: Original is disposed after transfer
- **Clause**: `sec-disposablestack.prototype-%symbol.transfer%` — DisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 5
- **Assertion**: After `.move()` is called, the original stack's [[DisposableState]] is ~disposed~.
- **Category**: positive
- **Preconditions**: A DisposableStack that has been transferred.
- **Test Description**: Create `const stack = new DisposableStack(); Object.transfer(stack);`. Call `stack.use({})`.
- **Expected**: ReferenceError thrown (stack is disposed).

### TC-0144: Name property is "[Symbol.transfer]"
- **Clause**: `sec-disposablestack.prototype-%symbol.transfer%` — DisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: (name property)
- **Assertion**: The `"name"` property is `"[Symbol.transfer]"`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `DisposableStack.prototype[Symbol.transfer].name`.
- **Expected**: `"[Symbol.transfer]"`.

### TC-0145: Get("move") is observable (Proxy interception)
- **Clause**: `sec-disposablestack.prototype-%symbol.transfer%` — DisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 4
- **Assertion**: The `Get(_O_, "move")` is a dynamic property access, so it can be intercepted by a Proxy.
- **Category**: observable-ordering
- **Preconditions**: A DisposableStack wrapped in a Proxy that intercepts `get`.
- **Test Description**: Create a DisposableStack, wrap it in a Proxy that logs property accesses. Call `[Symbol.transfer]()` on the proxy via `DisposableStack.prototype[Symbol.transfer].call(proxy)`. Verify `"move"` appears in the access log.
- **Expected**: The `"move"` property access is observable.

## AsyncDisposableStack.prototype [ %Symbol.transfer% ] (`sec-asyncdisposablestack.prototype-%symbol.transfer%`)

### TC-0146: Throws TypeError if this lacks [[AsyncDisposableState]]
- **Clause**: `sec-asyncdisposablestack.prototype-%symbol.transfer%` — AsyncDisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 2
- **Assertion**: `RequireInternalSlot(_O_, [[AsyncDisposableState]])` throws TypeError for non-AsyncDisposableStack objects.
- **Category**: type-check
- **Preconditions**: A plain object.
- **Test Description**: Call `AsyncDisposableStack.prototype[Symbol.transfer].call({})`.
- **Expected**: TypeError thrown.

### TC-0147: Throws ReferenceError if disposed
- **Clause**: `sec-asyncdisposablestack.prototype-%symbol.transfer%` — AsyncDisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 3
- **Assertion**: If `_O_.[[AsyncDisposableState]]` is ~disposed~, throw a *ReferenceError*.
- **Category**: negative
- **Preconditions**: A disposed AsyncDisposableStack.
- **Test Description**: Create `const stack = new AsyncDisposableStack(); await stack[Symbol.asyncDispose]();`. Call `Object.transfer(stack)`.
- **Expected**: ReferenceError thrown.

### TC-0148: Calls .move() on the stack
- **Clause**: `sec-asyncdisposablestack.prototype-%symbol.transfer%` — AsyncDisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 4–5
- **Assertion**: Gets `"move"` method and calls it.
- **Category**: observable-ordering
- **Preconditions**: An AsyncDisposableStack.
- **Test Description**: Create `let moveCalled = false; const stack = new AsyncDisposableStack(); const origMove = stack.move; stack.move = function() { moveCalled = true; return origMove.call(this); };`. Call `Object.transfer(stack)`. Verify `moveCalled`.
- **Expected**: `moveCalled` is `true`.

### TC-0149: Returns result of .move()
- **Clause**: `sec-asyncdisposablestack.prototype-%symbol.transfer%` — AsyncDisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: 5
- **Assertion**: Returns the new AsyncDisposableStack from `.move()`.
- **Category**: positive
- **Preconditions**: An AsyncDisposableStack with a resource.
- **Test Description**: Create `const stack = new AsyncDisposableStack(); let disposed = false; stack.defer(async () => { disposed = true; }); const stack2 = Object.transfer(stack); await stack2[Symbol.asyncDispose]();`.
- **Expected**: `disposed` is `true`.

### TC-0150: Name property is "[Symbol.transfer]"
- **Clause**: `sec-asyncdisposablestack.prototype-%symbol.transfer%` — AsyncDisposableStack.prototype [ %Symbol.transfer% ] ( )
- **Step**: (name property)
- **Assertion**: The `"name"` property is `"[Symbol.transfer]"`.
- **Category**: positive
- **Preconditions**: None.
- **Test Description**: Check `AsyncDisposableStack.prototype[Symbol.transfer].name`.
- **Expected**: `"[Symbol.transfer]"`.

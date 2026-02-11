# Cross-Realm Iterator Transfer via `postMessage()`

A mechanism for transferring `Iterator` and `AsyncIterator` objects across
realm boundaries using `postMessage()`, modeled on [`ReadableStream`
transfer][rs-transfer] in the WHATWG Streams specification.

[rs-transfer]: https://streams.spec.whatwg.org/#rs-transfer

## Background

`ReadableStream`, `WritableStream`, and `TransformStream` are
[`[Transferable]`][transferable]. When transferred via `postMessage()`, the
structured clone algorithm creates a `MessagePort` pair. One port stays in
the sending realm, the other goes to the receiving realm. The ports
communicate using a message protocol (`"chunk"`, `"close"`, `"error"`,
`"pull"`) that bridges producer and consumer across the boundary.

This document applies the same pattern to iterators.

[transferable]: https://html.spec.whatwg.org/multipage/structured-data.html#transferable

## Design Decisions

1. **Sync `Iterator` becomes `AsyncIterator` on the receiving side.**
   `MessagePort` communication is asynchronous. Every `.next()` call
   requires a round-trip message, so a sync `Iterator` is always received
   as an `AsyncIterator`.

2. **Structured-cloneable values can be passed to `next()`.**
   `next(value)`, `return(value)`, and `throw(error)` all accept a value
   argument. These values must be structured-cloneable.

3. **`Iterator` and `AsyncIterator` are `[Transferable]`.**
   Both participate directly in the structured clone transfer steps, same
   as `ReadableStream`. No explicit wrapping step is needed.

4. **No buffering or prefetch.**
   Each `next()` is a single request-response round-trip. No internal
   queue, no high water mark, no read-ahead. Use `ReadableStream` if
   buffering is needed.

5. **Disentangle on `done: true`, with source-side `return()`.**
   When the iterator exhausts, the source side calls `iterator.return()`
   (if callable) for resource cleanup, then both sides disentangle. The
   consumer-side `return()` is a no-op after exhaustion. This ensures
   non-generator iterators with cleanup in `return()` are properly
   finalized, and makes `await using` work correctly (the disposal call
   to `return()` is harmless since cleanup already ran on the source).

6. **Concurrent `next()` calls are queued.**
   Multiple `next()` calls without awaiting are allowed, matching regular
   `AsyncIterator` behavior. Each call returns a Promise and sends a
   request to the source; responses resolve queued promises in order. If
   the iterator exhausts or errors mid-queue, remaining promises are
   resolved with `{ value: undefined, done: true }`. The source side
   serializes request processing (sync sources do this naturally; async
   sources use a request queue).

7. **Duck-typed iterators are supported.**
   Objects conforming to the iterator protocol but not inheriting from
   `Iterator` or `AsyncIterator` are detected and wrapped via
   `AsyncIterator.from()` (presupposes TC39 defines this). A duck-typed
   iterator whose `next()` returns a `Promise` also works, since the
   async source handler awaits all results.

8. **Bare iterables are supported.**
   Objects with `Symbol.asyncIterator` or `Symbol.iterator` (but which
   are not themselves iterators) are supported. The transfer steps invoke
   the factory to obtain the iterator, then transfer that.
   `Symbol.asyncIterator` takes precedence over `Symbol.iterator`,
   matching `for await...of`.

## Type Promotion

| Source side     | Receiving side   |
|-----------------|------------------|
| `Iterator`      | `AsyncIterator`  |
| `AsyncIterator` | `AsyncIterator`  |

## Message Protocol

### Receiving -> Sending (requests)

| Message                                               | Purpose                    |
|-------------------------------------------------------|----------------------------|
| `{ type: "next",   value: <cloneable or undefined> }` | Pull next value            |
| `{ type: "return", value: <cloneable or undefined> }` | Early termination          |
| `{ type: "throw",  value: <cloneable error> }`        | Error injection            |

### Sending -> Receiving (responses)

| Message                                                       | Purpose                   |
|---------------------------------------------------------------|---------------------------|
| `{ type: "result", value: { value: <cloneable>, done: <boolean> } }` | Normal iterator result    |
| `{ type: "error",  value: <cloneable error> }`                | Unhandled error from source |

## Disentanglement Rules

The port pair is disentangled when the conversation is over:

| Event                                       | Who disentangles  | Source-side `return()` |
|---------------------------------------------|-------------------|------------------------|
| Result with `done: true` (from `next`)      | Both sides        | Yes (fire-and-forget)  |
| Result with `done: true` (from `throw`)     | Both sides        | Yes (fire-and-forget)  |
| `return` (regardless of result)             | Source side        | N/A (explicit)         |
| `throw` that produces an abrupt completion  | Source side        | No                     |
| `"error"` message received                  | Consumer side      | No                     |
| `messageerror` event                        | Both sides         | Yes (existing)         |

When `done: true` is produced by `next` or `throw`, the source side
calls `iterator.return()` (if callable) before disentangling. The result
is discarded and errors are swallowed — this is purely resource cleanup.

When `throw` produces a normal completion (the generator caught the error
and yielded another value), the port stays alive.

## Interface Definitions

```webidl
[Exposed=*, Transferable]
interface Iterator { /* ... existing definition ... */ };

[Exposed=*, Transferable]
interface AsyncIterator { /* ... existing definition ... */ };
```

### Internal Slots (added for transfer)

| Internal Slot  | Description                                                  |
|----------------|--------------------------------------------------------------|
| `[[Detached]]` | A boolean flag set to **true** when the iterator is transferred |

After transfer, the original object in the sending realm is detached.
Subsequent calls to `next()` / `return()` / `throw()` throw `TypeError`.
The iterator is now owned by the message handler on the sending-side port.

## Transfer Steps

### `Iterator` transfer steps

Given *value* and *dataHolder*:

1. Let *iterator* be ? ResolveIteratorForTransfer(*value*).
2. Let *port1* be a new `MessagePort` in the current Realm.
3. Let *port2* be a new `MessagePort` in the current Realm.
4. Entangle *port1* and *port2*.
5. If *iterator* is an async iterator, perform
   ! SetUpCrossRealmTransformAsyncIteratorSource(*iterator*, *port1*).
6. Otherwise, perform
   ! SetUpCrossRealmTransformIteratorSource(*iterator*, *port1*).
7. Set *dataHolder*.[[port]] to
   ! StructuredSerializeWithTransfer(*port2*, « *port2* »).

### `Iterator` transfer-receiving steps

Given *dataHolder* and *value*:

1. Let *deserializedRecord* be
   ! StructuredDeserializeWithTransfer(*dataHolder*.[[port]], the current
   Realm).
2. Let *port* be *deserializedRecord*.[[Deserialized]].
3. Perform ! SetUpCrossRealmTransformAsyncIteratorConsumer(*value*, *port*).

> NOTE: The receiving side always produces an `AsyncIterator`, regardless of
> whether the source was sync or async.

### `AsyncIterator` transfer steps

Given *value* and *dataHolder*:

1. Let *iterator* be ? ResolveIteratorForTransfer(*value*).
2. Let *port1* be a new `MessagePort` in the current Realm.
3. Let *port2* be a new `MessagePort` in the current Realm.
4. Entangle *port1* and *port2*.
5. If *iterator* is an async iterator, perform
   ! SetUpCrossRealmTransformAsyncIteratorSource(*iterator*, *port1*).
6. Otherwise, perform
   ! SetUpCrossRealmTransformIteratorSource(*iterator*, *port1*).
7. Set *dataHolder*.[[port]] to
   ! StructuredSerializeWithTransfer(*port2*, « *port2* »).

### `AsyncIterator` transfer-receiving steps

Given *dataHolder* and *value*:

1. Let *deserializedRecord* be
   ! StructuredDeserializeWithTransfer(*dataHolder*.[[port]], the current
   Realm).
2. Let *port* be *deserializedRecord*.[[Deserialized]].
3. Perform ! SetUpCrossRealmTransformAsyncIteratorConsumer(*value*, *port*).

## Abstract Operations

### ResolveIteratorForTransfer(*value*)

Resolves the value into a concrete iterator for transfer.

1. If *value* is an `AsyncIterator` instance, return *value*.
2. If *value* is an `Iterator` instance, return *value*.
3. Let *asyncIteratorMethod* be ? GetMethod(*value*, `@@asyncIterator`).
4. If *asyncIteratorMethod* is not **undefined**:
   1. Let *iterator* be ? Call(*asyncIteratorMethod*, *value*).
   2. If *iterator* does not have a callable `next` method, throw a
      `TypeError`.
   3. Return *iterator*.

      > NOTE: *iterator* is treated as an async iterator.

5. Let *iteratorMethod* be ? GetMethod(*value*, `@@iterator`).
6. If *iteratorMethod* is not **undefined**:
   1. Let *iterator* be ? Call(*iteratorMethod*, *value*).
   2. If *iterator* does not have a callable `next` method, throw a
      `TypeError`.
   3. Return *iterator*.

      > NOTE: *iterator* is treated as a sync iterator.

7. If *value* has a callable `next` method:
   1. Return *value*.

      > NOTE: *value* is treated as an async iterator. `await` on a
      > non-Promise return from `next()` is a no-op, so this correctly
      > handles both sync and Promise-returning duck-typed iterators.

8. Throw a `TypeError`.

> NOTE: `@@asyncIterator` takes precedence over `@@iterator`, consistent
> with `for await...of` semantics.

### PackAndPostMessage(*port*, *type*, *value*)

Per the [Streams spec definition][pack-and-post].

[pack-and-post]: https://streams.spec.whatwg.org/#pack-and-post-message

1. Let *message* be OrdinaryObjectCreate(**null**).
2. Perform ! CreateDataProperty(*message*, `"type"`, *type*).
3. Perform ! CreateDataProperty(*message*, `"value"`, *value*).
4. Let *targetPort* be the port with which *port* is entangled, if any;
   otherwise let it be **null**.
5. Let *options* be «[ `"transfer"` -> « » ]».
6. Run the message port post message steps providing *targetPort*, *message*,
   and *options*.

> NOTE: A JavaScript object with a **null** prototype is used to avoid
> interference from `%Object.prototype%`.

### PackAndPostMessageHandlingError(*port*, *type*, *value*)

1. Let *result* be PackAndPostMessage(*port*, *type*, *value*).
2. If *result* is an abrupt completion:
   1. Perform ! CrossRealmTransformSendError(*port*, *result*.[[Value]]).
3. Return *result* as a completion record.

### CrossRealmTransformSendError(*port*, *error*)

1. Perform PackAndPostMessage(*port*, `"error"`, *error*), discarding the
   result.

> NOTE: As we are already in an errored state when this abstract operation
> is performed, we cannot handle further errors, so we just discard them.

### SetUpCrossRealmTransformIteratorSource(*iterator*, *port*)

Sets up the sending side for a **sync** iterator.

1. Add a handler for *port*'s `message` event with the following steps:
   1. Let *data* be the data of the message.
   2. Assert: *data* is an Object.
   3. Let *type* be ! Get(*data*, `"type"`).
   4. Let *value* be ! Get(*data*, `"value"`).
   5. Assert: *type* is a String.
   6. If *type* is `"next"`:
      1. Let *result* be Completion(IteratorNext(*iterator*, *value*)).
      2. If *result* is an abrupt completion:
         1. Perform ! PackAndPostMessage(*port*, `"error"`,
            *result*.[[Value]]).
         2. Disentangle *port*.
      3. Otherwise:
         1. Let *iterResult* be *result*.[[Value]].
         2. Let *done* be ? IteratorComplete(*iterResult*).
         3. Let *resultValue* be ? IteratorValue(*iterResult*).
         4. Let *responseObject* be OrdinaryObjectCreate(**null**).
         5. Perform ! CreateDataProperty(*responseObject*, `"value"`,
            *resultValue*).
         6. Perform ! CreateDataProperty(*responseObject*, `"done"`, *done*).
          7. Perform PackAndPostMessageHandlingError(*port*, `"result"`,
             *responseObject*).
          8. If *done* is **true**:
             1. If *iterator* has a callable `return` method, perform
                Completion(Call(*iterator*.`return`, *iterator*)),
                discarding the result.
             2. Disentangle *port*.
   7. Otherwise, if *type* is `"return"`:
      1. If *iterator* does not have a `return` method:
         1. Let *responseObject* be OrdinaryObjectCreate(**null**).
         2. Perform ! CreateDataProperty(*responseObject*, `"value"`,
            *value*).
         3. Perform ! CreateDataProperty(*responseObject*, `"done"`,
            **true**).
         4. Perform ! PackAndPostMessage(*port*, `"result"`,
            *responseObject*).
         5. Disentangle *port*.
      2. Otherwise:
         1. Let *result* be Completion(Call(*iterator*.`return`, *iterator*,
            « *value* »)).
         2. If *result* is an abrupt completion:
            1. Perform ! PackAndPostMessage(*port*, `"error"`,
               *result*.[[Value]]).
         3. Otherwise:
            1. Perform ! PackAndPostMessage(*port*, `"result"`,
               *result*.[[Value]]).
         4. Disentangle *port*.
   8. Otherwise, if *type* is `"throw"`:
      1. If *iterator* does not have a `throw` method:
         1. Perform ! PackAndPostMessage(*port*, `"error"`, *value*).
         2. Disentangle *port*.
      2. Otherwise:
         1. Let *result* be Completion(Call(*iterator*.`throw`, *iterator*,
            « *value* »)).
         2. If *result* is an abrupt completion:
            1. Perform ! PackAndPostMessage(*port*, `"error"`,
               *result*.[[Value]]).
            2. Disentangle *port*.
         3. Otherwise:
            1. Let *iterResult* be *result*.[[Value]].
            2. Let *done* be ? IteratorComplete(*iterResult*).
            3. Let *resultValue* be ? IteratorValue(*iterResult*).
            4. Let *responseObject* be OrdinaryObjectCreate(**null**).
            5. Perform ! CreateDataProperty(*responseObject*, `"value"`,
               *resultValue*).
            6. Perform ! CreateDataProperty(*responseObject*, `"done"`,
               *done*).
             7. Perform PackAndPostMessageHandlingError(*port*, `"result"`,
                *responseObject*).
             8. If *done* is **true**:
                1. If *iterator* has a callable `return` method, perform
                   Completion(Call(*iterator*.`return`, *iterator*)),
                   discarding the result.
                2. Disentangle *port*.
2. Add a handler for *port*'s `messageerror` event with the following steps:
   1. Let *error* be a new `"DataCloneError"` DOMException.
   2. If *iterator* has a callable `return` method, perform
      Completion(Call(*iterator*.`return`, *iterator*)), discarding the
      result.
   3. Disentangle *port*.
3. Enable *port*'s port message queue.

> NOTE: Implementations are encouraged to explicitly handle failures from
> the asserts in this algorithm, as the input might come from an untrusted
> context. Failure to do so could lead to security issues.

### SetUpCrossRealmTransformAsyncIteratorSource(*iterator*, *port*)

Sets up the sending side for an **async** (or duck-typed) iterator.

1. Let *requestQueue* be a new empty List.
2. Let *processing* be **false**.
3. Let *done* be **false**.
4. Let *ProcessQueue* be the following steps:
   1. If *processing* is **true**, return.
   2. Set *processing* to **true**.
   3. While *requestQueue* is not empty and *done* is **false**:
      1. Let *request* be the first element of *requestQueue*; remove it.
      2. Let *type* be *request*.[[Type]].
      3. Let *value* be *request*.[[Value]].
      4. Perform ProcessRequest(*type*, *value*) (below).
   4. Set *processing* to **false**.
5. Let ProcessRequest(*type*, *value*) be the following steps:
   1. If *type* is `"next"`:
      1. Let *resultPromise* be
         Completion(Call(*iterator*.`next`, *iterator*, « *value* »)).
      2. If *resultPromise* is an abrupt completion:
         1. Perform ! PackAndPostMessage(*port*, `"error"`,
            *resultPromise*.[[Value]]).
         2. Set *done* to **true**. Disentangle *port*.
         3. Return.
      3. Await *resultPromise*.[[Value]]. On fulfillment with *iterResult*:
         1. Let *iterDone* be IteratorComplete(*iterResult*).
         2. Let *resultValue* be IteratorValue(*iterResult*).
         3. Let *responseObject* be OrdinaryObjectCreate(**null**).
         4. Perform ! CreateDataProperty(*responseObject*, `"value"`,
            *resultValue*).
         5. Perform ! CreateDataProperty(*responseObject*, `"done"`,
            *iterDone*).
         6. Perform PackAndPostMessageHandlingError(*port*, `"result"`,
            *responseObject*).
         7. If *iterDone* is **true**:
            1. If *iterator* has a callable `return` method:
               1. Let *returnPromise* be
                  Completion(Call(*iterator*.`return`, *iterator*)).
               2. If *returnPromise* is a normal completion, await
                  *returnPromise*.[[Value]], discarding the result.
            2. Set *done* to **true**. Disentangle *port*.
      4. On rejection with *reason*:
         1. Perform ! PackAndPostMessage(*port*, `"error"`, *reason*).
         2. Set *done* to **true**. Disentangle *port*.
   2. Otherwise, if *type* is `"return"`:
      1. If *iterator* does not have a `return` method:
         1. Let *responseObject* be OrdinaryObjectCreate(**null**).
         2. Perform ! CreateDataProperty(*responseObject*, `"value"`,
            *value*).
         3. Perform ! CreateDataProperty(*responseObject*, `"done"`,
            **true**).
         4. Perform ! PackAndPostMessage(*port*, `"result"`,
            *responseObject*).
         5. Set *done* to **true**. Disentangle *port*.
         6. Return.
      2. Let *resultPromise* be
         Completion(Call(*iterator*.`return`, *iterator*, « *value* »)).
      3. If *resultPromise* is an abrupt completion:
         1. Perform ! PackAndPostMessage(*port*, `"error"`,
            *resultPromise*.[[Value]]).
         2. Set *done* to **true**. Disentangle *port*.
         3. Return.
      4. Await *resultPromise*.[[Value]]. On fulfillment with *iterResult*:
         1. Perform ! PackAndPostMessage(*port*, `"result"`, *iterResult*).
         2. Set *done* to **true**. Disentangle *port*.
      5. On rejection with *reason*:
         1. Perform ! PackAndPostMessage(*port*, `"error"`, *reason*).
         2. Set *done* to **true**. Disentangle *port*.
   3. Otherwise, if *type* is `"throw"`:
      1. If *iterator* does not have a `throw` method:
         1. Perform ! PackAndPostMessage(*port*, `"error"`, *value*).
         2. Set *done* to **true**. Disentangle *port*.
         3. Return.
      2. Let *resultPromise* be
         Completion(Call(*iterator*.`throw`, *iterator*, « *value* »)).
      3. If *resultPromise* is an abrupt completion:
         1. Perform ! PackAndPostMessage(*port*, `"error"`,
            *resultPromise*.[[Value]]).
         2. Set *done* to **true**. Disentangle *port*.
         3. Return.
      4. Await *resultPromise*.[[Value]]. On fulfillment with *iterResult*:
         1. Let *iterDone* be IteratorComplete(*iterResult*).
         2. Let *resultValue* be IteratorValue(*iterResult*).
         3. Let *responseObject* be OrdinaryObjectCreate(**null**).
         4. Perform ! CreateDataProperty(*responseObject*, `"value"`,
            *resultValue*).
         5. Perform ! CreateDataProperty(*responseObject*, `"done"`,
            *iterDone*).
         6. Perform PackAndPostMessageHandlingError(*port*, `"result"`,
            *responseObject*).
         7. If *iterDone* is **true**:
            1. If *iterator* has a callable `return` method:
               1. Let *returnPromise* be
                  Completion(Call(*iterator*.`return`, *iterator*)).
               2. If *returnPromise* is a normal completion, await
                  *returnPromise*.[[Value]], discarding the result.
            2. Set *done* to **true**. Disentangle *port*.
      5. On rejection with *reason*:
         1. Perform ! PackAndPostMessage(*port*, `"error"`, *reason*).
         2. Set *done* to **true**. Disentangle *port*.
6. Add a handler for *port*'s `message` event with the following steps:
   1. Let *data* be the data of the message.
   2. Assert: *data* is an Object.
   3. Let *type* be ! Get(*data*, `"type"`).
   4. Let *value* be ! Get(*data*, `"value"`).
   5. Assert: *type* is a String.
   6. Append Record { [[Type]]: *type*, [[Value]]: *value* } to
      *requestQueue*.
   7. Perform *ProcessQueue*.
7. Add a handler for *port*'s `messageerror` event with the following steps:
   1. Let *error* be a new `"DataCloneError"` DOMException.
   2. If *iterator* has a callable `return` method, perform
      Completion(Call(*iterator*.`return`, *iterator*)), discarding the
      result.
   3. Set *done* to **true**. Disentangle *port*.
8. Enable *port*'s port message queue.

> NOTE: The *requestQueue* and *ProcessQueue* mechanism serializes async
> request processing. Multiple requests may arrive while an `await` is
> pending; they queue up and are processed in FIFO order. The *done*
> flag short-circuits remaining queued requests after the iterator
> exhausts or errors — the consumer handles these via its own drain
> logic.
>
> Implementations are encouraged to explicitly handle failures from the
> asserts in this algorithm, as the input might come from an untrusted
> context. Failure to do so could lead to security issues.

### SetUpCrossRealmTransformAsyncIteratorConsumer(*value*, *port*)

Sets up the receiving side. Always produces an `AsyncIterator`.

1. Let *pendingQueue* be a new empty List.
   Each element is a Record { [[Resolve]], [[Reject]] }.
2. Let *finished* be **false**.
3. Let *DrainQueue* be the following steps:
   1. For each *entry* in *pendingQueue*:
      1. Let *doneResult* be OrdinaryObjectCreate(**null**).
      2. Perform ! CreateDataProperty(*doneResult*, `"value"`, **undefined**).
      3. Perform ! CreateDataProperty(*doneResult*, `"done"`, **true**).
      4. Call *entry*.[[Resolve]](*doneResult*).
   2. Empty *pendingQueue*.
   3. Disentangle *port*.
4. Add a handler for *port*'s `message` event with the following steps:
   1. Let *data* be the data of the message.
   2. Assert: *data* is an Object.
   3. Let *type* be ! Get(*data*, `"type"`).
   4. Let *responseValue* be ! Get(*data*, `"value"`).
   5. Assert: *type* is a String.
   6. Assert: *pendingQueue* is not empty.
   7. Let *entry* be the first element of *pendingQueue*; remove it.
   8. If *type* is `"result"`:
      1. Let *done* be ! Get(*responseValue*, `"done"`).
      2. Call *entry*.[[Resolve]](*responseValue*).
      3. If *done* is **true**:
         1. Set *finished* to **true**.
         2. Perform *DrainQueue*.
   9. Otherwise, if *type* is `"error"`:
      1. Call *entry*.[[Reject]](*responseValue*).
      2. Set *finished* to **true**.
      3. Perform *DrainQueue*.
5. Add a handler for *port*'s `messageerror` event with the following steps:
   1. Let *error* be a new `"DataCloneError"` DOMException.
   2. Set *finished* to **true**.
   3. If *pendingQueue* is not empty:
      1. Let *entry* be the first element of *pendingQueue*; remove it.
      2. Call *entry*.[[Reject]](*error*).
   4. Perform *DrainQueue*.
6. Enable *port*'s port message queue.
7. Let *nextMethod* be a new built-in function that, given *arg*:
   1. If *finished* is **true**, return a promise resolved with
      OrdinaryObjectCreate(**null**) with properties `"value"` ->
      **undefined** and `"done"` -> **true**.
   2. Let *promise* be a new promise.
   3. Append Record { [[Resolve]]: *promise*'s resolve, [[Reject]]:
      *promise*'s reject } to *pendingQueue*.
   4. Perform ! PackAndPostMessage(*port*, `"next"`, *arg*).
   5. Return *promise*.
8. Let *returnMethod* be a new built-in function that, given *arg*:
   1. If *finished* is **true**, return a promise resolved with
      OrdinaryObjectCreate(**null**) with properties `"value"` -> *arg*
      and `"done"` -> **true**.
   2. Set *finished* to **true**.
   3. Let *promise* be a new promise.
   4. Append Record { [[Resolve]]: *promise*'s resolve, [[Reject]]:
      *promise*'s reject } to *pendingQueue*.
   5. Perform ! PackAndPostMessage(*port*, `"return"`, *arg*).
   6. Return *promise*.
9. Let *throwMethod* be a new built-in function that, given *arg*:
   1. If *finished* is **true**, return a promise rejected with *arg*.
   2. Let *promise* be a new promise.
   3. Append Record { [[Resolve]]: *promise*'s resolve, [[Reject]]:
      *promise*'s reject } to *pendingQueue*.
   4. Perform ! PackAndPostMessage(*port*, `"throw"`, *arg*).
   5. Return *promise*.
10. Set *value*'s `next` method to *nextMethod*.
11. Set *value*'s `return` method to *returnMethod*.
12. Set *value*'s `throw` method to *throwMethod*.

> NOTE: Multiple `next()` calls can be in flight simultaneously. Each
> sends a request message and appends to *pendingQueue*. Responses
> resolve queued promises in FIFO order. When `done: true` or an error
> arrives, the responding promise is resolved/rejected, then all
> remaining queued promises are drained with `{ value: undefined,
> done: true }` — the source has closed the port, so those requests
> will never receive responses.
>
> After *finished* is **true**, `next()` returns
> `{ value: undefined, done: true }`, `return()` returns
> `{ value: arg, done: true }`, and `throw()` rejects -- all without
> touching the port. This matches the expected behavior of an exhausted
> async iterator and avoids posting messages to a disentangled port.

## Usage Examples

### Transferring an `AsyncIterator` to a Worker

```js
async function* generateData() {
  for (let i = 0; i < 1000; i++) {
    yield { id: i, data: await fetchChunk(i) };
  }
}

const iter = generateData();
worker.postMessage(iter, { transfer: [iter] });
// iter is now detached in this realm
```

In the worker:

```js
self.onmessage = async (e) => {
  const iter = e.data; // AsyncIterator
  for await (const item of iter) {
    process(item);
  }
};
```

### Transferring a sync `Iterator`

```js
const iter = [1, 2, 3, 4, 5].values();
worker.postMessage(iter, { transfer: [iter] });
// iter is detached; worker receives an AsyncIterator
```

### Transferring a bare iterable

```js
const myIterable = {
  [Symbol.asyncIterator]() {
    let i = 0;
    return {
      next() { return Promise.resolve({ value: i++, done: i > 10 }); }
    };
  }
};

worker.postMessage(myIterable, { transfer: [myIterable] });
// The transfer steps invoke [Symbol.asyncIterator]() to obtain the
// iterator, then set up the cross-realm bridge on that iterator.
```

### Transferring a duck-typed iterator

```js
const iter = {
  i: 0,
  next() { return { value: this.i++, done: this.i > 10 }; }
};

worker.postMessage(iter, { transfer: [iter] });
// Detected as duck-typed iterator (has callable next),
// wrapped as async, transferred.
```

### Generator protocol over the boundary

```js
// Sending realm
function* stateMachine() {
  let input = yield 'ready';
  while (input !== 'stop') {
    input = yield `processed: ${input}`;
  }
  return 'done';
}

const iter = stateMachine();
worker.postMessage(iter, { transfer: [iter] });
```

```js
// Receiving realm (worker)
self.onmessage = async (e) => {
  const iter = e.data; // AsyncIterator

  let result = await iter.next();       // { value: 'ready', done: false }
  result = await iter.next('hello');    // { value: 'processed: hello', done: false }
  result = await iter.next('world');    // { value: 'processed: world', done: false }
  result = await iter.next('stop');     // { value: 'done', done: true }
  // Port is now disentangled.
};
```

## Comparison with `ReadableStream` Transfer

| Aspect                  | `ReadableStream`                          | Iterator / `AsyncIterator`                      |
|-------------------------|-------------------------------------------|-------------------------------------------------|
| **Protocol type**       | Push with backpressure                    | Strict request-response                         |
| **Buffering**           | Internal queue with high water mark       | None                                            |
| **Backpressure**        | `"pull"` messages + backpressure promise  | Implicit (no next request until response)       |
| **Value passing**       | Chunks flow one direction                 | Bidirectional (`next(value)` sends values back) |
| **Receiving type**      | `ReadableStream`                          | Always `AsyncIterator`                          |
| **Completion signal**   | `"close"` message                         | Result with `done: true`                        |
| **Source handler**      | Pipe into cross-realm writable            | Direct message handler on port                  |

## Dependencies

- **`AsyncIterator.from()`**: This design presupposes that TC39 defines
  `AsyncIterator.from()` for wrapping duck-typed iterators. Without it,
  equivalent wrapping logic would need to be specified inline in the
  transfer steps.

## Open Questions

1. **Should `for await...of` work transparently on the transferred
   iterator?** It should, since the receiving side is a proper
   `AsyncIterator` with a conformant `next()` method. The `throw()` and
   `return()` methods ensure proper cleanup when breaking out of loops.

2. **Should the structured clone algorithm attempt to clone (not transfer)
   the *values* returned by `next()`, or should transfer lists be
   supported per-value?** The current design clones values via
   `PackAndPostMessage`, which uses the message port post message steps.
   Supporting per-value transfer lists (e.g., for `ArrayBuffer` chunks)
   would require extending the protocol.

3. **What happens if the sending realm is destroyed (e.g., an iframe is
   removed) while the iterator is in use?** The port would be
   disentangled, and the next `next()` call on the consumer side would
   never resolve. A `messageerror` or port closure event could be used to
   reject the pending promise.

## Relationship to the Transfer Protocol Proposal

The [Transfer Protocol proposal](./README.md) introduces `Symbol.transfer`,
`Object.transfer()`, `Reflect.transfer()`, a `[[Transfer]]` internal method,
and a Proxy `transfer` handler trap -- all operating within a single realm.
Cross-realm iterator transfer operates across realms via `postMessage()`.
The two are complementary.

### Two Levels of Transfer

The Transfer Protocol proposal identifies two levels:

```
Same realm                            Cross realm
─────────────                         ─────────────
Object.transfer(obj)                  worker.postMessage(msg, { transfer: [obj] })
  │                                     │
  ├─ Invokes [[Transfer]]               ├─ Structured clone algorithm
  ├─ Calls [Symbol.transfer]()          ├─ Checks [Transferable] internal
  ├─ Detaches source                    ├─ Detaches source
  └─ Returns new owner (same realm)     └─ Reconstructs in target realm
```

This document fills in the right column for iterators. The Transfer
Protocol proposal fills in the left. Both detach the source. The difference
is where the new owner lives.

### Shared `[[Detached]]` Semantics

Both mechanisms use the same `[[Detached]]` internal slot. When
`[[Detached]]` is **true**, `.next()`, `.return()`, `.throw()`,
`[Symbol.iterator]()`, and `[Symbol.transfer]()` all throw `TypeError`.

Post-transfer behavior is identical regardless of which mechanism detached
the iterator:

| Method on detached iterator | Same-realm transfer | Cross-realm transfer |
|-----------------------------|---------------------|----------------------|
| `.next()`                   | Throws `TypeError`  | Throws `TypeError`   |
| `.return()`                 | Throws `TypeError`  | Throws `TypeError`   |
| `.throw()`                  | Throws `TypeError`  | Throws `TypeError`   |
| `[Symbol.iterator]()`      | Throws `TypeError`  | Throws `TypeError`   |
| `[Symbol.transfer]()`      | Throws `TypeError`  | Throws `TypeError`   |
| `[Symbol.dispose]()`       | No-op               | No-op                |

### Mutual Exclusivity

`[Symbol.transfer]()` throws `TypeError` if already detached. Either
form of transfer prevents the other on the same instance:

```js
const iter = generateData();

// Case 1: Same-realm transfer first
const owned = Object.transfer(iter);
worker.postMessage(iter, { transfer: [iter] });
// postMessage transfer steps find [[Detached]] is true → throws

// Case 2: Cross-realm transfer first
worker.postMessage(iter, { transfer: [iter] });
Object.transfer(iter);
// [[Transfer]] calls [Symbol.transfer](), which checks [[Detached]] → throws
```

Same-realm transfer can precede cross-realm transfer in an ownership
pipeline:

```js
const iter = generateData();
const owned = Object.transfer(iter);
// iter is detached; owned is a new Iterator in this realm
worker.postMessage(owned, { transfer: [owned] });
// owned is now detached; the worker has the iterator
```

Each step detaches the previous holder.

### `postMessage()` Could Invoke `[Symbol.transfer]()`

The Transfer Protocol proposal notes that the web platform could recognize
`Symbol.transfer` in `postMessage()` transfer lists as a WHATWG/W3C
follow-on. If so, the `postMessage()` transfer steps could be:

1. Invoke `[Symbol.transfer]()` on the source iterator to obtain a
   new iterator and detach the source (same-realm detach semantics).
2. Set up the cross-realm `MessagePort` bridge using the *new* iterator
   as the source.

This unifies detachment -- `postMessage()` transfer would use
`[Symbol.transfer]()` for the detach step rather than independently
setting `[[Detached]]` to **true**.

The tradeoff: `[Symbol.transfer]()` returns a new object with the
iterator's state moved into it. That new object exists solely to serve as
the source for the `MessagePort` bridge -- never exposed to user code.
Same-realm transfer machinery runs as a side effect of cross-realm
transfer.

The alternative (and the approach this document takes) is for the
`postMessage()` transfer steps to set `[[Detached]]` directly and retain
the original iterator's internal slots for the `MessagePort` bridge,
without invoking `[Symbol.transfer]()`. This avoids creating a throwaway
intermediate object and keeps the two mechanisms independent.

### Duck-Typed Iterators: Divergent Handling

The two mechanisms handle duck-typed iterators differently:

| Iterator type                 | `Object.transfer()`                          | `postMessage()` transfer                     |
|-------------------------------|----------------------------------------------|----------------------------------------------|
| Proper `Iterator` instance    | Uses `Iterator.prototype[@@transfer]`        | Sets up sync source handler on port          |
| Proper `AsyncIterator` instance | Uses `AsyncIterator.prototype[@@transfer]` | Sets up async source handler on port         |
| Duck-typed `{ next() {} }`   | Throws `TypeError` (no `[Symbol.transfer]`)  | Detected and wrapped as async via `ResolveIteratorForTransfer` |
| Bare iterable                 | Throws `TypeError` (no `[Symbol.transfer]`)  | Factory invoked, iterator obtained and transferred |

`Object.transfer()` is strict: the object must implement
`[Symbol.transfer]()`. Duck-typed iterators have no shared prototype on
which to define the method, so they are excluded.

`postMessage()` is more permissive: `ResolveIteratorForTransfer` detects
duck-typed iterators by checking for a callable `next` method and wraps
them. Duck-typed iterators are common in the ecosystem, and requiring
wrapping before `postMessage()` would be an unnecessary obstacle.

To use both mechanisms with a duck-typed iterator, wrap explicitly:

```js
const plain = { i: 0, next() { return { value: this.i++, done: this.i > 3 }; } };

// Same-realm: wrap first, then transfer
const wrapped = Iterator.from(plain);
const owned = Object.transfer(wrapped);

// Cross-realm: works directly (ResolveIteratorForTransfer handles it)
worker.postMessage(plain, { transfer: [plain] });
```

### The `Iterator.from()` Leaky Abstraction

The Transfer Protocol proposal's Open Question 2 asks whether
`Iterator.from()` should bridge non-transferable iterators and leans toward
*no* -- transferring the wrapper detaches the wrapper but leaves the
underlying plain object accessible.

Cross-realm transfer sidesteps this. When `ResolveIteratorForTransfer`
wraps a duck-typed iterator, the wrapper exists only on the sending side as
the source for the `MessagePort` bridge. The plain object is not detached
(no `[[Detached]]` slot), but cannot reach the receiving realm. The
original remains accessible in the sending realm, but the cross-realm
consumer is isolated from interference.

This is a weaker guarantee than `Object.transfer()` (where the source is
rendered unusable), but matches `ReadableStream` transfer -- the original
stream is locked and piped, but the underlying source callbacks still run
in the sending realm.

### Proxy `transfer` Trap

The Proxy `transfer` trap intercepts `[[Transfer]]()` on Proxy exotic
objects. It is relevant to same-realm transfer only -- `postMessage()`
does not invoke `[[Transfer]]()`, so the trap does not fire during
cross-realm transfer.

A Proxy membrane intercepting same-realm transfer will *not* intercept
cross-realm transfer of that iterator. If cross-realm transfer steps
invoked `[Symbol.transfer]()` as discussed above, the trap would fire,
restoring the membrane's ability to intercept -- another argument for
unifying the detach mechanism.

### `using` / `await using`

After either form of transfer, if the original was bound with
`using` / `await using`, the dispose call at block exit must be a no-op:

```js
{
  await using iter = source[Symbol.asyncIterator]();
  worker.postMessage(iter, { transfer: [iter] });
  // iter is detached via postMessage transfer
}
// iter[Symbol.asyncDispose]() is called here -- must be a silent no-op
```

Same behavior as the same-realm case in the Transfer Protocol proposal.

### Summary

| Aspect                           | Same-realm (`Object.transfer`)               | Cross-realm (`postMessage`)                       |
|----------------------------------|----------------------------------------------|---------------------------------------------------|
| **Specification**                | TC39 (ECMAScript)                            | WHATWG (HTML / Streams-like)                      |
| **Mechanism**                    | `[[Transfer]]` -> `[Symbol.transfer]()`      | Structured clone transfer steps + `MessagePort`   |
| **Result location**              | Same realm                                   | Different realm                                   |
| **Result type**                  | Same as source (`Iterator` or `AsyncIterator`) | Always `AsyncIterator`                          |
| **Detach mechanism**             | `[Symbol.transfer]()` sets `[[Detached]]`    | Transfer steps set `[[Detached]]` directly        |
| **Duck-typed iterators**         | Not supported (throws `TypeError`)           | Detected and wrapped via `ResolveIteratorForTransfer` |
| **Bare iterables**               | Not supported (throws `TypeError`)           | Factory invoked, iterator transferred             |
| **Proxy trap fires**             | Yes                                          | No (unless steps invoke `[Symbol.transfer]()`)    |
| **Bidirectional values**         | N/A (same object, direct calls)              | Yes (`next(value)` via `MessagePort`)             |
| **`[[Detached]]` post-state**    | Identical                                    | Identical                                         |
| **`[Symbol.dispose]()` post-state** | No-op                                     | No-op                                             |

## Spec Changes Required for Standardization

Coordinated changes across multiple specifications.

### ECMAScript (TC39)

Changes to `Iterator` and `AsyncIterator` definitions, independent of
whether cross-realm transfer is pursued:

**1. `[[Detached]]` internal slot on all built-in iterator types.**

Every built-in iterator prototype (`%ArrayIteratorPrototype%`,
`%MapIteratorPrototype%`, `%SetIteratorPrototype%`,
`%StringIteratorPrototype%`, `%RegExpStringIteratorPrototype%`,
`%GeneratorPrototype%`, `%AsyncGeneratorPrototype%`, and Iterator Helper
result prototypes) needs a `[[Detached]]` boolean internal slot,
initialized to **false**. Both same-realm and cross-realm transfer need
this slot.

**2. Detach guards on `.next()`, `.return()`, `.throw()`.**

Every built-in iterator's `.next()`, `.return()`, and `.throw()` must
check `[[Detached]]` before proceeding; throw `TypeError` if **true**.
`[Symbol.iterator]()` and `[Symbol.asyncIterator]()` must also throw on
detached instances to prevent `for...of` / `for await...of` from
silently entering iteration on a dead iterator.

**3. `AsyncIterator.from()` (dependency).**

`ResolveIteratorForTransfer` relies on `AsyncIterator.from()` for
wrapping duck-typed iterators. This is part of the
[Iterator Helpers proposal][iterator-helpers] (Stage 3). Without it,
the wrapping logic would need to be specified inline in the transfer
steps.

[iterator-helpers]: https://github.com/tc39/proposal-iterator-helpers

**4. `[Symbol.dispose]()` / `[Symbol.asyncDispose]()` no-op on detached.**

If iterators gain dispose semantics (via Explicit Resource Management),
dispose must silently return on detached instances rather than throwing.

### HTML Standard (WHATWG)

The HTML spec owns the structured clone algorithm, `[Transferable]`,
`MessagePort`, and `postMessage()` transfer list processing.

**1. Add `Iterator` and `AsyncIterator` to `[Transferable]`.**

`Iterator` and `AsyncIterator` would join `ReadableStream`,
`WritableStream`, `MessagePort`, `ArrayBuffer`, etc. Precedent:
`ArrayBuffer` is defined in ECMAScript and declared transferable in
the HTML spec.

Complication: `Iterator` and `AsyncIterator` in ECMAScript are
prototype objects, not constructors with fixed internal slots. The
`[Transferable]` mechanism is defined in terms of platform interface
objects. The spec would need to define how transfer steps apply to
objects inheriting from `Iterator.prototype` or
`AsyncIterator.prototype`.

Cleaner approach: define transfer steps on `Iterator.prototype` and
`AsyncIterator.prototype` that apply to any inheriting object, the
same way `ReadableStream`'s transfer steps are defined once on the
interface.

**2. Define transfer steps and transfer-receiving steps.**

As described in this document:

- **Iterator transfer steps**: `ResolveIteratorForTransfer`, create
  `MessageChannel`, set up source handler on port1, serialize port2.
- **Iterator transfer-receiving steps**: deserialize port2, run
  `SetUpCrossRealmTransformAsyncIteratorConsumer`.
- **AsyncIterator transfer steps**: same structure, async source handler.
- **AsyncIterator transfer-receiving steps**: identical to Iterator's.

Same pattern as `ReadableStream` transfer steps (Streams spec §4.2.6).

**3. `ResolveIteratorForTransfer` abstract operation.**

Resolves the value into a concrete iterator, handling proper instances,
bare iterables (invoking `@@asyncIterator` or `@@iterator`), and
duck-typed iterators (detecting a callable `next`).

This operation has **side effects** in the iterable cases (steps 3-6):
it calls user-defined `@@asyncIterator` or `@@iterator`. The structured
clone algorithm does not typically invoke user-defined methods during
transfer steps (it reads internal slots). Justification: the developer
explicitly placed the value in the transfer list.

**4. Extend `postMessage()` transfer list processing.**

`StructuredSerializeWithTransfer` iterates the transfer list and checks
whether each object is `[Transferable]`. Adding `Iterator` /
`AsyncIterator` to `[Transferable]` is sufficient for proper instances.

Duck-typed iterators and bare iterables are the problem: they are *not*
`[Transferable]` platform objects. The algorithm currently throws
`DataCloneError` for unrecognized objects in the transfer list. It would
need to attempt `ResolveIteratorForTransfer` as a fallback before
throwing.

Alternatively, restrict cross-realm transfer to proper `Iterator` and
`AsyncIterator` instances only, requiring duck-typed iterators and bare
iterables to be wrapped with `AsyncIterator.from()` before placement in
the transfer list. Simpler, but less convenient.

### Streams Standard (WHATWG) or Companion Spec

The `MessagePort` bridge operations need a spec home:

**Option A: Streams spec.** The Streams spec already defines
`SetUpCrossRealmTransformReadable`, `SetUpCrossRealmTransformWritable`,
`PackAndPostMessage`, etc. in §8.2. The iterator operations follow the
same pattern and could live alongside them, reusing existing helpers.
Downside: iterators are not conceptually part of a streams spec.

**Option B: HTML spec.** Self-contained alongside the transfer steps.
Avoids adding iterator content to the Streams spec but requires
duplicating `PackAndPostMessage` helpers.

**Option C: Standalone spec.** A separate "Transferable Iterators" spec.
Cleanest separation, heaviest organizationally.

### WebIDL

`Iterator` and `AsyncIterator` are ECMAScript intrinsics
(`%IteratorPrototype%`, `%AsyncIteratorPrototype%`), not WebIDL
interfaces. Two options:

- Define WebIDL interfaces mapping to the ECMAScript prototypes, or
- Define transfer steps procedurally in terms of ECMAScript internal
  slots and prototype checks, bypassing WebIDL's `[Transferable]`.

The latter is more practical. The HTML spec already does this for
`ArrayBuffer`.

### Receiving-Side Type Identification

The receiving realm needs to know what type to create from the
`dataHolder`. `StructuredDeserializeWithTransfer` works as follows:

1. For each `transferDataHolder` in the serialized transfer data:
   1. If `transferDataHolder.[[Type]]` is `"ArrayBuffer"` or
      `"ResizableArrayBuffer"`, create the appropriate buffer type.
   2. Otherwise, let `interfaceName` be
      `transferDataHolder.[[Type]]`. Create a new instance of that
      interface in the target realm, then run its transfer-receiving
      steps.

`dataHolder.[[Type]]` is set during transfer steps on the sending side
and consumed on the receiving side to dispatch construction. For
`ReadableStream`, `[[Type]]` is `"ReadableStream"`.

For iterators, the receiving side always produces an `AsyncIterator`
regardless of what went in.

**The sending side's type information is not preserved.** The receiving
side gets a plain `AsyncIterator` backed by a `MessagePort`. The
original object's prototype chain, class identity, additional methods,
and properties are all gone:

```js
class Foo {
  name = 'foo';
  *[Symbol.iterator]() { yield 1; yield 2; yield 3; }
  describe() { return `I am ${this.name}`; }
}

const foo = new Foo();
const iter = foo[Symbol.iterator]();
worker.postMessage(iter, { transfer: [iter] });
```

In the worker, the received object:

- Is **not** a `Foo` -- the worker has no reference to `Foo`.
- Is **not** a `Generator` -- the generator's prototype chain is gone.
- Has **no** `.describe()` method or `.name` property.
- **Is** a plain `AsyncIterator` with only `next()`, `return()`,
  `throw()`, and `[Symbol.asyncIterator]()`.
- `await receivedIter.next()` produces `{ value: 1, done: false }` --
  *values* cross the boundary, *type* does not.

Same principle as `ReadableStream`: transferring a subclass produces a
base `ReadableStream` on the receiving side. Custom methods beyond the
standard `next`/`return`/`throw` protocol do not cross the `MessagePort`
boundary.

This creates several design problems:

**Problem 1: What `[[Type]]` value identifies "AsyncIterator"?**

`AsyncIterator` is not a WebIDL interface -- it is an ECMAScript
intrinsic (`%AsyncIteratorPrototype%`). The algorithm needs a special
case, like the existing `"ArrayBuffer"` and `"ResizableArrayBuffer"`
branches:

```
Otherwise, if transferDataHolder.[[Type]] is "AsyncIterator", then:
  1. Let value be OrdinaryObjectCreate(
     targetRealm.[[Intrinsics]].[[%AsyncIteratorPrototype%]]).
  2. Perform the AsyncIterator transfer-receiving steps given
     transferDataHolder and value.
```

**Problem 2: The receiving-side object must be a proper `AsyncIterator`.**

The object created in the target realm must:

- Have `%AsyncIteratorPrototype%` in its prototype chain.
- Have `Symbol.asyncIterator` return `this`.
- Have `next()`, `return()`, `throw()` installed by the consumer setup.

For `AsyncIterator`, the "empty shell" before transfer-receiving steps
is an ordinary object whose prototype is `%AsyncIteratorPrototype%`
from the target realm. The consumer setup installs the methods.

Concretely:

```
Object.create(AsyncIterator.prototype, {
  [Symbol.asyncIterator]: { value: function() { return this; } }
})
```

...with `next`, `return`, and `throw` installed by the transfer-receiving
steps.

**Problem 3: Both `Iterator` and `AsyncIterator` map to the same
receiving type.**

The receiving side is always `AsyncIterator`, so `dataHolder.[[Type]]`
is always `"AsyncIterator"`. The sync/async distinction matters only on
the sending side (which source handler to set up).

Both transfer steps set the same `[[Type]]`, and a single set of
transfer-receiving steps handles both:

```
Iterator transfer steps:
  ...
  Set dataHolder.[[Type]] to "AsyncIterator".
  Set dataHolder.[[port]] to
    ! StructuredSerializeWithTransfer(port2, « port2 »).

AsyncIterator transfer steps:
  ...
  Set dataHolder.[[Type]] to "AsyncIterator".
  Set dataHolder.[[port]] to
    ! StructuredSerializeWithTransfer(port2, « port2 »).

AsyncIterator transfer-receiving steps (shared):
  1. Let deserializedRecord be
     ! StructuredDeserializeWithTransfer(dataHolder.[[port]],
     the current Realm).
  2. Let port be deserializedRecord.[[Deserialized]].
  3. Perform ! SetUpCrossRealmTransformAsyncIteratorConsumer(
     value, port).
```

**Problem 4: Admitting objects into the transfer list.**

`StructuredSerializeWithTransfer` requires each transfer list item to
have `[[ArrayBufferData]]` or `[[Detached]]`. Proper `Iterator` /
`AsyncIterator` instances have `[[Detached]]` (per the ECMAScript
changes above), so they pass. Duck-typed iterators and bare iterables
have neither slot and throw `DataCloneError` before
`ResolveIteratorForTransfer` runs.

Three approaches:

**(a) Restrict the transfer list to proper instances.** Duck-typed
iterators and bare iterables must be wrapped with `AsyncIterator.from()`
before placement in the transfer list. This requires no changes to the
transfer list validation algorithm and is the simplest path. It does
mean:

```js
// This works (proper Iterator instance with [[Detached]]):
const iter = [1, 2, 3].values();
worker.postMessage(iter, { transfer: [iter] });

// This does NOT work (duck-typed, no [[Detached]]):
const duck = { next() { return { value: 1, done: false }; } };
worker.postMessage(duck, { transfer: [duck] }); // DataCloneError

// This works (wrap first):
const wrapped = AsyncIterator.from(duck);
worker.postMessage(wrapped, { transfer: [wrapped] });
```

**(b) Extend the transfer list validation.** Add a third condition:
"or the object conforms to the iterator protocol (has a callable `next`,
`@@iterator`, or `@@asyncIterator`)." More permissive, but changes a
fundamental invariant — the check is currently purely structural
(internal slots), not behavioral (calling methods).

**(c) Add `[[Detached]]` dynamically.** During
`ResolveIteratorForTransfer`, if the resolved iterator lacks
`[[Detached]]`, add one. Unusual — internal slots are normally fixed at
creation — but satisfies validation without changing the algorithm's
structure.

Approach **(a)** is the most conservative. Under it, the
`ResolveIteratorForTransfer` duck-type detection steps (3-7) only apply
to proper `Iterator`/`AsyncIterator` instances that wrap a duck-typed
iterator — i.e. objects produced by `AsyncIterator.from()`.

**Problem 5: Realm-correct prototype chain.**

The received `AsyncIterator`'s prototype must come from the *target*
realm. This is standard for structured clone (transferred
`ReadableStream` is created "in the current Realm"), but worth stating
explicitly: the Worker receives an `AsyncIterator` with the Worker's
`%AsyncIteratorPrototype%`, not the main thread's.

- `receivedIter instanceof AsyncIterator` works (checks Worker's).
- Iterator Helper methods (`.map()`, `.filter()`, `.take()`, etc.)
  come from the Worker's realm.
- Same as transferred `ReadableStream` — a proper instance of the
  target realm, not a foreign object.

### Summary of Changes by Specification

| Specification | Changes |
|---------------|---------|
| **ECMAScript** | `[[Detached]]` on built-in iterators; detach guards on `.next()`, `.return()`, `.throw()`, `@@iterator`, `@@asyncIterator`; dispose no-op when detached. Depends on `AsyncIterator.from()`. |
| **HTML** | `"AsyncIterator"` branch in `StructuredDeserializeWithTransfer`; transfer steps for `Iterator`/`AsyncIterator` (both set `[[Type]]` to `"AsyncIterator"`); shared transfer-receiving steps; `ResolveIteratorForTransfer`; optionally extend transfer list validation for duck-types. |
| **Streams** *(Option A)* | `SetUpCrossRealmTransformIteratorSource`, `SetUpCrossRealmTransformAsyncIteratorSource`, `SetUpCrossRealmTransformAsyncIteratorConsumer` alongside §8.2. |
| **WebIDL** | None expected — transfer defined procedurally via internal slots, not `[Transferable]`. |

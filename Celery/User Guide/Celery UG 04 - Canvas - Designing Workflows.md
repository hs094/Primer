# UG 04 — Canvas: Designing Workflows

🔑 A signature is a serializable, pickle-safe partial of a task; canvas primitives (`group`, `chain`, `chord`, `map`, `starmap`, `chunks`) compose signatures into workflows.

## Signatures

A signature wraps args, kwargs, and execution options for a task so it can travel over the wire.

```python
from celery import signature

signature('tasks.add', args=(2, 2), countdown=10)
add.signature((2, 2), countdown=10)
add.s(2, 2)                           # shortcut
```

Inspect:

```python
s = add.s(2, 2, debug=True)
s.args      # (2, 2)
s.kwargs    # {'debug': True}
s.options   # {}
```

Invoke:

```python
add.s(2, 2)()                          # in-process → 4
add.s(2, 2).delay()                    # async via worker
add.s(2, 2).apply_async(countdown=1)
```

## Partial args / kwargs

Args from `.s(...)` are **prepended** when called later; kwargs are **merged**:

```python
partial = add.s(2)
partial.delay(4)               # → add(4, 2)

s = add.s(2, 2)
s.delay(debug=True)            # → add(2, 2, debug=True)
```

Set options after the fact:

```python
add.s(2, 2).set(countdown=1)
```

## Immutable signatures (`.si()`)

Use when the parent's return value should *not* be passed in:

```python
add.si(2, 2)
add.apply_async((2, 2), link=reset_buffers.si())
```

## Chain — sequential

```python
from celery import chain

res = chain(add.s(2, 2), add.s(4), add.s(8))()
res.get()                              # → 16

(add.s(2, 2) | add.s(4) | add.s(8))().get()   # pipe operator → 16

c1 = (add.s(4) | mul.s(8))             # partial chain
res = c1(16)
res.get()                              # → 160
```

Walk back through intermediates:

```python
res.parent.get()         # earlier step
res.parent.parent.get()
```

A chain inherits the task id of its final task (Celery 5.4+).

## Group — parallel fan-out

```python
from celery import group

g = group(add.s(2, 2), add.s(4, 4))
res = g()
res.get()                              # → [4, 8]

group(add.s(i, i) for i in range(100))()
```

`GroupResult` API:

```python
result.ready()             # all done?
result.successful()        # all succeeded?
result.failed()            # any failed?
result.waiting()           # any pending?
result.completed_count()
result.revoke()
result.join()              # block, return list of results
```

## Chord — group + callback

A chord is a group (the **header**) plus a callback (the **body**) that runs once every header task completes. The body receives the list of header results.

```python
from celery import chord

callback = tsum.s()
header = [add.s(i, i) for i in range(100)]
result = chord(header)(callback)
result.get()                           # → 9900

chord(add.s(i, i) for i in range(100))(tsum.s()).get()
```

Immutable callback (ignores header results):

```python
chord((import_contact.s(c) for c in contacts),
      notify_complete.si(import_id)).apply_async()
```

Error handling:

```python
@app.task
def on_chord_error(request, exc, traceback):
    print(f'Task {request.id!r} raised error: {exc!r}')

c = (group(add.s(i, i) for i in range(10)) |
     tsum.s().on_error(on_chord_error.s())).delay()
```

⚠️ Chords require a **result backend**, and tasks in the header must not have `ignore_result=True` — the callback needs each result.

## Group → chord upgrade via pipe

A group piped into a single task auto-becomes a chord:

```python
c3 = (group(add.s(i, i) for i in range(10)) | tsum.s())
c3().get()                             # → 90
```

In v5.x, single-item groups inside chains unroll into plain chains; multi-item ones become chords:

```python
chain(add.s(2, 2), group(add.s(1)), add.s(1))
# → add(2, 2) | add(1) | add(1)

chain(add.s(2, 2), group(add.s(1), add.s(2)), add.s(1))
# → add(2, 2) | %add((add(1), add(2)), 1)
```

## Map & Starmap

Single message, applied to a sequence by *one* worker:

```python
tsum.map([list(range(10)), list(range(100))])
# → [45, 4950]

add.starmap(zip(range(10), range(10)))
# → [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

add.starmap(zip(range(10), range(10))).apply_async(countdown=10)
```

Use these when the per-task overhead would dwarf the work.

## Chunks

Split an iterable into N-sized batches and dispatch one task per batch:

```python
res = add.chunks(zip(range(100), range(100)), 10)()
res.get()         # [[0, 2, ...], [20, 22, ...], ...]

g = add.chunks(zip(range(100), range(100)), 10).group()
g.skew(start=1, stop=10)()             # stagger countdowns
```

## Composed example

```python
new_user = (create_user.s() | group(
    import_contacts.s(),
    send_welcome_email.s(),
))
new_user.delay(username='art', first='Art', email='art@x.com')
```

💡 Reach for `chain` for pipelines, `group` for fan-out, `chord` when the next step needs *all* fan-out results, and `map`/`starmap`/`chunks` to amortise messaging cost. Calling primitives → [[Celery UG 03 - Calling Tasks]].

Modern Versions of **Python** have support for "Asynchronous Code" using something called "coroutines", with **async** and **await** syntax.
### More technical details[¶](https://fastapi.tiangolo.com/async/#more-technical-details)

You might have noticed that `await` can only be used inside of functions defined with `async def`.

But at the same time, functions defined with `async def` have to be "awaited". So, functions with `async def` can only be called inside of functions defined with `async def` too.

So, about the egg and the chicken, how do you call the first `async` function?

If you are working with **FastAPI** you don't have to worry about that, because that "first" function will be your _path operation function_, and FastAPI will know how to do the right thing.

But if you want to use `async` / `await` without FastAPI, you can do it as well.

### Write your own async code[¶](https://fastapi.tiangolo.com/async/#write-your-own-async-code)

Starlette (and **FastAPI**) are based on [AnyIO](https://anyio.readthedocs.io/en/stable/), which makes it compatible with both Python's standard library [asyncio](https://docs.python.org/3/library/asyncio-task.html) and [Trio](https://trio.readthedocs.io/en/stable/).

In particular, you can directly use [AnyIO](https://anyio.readthedocs.io/en/stable/) for your advanced concurrency use cases that require more advanced patterns in your own code.

And even if you were not using FastAPI, you could also write your own async applications with [AnyIO](https://anyio.readthedocs.io/en/stable/) to be highly compatible and get its benefits (e.g. _structured concurrency_).

I created another library on top of AnyIO, as a thin layer on top, to improve a bit the type annotations and get better **autocompletion**, **inline errors**, etc. It also has a friendly introduction and tutorial to help you **understand** and write **your own async code**: [Asyncer](https://asyncer.tiangolo.com/). It would be particularly useful if you need to **combine async code with regular** (blocking/synchronous) code.
## Underlying Technical Details
### Path operation functions[¶](https://fastapi.tiangolo.com/async/#path-operation-functions)

When you declare a _path operation function_ with normal `def` instead of `async def`, it is run in an external threadpool that is then awaited, instead of being called directly (as it would block the server).

If you are coming from another async framework that does not work in the way described above and you are used to defining trivial compute-only _path operation functions_ with plain `def` for a tiny performance gain (about 100 nanoseconds), please note that in **FastAPI** the effect would be quite opposite. In these cases, it's better to use `async def` unless your _path operation functions_use code that performs blocking I/O.

Still, in both situations, chances are that **FastAPI** will [still be faster](https://fastapi.tiangolo.com/#performance) than (or at least comparable to) your previous framework.

### Dependencies[¶](https://fastapi.tiangolo.com/async/#dependencies)

The same applies for [dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/). If a dependency is a standard `def` function instead of `async def`, it is run in the external threadpool.

### Sub-dependencies[¶](https://fastapi.tiangolo.com/async/#sub-dependencies)

You can have multiple dependencies and [sub-dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/sub-dependencies/) requiring each other (as parameters of the function definitions), some of them might be created with `async def` and some with normal `def`. It would still work, and the ones created with normal `def` would be called on an external thread (from the threadpool) instead of being "awaited".

### Other utility functions[¶](https://fastapi.tiangolo.com/async/#other-utility-functions)

Any other utility function that you call directly can be created with normal `def` or `async def`and FastAPI won't affect the way you call it.

This is in contrast to the functions that FastAPI calls for you: _path operation functions_ and dependencies.

If your utility function is a normal function with `def`, it will be called directly (as you write it in your code), not in a threadpool, if the function is created with `async def` then you should `await`for that function when you call it in your code.
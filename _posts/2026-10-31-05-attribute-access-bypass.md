---
title: "05 — Attribute Access Bypass"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 05 — Attribute Access Bypass

When `.` is blocked, dunder names are blocked, or `getattr` is missing, you
can still reach attributes through alternative resolution paths.

## `getattr` + `dir()` indexing

Works when `.` is blocked but `getattr` and `dir` survive:

```python
getattr(obj, dir(obj)[i])
```

`python_pyjail_2` ("Dot shall not pass!"):
```python
getout(getattr(getout, dir(getout)[29])[list(getattr(getout, dir(getout)[29]))[7]])
```
- `dir(getout)[29]` == `'func_globals'`
- `list(...)` lists its keys
- index `[7]` == `'passwd'`
- the inner `getattr` reads the password value
- passed to `getout`, which compares and prints the flag.

## String formatting (text leak)

Format strings traverse attributes at runtime; the dunder name lives in a
*string constant*, not in `co_names`, so source-name filters miss it:

```python
print("{0.__globals__}".format(say_hello))
print("{0.__globals__[platform].os}".format(say_hello))
```

Caveat: the result is **text**, not an object. You cannot call or index it.
Use formatting for reconnaissance only, then switch to a real object path
(traceback, `type()`, or `getattr`).

## `type()` with dunder string keys

Build a class whose method names are dunder strings supplied as dict keys:

```python
M = type("M", (), {
    "__enter__": (lambda s: s),
    "__exit__": handler,
})
```

The identifiers `__enter__`/`__exit__` never appear as Python names; they are
string constants. This is how `python_eval_is_evil` defines a context
manager under a `co_names` filter that rejects any name containing `__`.

## Context manager `__exit__` receives the traceback

```python
with M():
    1 / 0
```

Python calls `handler(self, exc_type, exc_value, traceback)` with the real
traceback of the failed block. The traceback is an ordinary parameter named
`tb` — no dunder access needed. From there, walk frames (see
`07-traceback-frame-escape.md`).

## Indexing `__dict__` / `vars`

```python
obj.__dict__["key"]          # if __dict__ reachable
vars(obj)["key"]
```

If `.` is blocked, use `getattr(obj, "__dict__")`.

## `list()` on mappings

```python
list(mapping)                 # keys
list(mapping)[i]              # one key
mapping[list(mapping)[i]]      # value
```

Avoids `.keys()`, `.items()`, `.get()` method calls.

## Operator/function alternatives

- `operator.attrgetter("attr")(obj)` if `operator` is importable.
- `getattr` via `__builtins__.getattr` if the real builtins are reachable.
- `object.__getattribute__(obj, "attr")`.

## When the attribute name itself is filtered

Build the name from parts:
```python
"__" + "globals" + "__"
"get" + "_" + "your" + "_" + "keys"
```
String concatenation keeps the combined name out of `co_names` (each part is a
constant). Used in `python_jail_garbage_collector`.

## Combining techniques

Typical full escape when dots and dunder names are both blocked:
1. Use `type()` with string keys to make a context manager.
2. Raise an exception inside `with`.
3. In `__exit__`, receive `tb`.
4. Walk `tb.tb_frame` / `f_back` / `f_globals`.
5. Find a frame whose globals contain `platform` or real `builtins`.
6. Use real `builtins.open` to read the file.

This is exactly `python_eval_is_evil`.


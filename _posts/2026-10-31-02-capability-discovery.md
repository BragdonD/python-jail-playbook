---
title: "02 — Capability Discovery"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 02 — Capability Discovery

After fingerprinting, enumerate what you can still name and reach. The goal is
to find *one* object reference and *one* way to read an attribute by string.

## Naming things

- `dir()` with no argument → names in the current local scope.
- `dir(obj)` → attribute names of `obj` as a list of strings.
- `globals()` → current global dict (often restricted or empty).
- `locals()` → current local dict.
- `vars(obj)` → `obj.__dict__` when available.
- `__builtins__` → may be `{}` or a restricted dict; still mutable in many
  jails.

If `globals()`/`locals()` are removed, use `dir()` on objects you can name.

## Reading attributes

Normal:
```python
obj.attr
obj.__class__
```

By string (works when `.` is blocked or the name is a dunder):
```python
getattr(obj, "attr")
getattr(obj, dir(obj)[i])          # index into dir() output
```

Without `getattr` (removed), use formatting or `type()`:
```python
"{0.attr}".format(obj)            # returns text only
type("C", (), {"__enter__": ..., "__exit__": ...})  # dunder names as keys
```

## Dictionary keys without methods

```python
list(d)                           # keys, no .keys() needed
list(d)[i]                        # one key by index
```

This is how `python_pyjail_2` reads `func_globals` without a dot:
```python
list(getattr(getout, dir(getout)[29]))[7]   # 'passwd'
```

## Function introspection

```python
f.__code__        # or f.func_code in Py2
f.__globals__     # or f.func_globals
f.__closure__
f.__code__.co_consts
f.__code__.co_varnames
f.__code__.co_names
f.__code__.co_freevars
```

## Module introspection

```python
m.__dict__
m.__file__
m.__name__
m.__package__
```

## Type/class introspection

```python
obj.__class__
cls.__bases__
cls.__mro__
cls.__dict__
cls.__subclasses__()      # on object, lists all loaded subclasses
```

## Reaching globals through a function

Any function defined in a module keeps that module's globals:
```python
some_function.__globals__     # real module namespace
```
In `python_eval_is_evil`, `say_hello` is defined in the service module, so
`say_hello.__globals__` contains `platform`, real `builtins`, `sandbox`, etc.

## Object graph navigation checklist

- [ ] Can I name `object`? (via `().__class__.__base__` or `type` tricks)
- [ ] Can I call `object.__subclasses__()`?
- [ ] Does any subclass expose `__init__.__globals__`?
- [ ] Does that globals dict contain `sys`, `os`, `warnings`, `linecache`,
      `platform`, or `builtins`?
- [ ] Can I reach a function defined in the service module?
- [ ] Can I trigger an exception and capture its traceback?

If several are yes, you have a bridge (see `03-object-introspection.md`).

## When `.` is completely blocked

Fall back chain:
1. `getattr(obj, name)` with `name` from `dir()` indexing or a string literal.
2. `getattr(obj, dir(obj)[i])`.
3. String formatting for *text* leaks: `"{0.__globals__}".format(f)`.
4. `type()` with dunder names as dictionary keys (strings, not identifiers).
5. Context manager `__exit__` to receive a traceback as a parameter.

## When `getattr` is also blocked

Use formatting to *print* attributes (text only), then look for another path.
In `python_eval_is_evil`, formatting alone cannot yield a callable; the real
escape uses the traceback route (`07-traceback-frame-escape.md`).

## When `dir` is blocked

Try `obj.__dict__`, `vars(obj)`, `type(obj).__mro__`, or known attribute
names as string literals with `getattr`.

## Persistence check

Test whether mutations persist across evaluations:
```python
__builtins__["probe"] = 42
```
then next line:
```python
probe
```
If it resolves, you can stage long payloads (see `06-filter-bypass.md`).
`python_jail_exec` relies on this.


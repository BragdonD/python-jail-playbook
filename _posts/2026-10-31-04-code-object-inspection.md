---
title: "04 — Code Object Inspection"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 04 — Code Object Inspection

When a secret is compared inside a function, the function's code object
contains the secret as a constant or local. Reading it avoids triggering
filters or needing the correct argument.

## Reaching a code object

Python 2:
```python
f.func_code
f.func_code.co_consts
f.func_code.co_varnames
f.func_code.co_names
```

Python 3:
```python
f.__code__
f.__code__.co_consts
f.__code__.co_varnames
f.__code__.co_names
f.__code__.co_freevars
```

Also reachable without attribute spelling via `getattr`:
```python
getattr(f, dir(f)[...])   # 'func_code' / '__code__'
```

## What code objects expose

- `co_consts` — tuple of constants compiled into the function: literals,
  nested code objects, docstrings.
- `co_varnames` — local variable names (including parameters).
- `co_names` — global names referenced.
- `co_freevars` / `co_cellvars` — closure variables.
- `co_code` — raw bytecode (rarely needed; use `dis`).
- `co_argcount`, `co_flags`.

## Pattern: secret as a constant

`python_pyjail_1`:
```python
>>> print `exit.func_code.co_consts`
(None, '<redacted-flag>', -1, 'cat .passwd', 'You cannot escape !')
>>> exit(exit.func_code.co_consts[1])
Well done flag : <redacted-flag>
```

The jail blocked string literals and `__` attributes but allowed single-dot
`exit.func_code`. Inspecting constants revealed the secret argument.

## Pattern: secret built from constants

`python_jail_garbage_collector`: `get_your_keys(magic_word)` compares to a
string built by XORing a list of integers with a fixed key:

```python
fk.__code__.co_varnames   # ('magic_word', 'r', 'f', 'x')
fk.__code__.co_consts     # contains 19 numbers ~291105..291171 and strings
```

Decompile mentally:
```python
r = ''
f = [291105, 291152, 291104, ...]
for x in f:
    r += chr(x ^ 291171)
# r == '<redacted-secret>'
```

Both operands are in the code object, so the check is trivially invertible.

## Pattern: decompiling without executing

- `dis.dis(f)` shows bytecode.
- `inspect.getsource(f)` may work if source is available (often not in jails).
- Walk `co_consts` recursively: nested functions/lambdas are code objects too.

## Reading nested code objects

```python
for c in f.__code__.co_consts:
    if hasattr(c, "co_consts"):
        print(c.co_consts)
```

This matters for jails that check `co_names` recursively
(`python_eval_is_evil`'s `check_code`).

## When code objects are the goal, not the bridge

Sometimes the secret is not in constants but computed. Still inspect:
- `co_varnames` tells you local names (e.g., `magic_word`, `r`, `f`, `x`).
- `co_names` tells you which globals the function touches — a map of what
  exists in its module.
- Bytecode reveals branches and comparisons.

## Combining with introspection

Once you reach the service module via `gc.get_objects()`, its functions are
attributes of `mainmod.__dict__`. Build the function name with string
concatenation if `_`/names are filtered:
```python
fk = mainmod.__dict__['get'+'_'+'your'+'_'+'keys']
print(fk.__code__.co_consts)
```

## Caveats

- Code objects reflect *compiled* code; optimized builds may inline constants.
- `co_consts` order is implementation-defined but stable for a given build.
- In Python 3.11+, `co_consts` layout and adaptive bytecode change; `dis`
  output differs but constants remain readable.
- Reading a code object never executes the function, so filters on `eval`/
  `exec` do not apply.


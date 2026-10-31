---
title: "06 — Filter Bypass"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 06 — Filter Bypass

Filters fall into four families: length, character, word/name, and structural
(code-object) filters. Match the bypass to the filter.

## Length truncation

Problem: a needed expression is longer than the input limit
(`python_jail_exec`: 35 chars blocks `().__class__.__base__.__subclasses__()`).

Solution: stage across multiple evaluations when the jail reuses a mutable
`__builtins__` dict.

Pattern:
```python
__builtins__['B'] = __builtins__
B['u'] = B.update
u(o=().__class__.__base__)          # store object
u(subs=o.__subclasses__())          # store list
u(w=subs[59])                       # store bridge class
...
```

Each line stays under the limit; `u(...)` is 2 chars plus the assignment.
Alias `B` and `u` to shrink later lines.

General staging recipe:
1. Alias `__builtins__` to a short name.
2. Alias `dict.update` to a short name.
3. Store each intermediate in `__builtins__` under a short key.
4. Final expression uses the stored names.

Also consider:
- Abbreviating attribute chains by storing intermediate objects.
- Using `exec`/`eval` if available (rare in jails).
- Multi-line protocols: some jails read until a blank line, allowing longer
  compound statements in one submission.

## Character filters

| Blocked | Bypass |
|---|---|
| `.` | `getattr(obj, name)`, `dir()` indexing, formatting |
| `_` | build names from parts without underscore, or use `getattr` with strings containing `_` (the filter usually targets source chars, not string contents) |
| quotes | use `chr()` to build strings, or triple quotes if only single/double blocked |
| spaces | not usually blocked; if so, use tabs or parentheses |
| parentheses | very hard; try comprehensions, decorators, `with`, lambdas without parens (rare) |
| brackets | use `getattr`, `list()` indexing via `__getitem__`? `obj.__getitem__(i)` |
| braces | avoid dict literals; use `type(name, bases, dict)` or `dict([(k,v)])` |
| comma | use semicolons? Python has no statement separator except newline; use list/loop constructs |

Always test which characters are *actually* blocked; filters vary.

## Word / name filters

- Blocked identifier: use `getattr(obj, "name")` with the name as a string
  constant.
- Blocked substring in `co_names`: move the name into a string constant or
  build it with concatenation (`"os" + ".system"`).
- Blocked `__` in any name: use formatting, `type()` with string keys, or the
  traceback route.
- Blacklisted module names in `sanitize()`: concatenate (`'g'+'c'`, `sys` via
  attribute of another object rather than naming it).

## Structural (code-object) filters

`python_eval_is_evil` walks `co_consts` recursively and rejects any code
object whose `co_names` contains `__`:

```python
def check_code(code):
    if "__" in str(code.co_names):
        raise Exception("Try again")
    for const in code.co_consts:
        if hasattr(const, "co_names"):
            check_code(const)
```

Bypasses:
- Keep dunder names out of `co_names` by putting them in string constants.
- Use `str.format` attribute traversal (format spec is a string constant).
- Use `type(name, bases, dict)` where dunder names are dict keys (strings).
- Use `with` + `__exit__` to receive a traceback without naming `__traceback__`.
- Lambdas and comprehensions create code objects too; keep their `co_names`
  clean.

## Blacklist evasion via indirection

Instead of naming a blocked module, reach it through an object:
```python
platform = frame.f_globals["platform"]   # not the literal 'sys'
osmod = platform.os                     # not 'os' as a name
```
The filter sees `platform`, `osmod`, `frame`, `f_globals` — none blocked.

## When a filter is an allowlist (restricted builtins)

If only a small set of builtins is exposed, find which exposed function is
defined in a real module:
- `say_hello` in `python_eval_is_evil` is defined in the service module, so
  `say_hello.__globals__` leaks everything.
- `exit` in `python_pyjail_1` exposes `func_code`.

Any exposed callable defined in a module is a bridge.

## Recon before bypass

Print the filter's exact rejection to identify its type:
- `Try again` after compile → `co_names` filter.
- `No no no..` during exec → runtime blacklist.
- Silent truncation → length filter.
- `You're in jail dude ...` → character filter.

## Caveats

- Filters can stack (length + chars + names). Solve one at a time.
- A bypass that works locally may fail remotely if the filter differs; always
  test against the real service.
- Multi-line protocols require exact blank-line termination.


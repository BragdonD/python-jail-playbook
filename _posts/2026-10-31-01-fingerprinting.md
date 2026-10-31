---
title: "01 — Fingerprinting"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 01 — Fingerprinting

Before escaping, characterize the jail. The error messages and behavior tell
you which defenses exist and which Python version you are targeting.

## Python version

| Probe | Python 2 | Python 3 |
|---|---|---|
| `1/0` | `ZeroDivisionError: integer division or modulo by zero` | `ZeroDivisionError: division by zero` |
| `True` | `NameError: name 'True' is not defined` (old jails) | defined |
| `print 1` | works | SyntaxError |
| `print(1)` | works | works |
| `input("x")` | evaluates input (`eval`) | returns string |
| `raw_input` | exists | absent |
| `xrange` | exists | absent |
| `unicode` | exists | absent |
| `basestring` | exists | absent |
| f-strings | SyntaxError | 3.6+ |

Also check `sys.version` if reachable later; until then use the probes above.

## Prompt and I/O protocol

- Interactive REPL: input echoed, result prefixed (`Result:`, `Out:`).
- `exec`/`eval` loop: often reads until a blank line, then compiles.
- SSH challenge: banner may state rules ("Dot shall not pass!", "Everything is
  in exit()").
- Truncation: send a long literal and measure. `python_jail_exec` truncates to
  35 characters.

## Filters to detect

Test each in isolation and record the exact rejection text.

- **Character filters:** `.`, `_`, `(`, `)`, `[`, `]`, `{`, `}`, `"`, `'`,
  space, comma, semicolon, newline, backslash.
- **Word filters:** `import`, `open`, `eval`, `exec`, `compile`, `input`,
  `globals`, `locals`, `getattr`, `__import__`, `os`, `sys`, `gc`, `builtins`,
  `platform`, `subprocess`.
- **Name filters:** any identifier containing `__` (checked via `co_names`).
- **Length filters:** input truncated to N characters.
- **AST/code filters:** `check_code` walking `co_consts` recursively.

## Useful probes

```python
1+1
().__class__
"".__class__
(1).__class__
dir()
globals()
locals()
__builtins__
help()
exit()
quit()
```

## Error oracle

Error messages are information:
- `TypeError: exit() takes exactly 1 argument (0 given)` → function signature.
- `NameError: name 'True' is not defined` → Py2 or restricted globals.
- `You're in jail dude ...` → character filter.
- `Try again` → name/word filter.
- `No no no..` → blacklist.

## Version-specific jail notes

- Python 2 jails often expose `func_code`, `func_globals`, `im_func`,
  `im_class`. Python 3 uses `__code__`, `__globals__`, `__func__`.
- Python 2 `input()` is `eval`; Python 3 `input()` is `raw_input`.
- `print` is a statement in Py2, a function in Py3.
- `str.format` attribute traversal works in both.
- Traceback objects (`tb_frame`, `f_back`, `f_globals`) exist in both.
- `gc.get_objects()` exists in both.

## What to record

Create a small table for the target:

| Property | Value |
|---|---|
| Python version | |
| Input protocol | |
| Max input length | |
| Blocked chars | |
| Blocked words | |
| Available builtins | |
| Available attributes | |
| Error style | |

This table drives technique selection.


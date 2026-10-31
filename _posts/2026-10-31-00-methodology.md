---
title: "00 — Methodology"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 00 — Methodology

A reproducible loop for any Python jail. Do not skip steps; each step produces
information the next step needs.

## Phase 0 — Observe the interface

1. Connect (SSH/nc/HTTP). Note the prompt (`>>>`, `>>> `, `calc>`, none).
2. Send trivial input: `1+1`, `print(1)`, `help()`. Record echo, result
   prefix (`Result:`, `Error:`), and whether output is repr'd or printed.
3. Send syntax errors and exceptions. Does it echo input? Does it show
   tracebacks or only the exception message?
4. Send a blank line. Does it terminate a multi-line submission?
5. Send `exit`, `quit`, `EOF` (Ctrl-D). Note behavior.

## Phase 1 — Fingerprint

- Python version: `1/0` (ZeroDivisionError text), `True` (NameError in Py2),
  `print X` vs `print(X)`, `input` vs `raw_input`.
- Truncation: send a long known string; measure the cutoff.
- Blocked characters: try `.`, `_`, `(`, `)`, `[`, `]`, `{`, `}`, `"`, `'`,
  ` `, newline, `;`, `,`.
- Blocked words: try `import`, `open`, `eval`, `exec`, `os`, `sys`, `gc`,
  `globals`, `locals`, `getattr`, `__import__`.
- Builtins present: `dir()`, `globals()`, `locals()`, `vars()`, `__builtins__`.
- Attribute filter: `().__class__`, `"".__class__`, `(1).__class__`.

Record all of this before attempting escapes.

## Phase 2 — Capability discovery

Goal: find *any* object reference and *any* way to name attributes.

- `dir(obj)` lists attribute names as strings.
- `getattr(obj, name)` reads an attribute by string.
- `list(mapping)` lists keys of a dict without a method call.
- `obj.__dict__` / `vars(obj)` exposes instance/class attributes.
- `type(obj)`, `obj.__class__`.
- Function objects: `f.__code__`/`f.func_code`, `f.__globals__`/
  `f.func_globals`, `f.__closure__`.
- Module objects: `m.__dict__`, `m.__file__`, `m.__name__`.

If `.` is blocked, use `getattr` + `dir` indexing. If `getattr` is missing,
use format strings or `type()` with string keys.

## Phase 3 — Find a bridge

A bridge is a path from something you can name to something powerful
(`sys`, `os`, `builtins`, a frame, the service module).

Standard bridges:
1. `().__class__.__base__.__subclasses__()` → a pure-Python class →
   `__init__.__globals__` → `sys`/`warnings`/`linecache`.
2. `sys.modules['gc']` → `gc.get_objects()` → service module by `__file__`.
3. A function defined in the service module → its `__globals__` → real
   `builtins`/`platform`.
4. An exception traceback → frame chain → `f_globals`.

Always search candidates instead of assuming indices.

## Phase 4 — Reach the target

Once you have a real module namespace or `builtins`:
- Read files: `builtins.open(path).read()` or `os.open`/`os.read`.
- Execute: `os.system(cmd)`, `subprocess`, or `builtins.eval`/`exec`.
- Inspect: read `.passwd`, env vars (`os.environ`), or call the challenge's
  own verification function with a decoded secret.

## Phase 5 — Handle filters

If the escape fails because of a filter, identify which filter fired (error
text, `Try again`, truncation) and apply the matching bypass from
`06-filter-bypass.md`. Re-run from Phase 2 with the new capability.

## Phase 6 — Stabilize and document

- Reduce the payload to the shortest working form.
- Record exact remote output as evidence.
- Note environment-specific values (indices, offsets) and how to rediscover
  them.

## Anti-patterns

- Hardcoding subclass or `dir()` indices without printing candidates first.
- Assuming the jail's `__builtins__` is the real one.
- Treating a string result from formatting as a usable object.
- Forgetting that multi-line submission protocols may require a blank line.
- Using Python 3-only features against a Python 2 jail (and vice versa).


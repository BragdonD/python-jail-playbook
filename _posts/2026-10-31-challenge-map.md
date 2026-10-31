---
title: "Challenge Map"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# Challenge Map

Which Root-Me Python challenge exercises which technique.

| Challenge | Primary technique | Files |
|---|---|---|
| `python_pyjail_1` | Code-object constant inspection (`co_consts`) | `04-code-object-inspection.md` |
| `python_pyjail_2` | Dotless attribute access via `getattr` + `dir()` indexing | `05-attribute-access-bypass.md` |
| `python_jail_exec` | Subclass traversal + `warnings`/`linecache`/`os` + length staging | `03-object-introspection.md`, `06-filter-bypass.md` |
| `python_jail_garbage_collector` | Subclass → `sys` → `gc` → service module by `__file__`; code-object XOR decode | `03-object-introspection.md`, `04-code-object-inspection.md` |
| `python_eval_is_evil` | `co_names` dunder bypass via format/type; traceback frame escape | `05-attribute-access-bypass.md`, `06-filter-bypass.md`, `07-traceback-frame-escape.md` |
| `python2_input` | Python 2 `input()` as `eval` | `09-python2-input-eval.md` |

## Technique → challenge cross-reference

- **Fingerprinting** (`01`): all.
- **Capability discovery** (`02`): all, especially `python_pyjail_2` and `python_eval_is_evil`.
- **Object introspection** (`03`): `python_jail_exec`, `python_jail_garbage_collector`.
- **Code-object inspection** (`04`): `python_pyjail_1`, `python_jail_garbage_collector`.
- **Attribute-access bypass** (`05`): `python_pyjail_2`, `python_eval_is_evil`.
- **Filter bypass** (`06`): `python_jail_exec` (length), `python_eval_is_evil` (co_names).
- **Traceback escape** (`07`): `python_eval_is_evil`.
- **Files/shell** (`08`): `python_jail_exec`, `python_eval_is_evil`, `python2_input`.
- **Python 2 input eval** (`09`): `python2_input`.

## Suggested learning order

1. `python_pyjail_1` — easiest; teaches code-object inspection.
2. `python_pyjail_2` — teaches dotless attribute access.
3. `python_jail_exec` — teaches subclass traversal and staging.
4. `python_jail_garbage_collector` — teaches gc/service-module recovery and
   XOR decompilation.
5. `python_eval_is_evil` — hardest; combines filter bypass and traceback
   escape.
6. `python2_input` — different bug class (eval misuse), good contrast.

## Not covered here

- Jinja2 SSTI (`web_server/python_blind_ssti_filters_bypass`) — see that
  challenge's writeup; shares object traversal but is a template engine, not a
  Python jail.
- OS-level jails (AppArmor, seccomp) — see `apparmor_jail_intro`.
- JavaScript jails — see `javascript_jail`.


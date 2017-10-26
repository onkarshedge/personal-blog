---
title: "Python Getattr"
date: 2017-10-25T19:00:00+05:30
tags: ["python"]
draft: true
---

Today I learnt about `__getattr__` in python. Usually I have seen code where we have
db.object.save() or api.attribute and pressing Cmd+B in Intellij doesn't show anything or autocomplete because that
attribute doesn't exist. So whenever an attribute doesn't exist `__getattr__(self, name)` this method is called.
Here is an example.

```
class foo:
     def __init__(self):
             self.a = "koo"
     def __getattr__(self,name):
             print("attribute called")

>>> m=foo()
>>> m.a
'koo'
>>> m.b
attribute called
```
and `getattr` to get the attribute. If the string name is a constant, say 'foo', getattr(obj, 'foo') is exactly the same thing as obj.foo.
```
>>> getattr(m, "a")
'koo'
```

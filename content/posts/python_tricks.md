---
author: ["Paul Wong"]
title: "Python Tricks"
date: "2019-03-18"
description: "Python Tips and Tricks"
summary: "This post has some useful Python tips and tricks"
tags: ["python", "sorting", "snippet"]
categories: ["python", "snippet"]
ShowToc: true
TocOpen: true
---

[History of Python Wiki](https://en.wikipedia.org/wiki/History_of_Python)

## The Zen of Python

```python
import this
```

```
The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
```

## Python Class Descriptors

### Build in descriptors

#### `type()` - class type

```python
>>> class MyClass:
...     def __init__(self):
...         self.data = 0
...     def add_one(self):
...         self.data += 1
...
>>> mc = MyClass()
>>> type(mc)
<class '__main__.MyClass'>
>>>
```

#### `dir()` - class directory

```python
>>> dir(mc)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__getstate__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'add_one', 'data']
>>>
```

#### `vars()` - class variables

```python
>>> vars(MyClass)
mappingproxy({'__module__': '__main__', '__init__': <function MyClass.__init__ at 0x10ff374c0>, 'add_one': <function MyClass.add_one at 0x10ff37740>, '__dict__': <attribute '__dict__' of 'MyClass' objects>, '__weakref__': <attribute '__weakref__' of 'MyClass' objects>, '__doc__': None})
>>>
>>> vars(mc)
{'data': 0}
>>>
```

#### `id()` - memory location

```python
>>> hex(id(MyClass))
'0x7fe6fa806260'
>>>
>>> hex(id(mc))
'0x10ff38b50'
>>>
```

## Using F strings

Formatting a string is probably the number one task you’ll need to do almost daily. There are various ways you can format strings in Python; my favorite one is using f strings.

```python
# Formatting strings with f string.
str_val = 'books'
num_val = 15
print(f'{num_val} {str_val}') # 15 books
print(f'{num_val % 2 = }') # 1
print(f'{str_val!r}') # books

# Dealing with floats
price_val = 5.18362
print(f'{price_val:.2f}') # 5.18

# Formatting dates
from datetime import datetime;
date_val = datetime.utcnow()
print(f'{date_val=:%Y-%m-%d}') # date_val=2021-09-24
```

## Unfold List

Builds a list, using an iterator function and an initial seed value.

- The iterator function accepts one argument (seed) and must always return a list with two elements ([value, nextSeed]) or False to terminate.
- Use a generator function, fn_generator, that uses a while loop to call the iterator function and yield the value until it returns False.
- Use a list comprehension to return the list that is produced by the generator, using the iterator function.

Code

```python
def unfold(fn, seed):
  def fn_generator(val):
    while True:
      val = fn(val[1])
      if val == False: break
      yield val[0]
  return [i for i in fn_generator([None, seed])]
```

Use

```python
f = lambda n: False if n > 50 else [-n, n + 10]
unfold(f, 10) # [-10, -20, -30, -40, -50]
```

## Sorting Dictionaries

```python
>>> people = {3: "Jim", 2: "Jack", 4: "Jane", 1: "Jill"}
>>> print(f"unsorted     = {people}")
unsorted     = {3: 'Jim', 2: 'Jack', 4: 'Jane', 1: 'Jill'}
>>>
>>> key_sorted = dict(sorted(people.items()))
>>> print(f"{key_sorted   = }")
key_sorted   = {1: 'Jill', 2: 'Jack', 3: 'Jim', 4: 'Jane'}
>>>
>>> value_sorted = dict(sorted(people.items(), key=lambda item: item[1]))
>>> print(f"{value_sorted = }")
value_sorted = {2: 'Jack', 4: 'Jane', 1: 'Jill', 3: 'Jim'}
>>>
```

## Delete First Item in Dictionary

```python-repl=
del value_sorted[next(iter(value_sorted))]
```

## Indexing Combonations

```python
>>> import itertools
>>> a = [7, 5, 5, 2]
>>> list(itertools.combinations(enumerate(a), 2))
[((0, 7), (1, 5)), ((0, 7), (2, 5)), ((0, 7), (3, 3)), ((1, 5), (2, 5)), ((1, 5), (3, 3)), ((2, 5), (3, 3))]
>>> list((i,j) for ((i,_),(j,_)) in itertools.combinations(enumerate(a), 2))
[(0, 1), (0, 2), (0, 3), (1, 2), (1, 3), (2, 3)]
>>>
```

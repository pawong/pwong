---
author: ["Paul Wong"]
title: "Python Sorting"
date: "2021-01-23"
description: "Some quick python sorting solutions."
summary: "This post show some sorting solutions for Python."
tags: ["python", "sorting", "snippet"]
categories: ["python", "snippet"]
ShowToc: true
TocOpen: true
---

### Sort dictionary by key and value

```python
people = {3: "Jim", 2: "Jack", 4: "Jane", 1: "Jill"}

key_sorted = dict(sorted(people.items()))
print(f"{key_sorted   = }")

value_sorted = dict(sorted(people.items(), key=lambda item: item[1]))
print(f"{value_sorted = }")
```

Output

```
key_sorted   = {1: 'Jill', 2: 'Jack', 3: 'Jim', 4: 'Jane'}
value_sorted = {2: 'Jack', 4: 'Jane', 1: 'Jill', 3: 'Jim'}
```

### Sort a list of dictionaries

The next set of everyday list tasks is sorting. Depending on the data type of the items included in the lists, we’ll follow a slightly different way to sort them. Let’s first start with sorting a list of dictionaries.

```python
dicts_lists = [
    {
        "Name": "James",
        "Age": 20,
    },
    {
        "Name": "May",
        "Age": 14,
    },
    {
        "Name": "Katy",
        "Age": 23,
    },
]

# There are different ways to sort that list
# 1 - Using the sort/ sorted function based on the name or age
dicts_lists.sort(key=lambda item: item.get("Name"))
print(dicts_lists)
dicts_lists.sort(key=lambda item: item.get("Age"))
print(dicts_lists)

# 2 - Using itemgetter module based on name
from operator import itemgetter

f = itemgetter("Name")
dicts_lists.sort(key=f)
print(dicts_lists)
```

Output

```
[{'Name': 'James', 'Age': 20}, {'Name': 'Katy', 'Age': 23}, {'Name': 'May', 'Age': 14}]
[{'Name': 'May', 'Age': 14}, {'Name': 'James', 'Age': 20}, {'Name': 'Katy', 'Age': 23}]
[{'Name': 'James', 'Age': 20}, {'Name': 'Katy', 'Age': 23}, {'Name': 'May', 'Age': 14
```

### Sort a list based on another list

Sometimes, we may want or need to use one list to sort another. So, we’ll have a list of numbers (the indexes) and a list that I want to sort using these indexes

```python
a = ['blue', 'green', 'orange', 'purple', 'yellow']
b = [3, 2, 5, 4, 1]

#Use list comprehensions to sort these lists
sortedList =  [val for (_, val) in sorted(zip(b, a), key=lambda x: x[0])]
```

Output

```
['yellow', 'green', 'blue', 'purple', 'orange']
```

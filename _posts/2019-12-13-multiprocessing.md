---
type: posts
title: Python Multiprocessing
author: Amutheezan Sivagnanam
category: Computer Architecture
date: 2019-12-13
last_modified_at: 2020-02-07
tags:
- multiprocessing

---

I have developed a basic Python library to provide abstraction for parallel programming.
This library is based on the built-in ```multiprocessing``` library in Python and the third-party library ```ray```.
In this blog post, I will explain in detail the implementations in addition to the existing
documentation.

**CustomMP** is an abstraction of the ```multiprocessing``` library with a ```SharedList```.
```SharedList``` is a generalization of the commonly used ```Manager``` and ```Queue``` pair
used to keep track of tasks and results. The ```SharedList``` implementation handles this,
avoiding redundant re-implementation of the same ```multiprocessing``` structure each
time it is used. The motivation for this implementation is based on a discussion I
had on Stack Overflow regarding the proper way to implement a ```SharedList```.

Sample use case of **CustomMP**:

```python
from pyparallel.CustomMP import CMPSystem

limit = 4
args = (1,)
cmp_sys = CMPSystem(limit)
cmp_sys.add_proc(func=child_func, args=(args,))
contents = cmp_sys.run()
print(len(contents))
```

### **References**

1. [https://stackoverflow.com/questions/58927768/what-is-proper-way-to-use-shared-list-in-multiprocessing](https://stackoverflow.com/questions/58927768/what-is-proper-way-to-use-shared-list-in-multiprocessing)
2. [https://github.com/Amutheezan/PyParallel](https://github.com/Amutheezan/PyParallel)

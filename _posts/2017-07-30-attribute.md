---
type: posts
title: Which is the best multi-attribute sorting for python Array?
author: Amutheezan Sivagnanam
category: Tech Issues
date: 2017-07-30
tags:
- sorting
- python

---

#### QUESTION

This post is compiled based on the following
stackoverflow [question and answer section](https://stackoverflow.com/questions/46126792/quickest-sorting-mechanism-for-sorting-by-multiple-predicates-in-python).

I came across the issue while I need to sort some list of prediction probability and weights of tweets for adding them
as iteration list for semi-supervised methodology called "Self-training".

When I was randomly picking the first, n number of tweets, it cost around 2 hrs for about 40 iterations and 5 hrs for
around 100 iterations.

I have implemented following changes to do the sorting,

```python
a = [[1, 0.7, 1], [4, 0.8, 1], [5, 0.8, 0.99], [11, 0.9, 0.98]]
b = sorted(a, key=lambda x: x[1], reverse=True);
c = sorted(b, key=lambda x: x[2], reverse=True);
```

But it takes more than 5 hrs for 20 iterations itself, and I searched in StackOverflow and obtained two sets of
formulas, tested them, and compared the time differences. following are those two sets of formulas

```python
s = sorted(a, key=lambda x: (x[2], x[1]), reverse=True);
i = sorted(a, key=operator.itemgetter(2, 1), reverse=True);
```

Out of these, one will return the result quickly for a 5D Array with a size of around 20,000 ???

<strong> ANSWER: </strong>
These three implementations are simply equal and have the same overhead of doing the sorting. Thus, it takes the same amount of time. So these three options are equally replaceable with othe

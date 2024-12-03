---
title: "Advent of code: 2024"
date: 2024-12-02T12:58:32+02:00
draft: false
description: "I'm trying to do advent-of-code this year. Let's see how this goes."
---

I'm trying to do [Advent-of-code](https://adventofcode.com/2024) this year.
Let's see how this goes. I'll collect everything I manage here as a single
page that will be updated as I go along. I'll try to do it in Python and without
doing any imports and with as little lines (newlines for readability) as I
possibly can, meaning it will at some places get very messy...

# Day 1
[Challenge](https://adventofcode.com/2024/day/1)

```python
with open('./tmp/advent-of-code/input-day-1.txt') as f:
    data = f.read()

# For testing
#data = "1   2\n3   4\n2   1\n"

# parse input to list of lists of ints
lists = [
  [int(val) for val in row.split(" ") if val > ""] \
    for row in data.split("\n") if row > ""
]

# transpose without numpy
list1, list2 = [
  [row[i] for row in lists] \
    for i in range(len(lists[0]))
]

# answer "total distance"
total_distance = sum(
  [
    abs(n[0] - n[1]) \
      for n in zip(
        sorted(list1),
        sorted(list2)
      )
  ]
)

# answer "similarity score"
similarity_score = sum(
  [el*list2.count(el) for el in list1]
)
```

# Day 2
[Challenge](https://adventofcode.com/2024/day/2)


```python
with open('./tmp/advent-of-code/input-day-2.txt') as f:
    data = f.read()

# For testing
#data = "1   2   6   7\n3   4    5    7\n5   3    2    1\n"

reports = [
  [int(val) for val in row.split(" ") if val > ""] \
    for row in data.split("\n") if r > ""
]

# answer safe reports. MUST:
# - levels order should be the same as asc or desc ordered list,
# - number of unique levels in report should be the same as number of levels in report
# - max abs value between two consequtive levels must be less than or equal to 3   
safe_reports = len(
  list(
    filter(
      lambda report: (
        (
          report == sorted(report) or \
          report == sorted(report, reverse=True)
        ) and \
        len(set(report)) == len(report) and \
        max([abs(report[i]-report[i+1]) for i in range(len(report)-1)]) <= 3
      ), reports
    )
  )
)

# ugh... I should have done this in a more readable/comprehensible way before
# but here we go

# filter unsafe reports based on previous
unsafe_reports = list(
  filter(
    lambda report: (
      (
        report == sorted(report) or \
        report == sorted(report, reverse=True)
      ) and \
      len(set(report)) == len(report) and \
      max([abs(report[i]-report[i+1]) for i in range(len(report)-1)]) <= 3
    )!=True, reports
  )
)

# from the unsafe levels reports create combinations so slicing original leaving
# out only one level per iteration so
# e.g. [1,2,3,4,5] becomes
# [
#  [2,3,4,5],
#  [1,3,4,5],
#  [1,2,4,5],
#  [1,2,3,5],
#  [1,2,3,4],
# ]
# this is done in the lambda iterable input clause - we're still looping report
# by report but further looping it inside lambda
# within lambda why are testing the same previous three clauses for each
# permutation for report (order, number of unique, and difference of 3)
# recording True|False for each permutation and then filtering per report only
# those which get a True, True, True in response.
# Count all reports that got (at least) one True,True,True.
# Finally add it to the original safe_reports number

extra_safe_reports = list(
  map(
    lambda reports: [
      [
        report == sorted(report) or report == sorted(report, reverse=True),
        len(set(report)) == len(report),
        max([abs(report[i]-report[i+1]) for i in range(len(report)-1)]) <= 3
      ] for report in reports
    ].count([True,True,True]) > 0, \
      [
        [
          [unsafe_report[j] for j in range(len(unsafe_report)) if j!=i ] \
            for i in range(len(unsafe_report))
        ] \
          for unsafe_report in unsafe_reports
      ]
  )
).count(True)

total_safe_reports = safe_reports + extra_safe_reports

# (facepalm) most probably with normal procedural planning this would be
# (and look) much simpler
```

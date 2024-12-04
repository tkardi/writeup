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


## <a id="day1" href="#day1">Day 1</a>
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

## <a id="day2" href="#day2">Day 2</a>
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

## <a id="day3" href="#day3">Day 3</a>
[Challenge](https://adventofcode.com/2024/day/3)

```python
with open('./advent-of-code/input-day-3.txt') as f:
    lines = f.read()

# this looks like job for regex, but trying to stay true to the idea of no
# imports... (and i already know I'm going to weep when doing the second part)

# answer to "add all multiplications"
add_multiplication = sum(
  list(
    map(
      lambda int_pairs: int(int_pairs[0])*int(int_pairs[1]),
      list(
        filter(
          lambda possible_int: \
            possible_int[0].isdigit() and possible_int[1].isdigit(),
            [
              [
                token[0][1:],
                token[1][:token[1].find(")")]
              ] \
                for token in [
                  token.split(",") \
                    for token in list(
                      filter(
                        lambda possible_token: \
                          possible_token.startswith("(") and \
                          possible_token.find(",") > 1 and \
                          possible_token.find(")") > 3 and \
                          possible_token.find(",") < possible_token.find(")")-1,
                          lines.split("mul")
                      )
                    )
                ]
            ]
        )
      )
    )
  )
)

# ... and after seeing the second task, yes, my thoughts were correct in the
# beginning. BUT what the what... we are here to solve problems as they arise
# to exactly to the amount that is needed :D

# so with the do-s and don't-s. the general idea would be to:
# a) split the whole string at don't()-s to a list
# b) filter out only those that have the a do()-instruction in them
#    (meaning "".find("do()") > -1)
# c) split every filtered string at do()
# d) merge it back to a single string using only the slice of [1:] (because
#    whatever was before is a don't())
# e) run the same construction as before
#
# for this to work we need to add an explicit do() into the very beginning of
# the input string

dont_unless_do_add_multiplication = sum(
  list(
    map(
      lambda int_pairs: int(int_pairs[0])*int(int_pairs[1]),
      list(
        filter(
          lambda possible_int: \
            possible_int[0].isdigit() and possible_int[1].isdigit(),
            [
              [
                token[0][1:],
                token[1][:token[1].find(")")]
              ] \
                for token in [
                  token.split(",") \
                    for token in list(
                      filter(
                        lambda possible_token: \
                          possible_token.startswith("(") and \
                          possible_token.find(",") > 1 and \
                          possible_token.find(")") > 3 and \
                          possible_token.find(",") < possible_token.find(")")-1,
                          "".join(
                            [
                              "".join(dosplit[1:]) \
                                for dosplit in [
                                  considerdoing.split("do()") \
                                    for considerdoing in list(
                                      filter(
                                        lambda instruction: \
                                          instruction.find("do")>-1,
                                          [
                                            dontsplit \
                                              for dontsplit in \
                                                f"do(){lines}".split(
                                                  "don't()"
                                                )
                                          ]
                                      )
                                    )
                                ]
                            ]
                          ).split("mul")
                      )
                    )
                ]
            ]
        )
      )
    )
  )
)

## ... regex would have been nicer, but this works and shows it can be done as a
## "oneliner" without any imports :D

```

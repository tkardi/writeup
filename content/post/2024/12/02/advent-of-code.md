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

# <a id="index" href="#index">Index</a>
<a href="#day1">#1</a> | <a href="#day2">#2</a> | <a href="#day3">#3</a> |
<a href="#day4">#4</a> | <a href="#day5">#5</a>


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
<a href="#index">^ Index</a>

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
<a href="#index">^ Index</a>

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
<a href="#index">^ Index</a>

## <a id="day4" href="#day4">Day 4</a>
[Challenge](https://adventofcode.com/2024/day/4)

```python
with open('./advent-of-code/input-day-4.txt') as f:
    data = f.read()

# this going to be a lot of looping, so I'm kind of thinking in the line of
# moving windows. But instead of doing proper moving window of 4*4 letters
# (XMAS*XMAS) because it's easier to loop once and extract values per rows
# rather than try to extract the exact values within windows.
# A couple of ideas:
# - the number of windows we need is (matrix_width-3)*(matrix_height).
#   width-3 because XMAS is 4 char long, and height because we want to
#   search all rows
# - search only by rows and use a transposed matrix instead of searching by
#   columns
# - search for XMAS and SAMX, no need to reverse text in window
# - diagonals don't need transposed search
#

# the full matrix as list of lists (rows) of chars (columns)
lines = [
  [*row] \
    for row in data.split("\n") \
      if row > ""
]

# transpose chars matrix so
#[
#  ['A', 'B'],
#  ['C', 'D']
#]
# .. becomes
#[
#  ['A', 'C'],
#  ['B', 'D']
#]
transposed_lines = [
  [row[i] for row in lines] \
    for i in range(len(lines[0]))
]

# find count of XMAS
xmas_samx = len(
  list(
    filter(
      lambda cand: cand == 'XMAS' or cand == 'SAMX',
        [
          # flatten the inner lists, we don't need to know that
          cand for cands in \
            # search rows
            [
              [
                "".join(lines[row][column:column+4]) \
                  for column in range(len(lines[0])-3)
              ] \
                for row in range(len(lines))
            ]+\
            # search transposed rows
            [
              [
                "".join(transposed_lines[row][column:column+4]) \
                  for column in range(len(lines[0])-3)
              ] \
                for row in range(len(lines))
            ]+\
            # search desc diagonals
            [
              [
                "".join([lines[row+i][column+i] for i in range(4)]) \
                  for column in range(len(lines[0])-3)
              ] \
                for row in range(len(lines)-3)
            ]+\
            # search asc diagonals
            [
              [
                "".join([lines[row+3-i][column+i] for i in range(4)]) \
                  for column in range(len(lines[0])-3)
              ] \
                for row in range(len(lines)-3)
            ] \
              for cand in cands
        ]
    )
  )
)

# for the second challenge it looks like the window approach was good, let's
# adapt it. All MAS diagonals we are looking for will be always in the same
# window so we'll look for diagonals in 3*3 window and then check the
# indexes where both asc and desc lists have MAS or SAM in the same
# row and column index value

mas_sam = list(
  map(
    lambda diagonal: \
      [
        diagonal[0][row][column] in ["MAS","SAM"] and \
        diagonal[1][row][column] in ["MAS","SAM"] \
          for column in range(len(diagonal[0][1])) \
            for row in range(len(diagonal[0]))
      ],
      [[
        [
          [
            "".join([lines[row+i][column+i] for i in range(3)]) \
              for column in range(len(lines[0])-2)
          ] for row in range(len(lines)-2)
        ],
        [
          [
            "".join([lines[row+2-i][column+i] for i in range(3)]) \
              for column in range(len(lines[0])-2)
          ] for row in range(len(lines)-2)
        ]
      ]]
    )
  )[0].count(True)

```
<a href="#index">^ Index</a>

## <a id="day5" href="#day5">Day 5</a>
[Challenge](https://adventofcode.com/2024/day/5)

```python
with open('./advent-of-code/input-day-5.txt') as f:
    data = f.read()

rules, rows = [
  p.strip("\n").split("\n") \
    for p in data.split("\n\n")
]

rules_idx = dict(
  [
    [
      int(key),
      [
        int(v.split("|")[1]) \
          for v in \
            list(
              filter(
                lambda a: \
                  a.split("|")[0] == key,
                  rules
              )
            )
      ]
    ] \
      for key in \
        set(
          [
            s[0] for s in [
              rule.split("|") for rule in rules
            ]
          ]
        )
  ]
)

# sum of correct order mid-page-numbers
sum(
  list(
    map(
      lambda pages: \
        pages[int(len(pages)/2)],
        filter(
          lambda ps: [
            [
              rules_idx.get(p) == None or \
                [
                  c not in rules_idx[p] \
                    for c in ps[0:ps.index(p)]
                ].count(False)==0
            ][0] \
              for p in ps
          ].count(False) == 0,
          map(
            lambda pages: \
              [
                int(page) \
                  for page in pages
              ],
              [
                pages.split(",") \
                  for pages in rows
              ]
          )
        )
    )
  )
)

## second question
# NOK ordered pages:
# - find them by reversing the previously used filter

nok_pages_lists = list(
  filter(
    lambda ps: [
      [
        rules_idx.get(p) == None or \
          [
            c not in rules_idx[p] \
              for c in ps[0:ps.index(p)]
          ].count(False)==0
      ][0] \
        for p in ps
    ].count(False) > 0,
    map(
      lambda pages: \
        [
          int(page) \
            for page in pages
        ],
        [
          pages.split(",") \
            for pages in rows
        ]
    )
  )
)

# - ... and establish "correct order" (although we need only the middle-most page
#   number) by counting (per every page numbers list) how many times every
#   number appears in the "after-pages". The idea being that we can estimate
#   the order without really checking/fixing the order: if the list is
#   N numbers long and a number appers N-1 timees in the "afterlist", then it is
#   expected to have N-1 numbers in front for the current combination of
#   pagenumbers. Similarly, if it does not not appear in "after-pages", then it
#   must be the the first.
# This is not actually a correct way of establishing it, but just works
# here :) - e.g. if there's a number in the NOK pages list that does not appear
# neither as a key or as after-pages list in the rules_idx dict then it's
# location should not change.

sum_of_nok_mids = sum(
  list(
    map(
      lambda nok_pages, ord: \
        sorted(
          nok_pages,
          key=lambda x: \
            ord[nok_pages.index(x)]
        )[int((len(nok_pages)-1)/2)],
        nok_pages_lists,
        list(
          map(
            lambda nok_pages: [
              [
                page in v \
                  for v in [
                    rules_idx[p] for p in nok_pages
                  ]
              ].count(True) \
                for page in nok_pages
            ],
            nok_pages_lists
          )
        )
    )
  )
)

```
<a href="#index">^ Index</a>

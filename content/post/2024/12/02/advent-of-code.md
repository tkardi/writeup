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
<a href="#day1">#1</a> |

<a href="#day2">#2</a> | <a href="#day3">#3</a> | <a href="#day4">#4</a> |
<a href="#day5">#5</a> | <a href="#day6">#6</a> | <a href="#day7">#7</a>

<a href="#day8">#8</a>  


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

## <a id="day6" href="#day6">Day 6</a>
[Challenge](https://adventofcode.com/2024/day/6)

```python
with open('./advent-of-code/input-day-6.txt') as f:
    data = f.read()

# process map data to a list of lists of "network nodes"
guard_map = [
  [*row] \
    for row in data.split("\n") \
      if row > ""
]

# build "routing matrix" from "network nodes"
matrix = {
  (x, y): column \
    for y, row in enumerate(guard_map) \
      for x, column in enumerate(row)
}

# initialize trail by dinding guard location
path = [(k,v) for k, v in matrix.items() if v in ["^", ">", "v", "<"]]

# walk the path according to rules
_ = [
  path.insert(
    len(path),
    tuple(
      map(
        lambda mv: \
          # go up
          ( (mv[0][0], mv[0][1]-1), "^") if mv[1]=="^" and \
            matrix.get((mv[0][0], mv[0][1]-1), "X") != "#" else (
          # go right
          ( (mv[0][0]+1, mv[0][1]), ">") if mv[1]==">" and \
            matrix.get((mv[0][0]+1, mv[0][1]), "X") != "#" else (
          # go left
          ( (mv[0][0]-1, mv[0][1]), "<") if mv[1]=="<" and \
            matrix.get((mv[0][0]-1, mv[0][1]), "X") != "#" else (
          # go down
          ( (mv[0][0], mv[0][1]+1), "v") if mv[1]=="v" and \
            matrix.get((mv[0][0], mv[0][1]+1), "X") != "#" else (
          # turn right instead of moving up
          ( (mv[0][0], mv[0][1]), ">") if mv[1]=="^" and \
            matrix.get((mv[0][0], mv[0][1]-1), "X") == "#" else (
          # turn down instead of moving right
          ( (mv[0][0], mv[0][1]), "v") if mv[1]==">" and \
            matrix.get((mv[0][0]+1, mv[0][1]), "X") == "#" else (
          # turn left instead of moving down
          ( (mv[0][0], mv[0][1]), "<") if mv[1]=="v" and \
            matrix.get((mv[0][0], mv[0][1]+1), "X") == "#" else (
          # turn up instead of moving left
          ( (mv[0][0], mv[0][1]), "^") if mv[1]=="<" and \
            matrix.get((mv[0][0]-1, mv[0][1]), "X") == "#" else (
          # smth wrong...
          (-1,-1), "X")
                      )
                    )
                  )
                )
              )
            )
          ),
          [move]
      )
    )[0]
  ) \
    for move in path \
      if move[1] != "X" and \
        move[0][0] >= 0 and \
        move[0][1] >= 0 and \
        move[0][0] <= len(guard_map[0])-1 and \
        move[0][1] <= len(guard_map)-1
]

# number of distinct locations the guard visits

count_distinct_locations = len(
  set(
    [
      move[0] \
        for move in path[:-1]
    ]
  )
)

## and obstruction can be placed anywhere on the path
## except out of guard_map and the first loc where the
## guard is
obs = list(set([loc[0] for loc in path[:-1] if loc[0]!=path[0][0]]))

## initialize the possible paths the guard we
## will be walking
loop_paths = [path[0] for i in range(len(obs))]

## walk paths
_ = [
  [
    loop_path.insert(
      len(loop_path),
      tuple(
        map(
          lambda mv, rs: \
            # check loop, i.e move in this direction
            # is already in path. mark it as X for quits
            ( (mv[0][0], mv[0][1]), "X") if mv in loop_path[:-1] else (
            # go up
            ( (mv[0][0], mv[0][1]-1), "^") if mv[1]=="^" and \
              (mv[0][0], mv[0][1]-1) != rs and \
              matrix.get((mv[0][0], mv[0][1]-1), "X") != "#" else (
            # go right
            ( (mv[0][0]+1, mv[0][1]), ">") if mv[1]==">" and \
              (mv[0][0]+1, mv[0][1]) != rs and \
              matrix.get((mv[0][0]+1, mv[0][1]), "X") != "#" else (
            # go left
            ( (mv[0][0]-1, mv[0][1]), "<") if mv[1]=="<" and \
              (mv[0][0]-1, mv[0][1]) != rs and \
              matrix.get((mv[0][0]-1, mv[0][1]), "X") != "#" else (
            # go down
            ( (mv[0][0], mv[0][1]+1), "v") if mv[1]=="v" and \
              (mv[0][0], mv[0][1]+1) != rs and \
              matrix.get((mv[0][0], mv[0][1]+1), "X") != "#" else (
            # turn right instead of moving up
            ( (mv[0][0], mv[0][1]), ">") if mv[1]=="^" and \
              (matrix.get((mv[0][0], mv[0][1]-1), "X") == "#" or \
                (mv[0][0], mv[0][1]-1) == rs) else (
            # turn down instead of moving right
            ( (mv[0][0], mv[0][1]), "v") if mv[1]==">" and \
              (matrix.get((mv[0][0]+1, mv[0][1]), "X") == "#" or \
                (mv[0][0]+1, mv[0][1]) == rs) else (
            # turn left instead of moving down
            ( (mv[0][0], mv[0][1]), "<") if mv[1]=="v" and \
              (matrix.get((mv[0][0], mv[0][1]+1), "X") == "#" or \
                (mv[0][0], mv[0][1]+1) == rs) else (
            # turn up instead of moving left
            ( (mv[0][0], mv[0][1]), "^") if mv[1]=="<" and \
              (matrix.get((mv[0][0]-1, mv[0][1]), "X") == "#" or \
                (mv[0][0]-1, mv[0][1]) == rs) else (
            # smth wrong
            (-1,-1), "Y")
                            )
                          )
                        )
                      )
                    )
                  )
                )
              ),
            [move],
            [obs[i]]
        )
      )[0]
    ) \
      for move in loop_path if \
        move[1] not in [ "X", "Y"] and \
        move[0][0] >= 0 and \
        move[0][1] >= 0 and \
        move[0][0] <= len(guard_map[0])-1 and \
        move[0][1] <= len(guard_map)-1
  ] \
    for (i,loop_path) in enumerate(loop_paths)
]

## count number of paths finishing on a loop
possible_loopers = len(
  [
    obs[i] \
      for i, lp in enumerate(loop_paths) \
        if lp[-1][1] == 'X'
  ]
)

```
<a href="#index">^ Index</a>

## <a id="day7" href="#day7">Day 7</a>
[Challenge](https://adventofcode.com/2024/day/7)

```python
with open('./advent-of-code/input-day-7.txt') as f:
    data = f.read()

# prepare equations with all possible combinations of additions and
# multiplications. That will equal 2^number-of-spaces equations. create a
# range and cast to 0-padded binary so that we'll take 0 as addition and
# 1 as multiplication, e.g.:
#
# eq = "10 20 30"
# >>> [
# ...    f"{bin(i)[2:]}".rjust(eq.count(" "), "0") \
# ...        for i in range(pow(2,eq.count(" ")))
# ... ]
# ['00', '01', '10', '11']
# >>>
#
# the prepared statements itself is a dict with (result, equation-numbers) tuple
# as the key and a dict of equation mask and solution list key-value pairs
# as a value, e.g.:
#
# >>> {
# ...  (230, 10, 20, 30): {
# ...    "00": [10],
# ...    "01": [10],
# ...    "10": [10],
# ...    "11": [10]  
# ...  }
# ... }
# >>>
#

eqs = {
  tuple(
    [int(a)]+[int(f) for f in b.split(" ")]
  ):{
    f"{bin(i)[2:]}".rjust(b.count(" "), "0"):[int(b.split(" ")[0])] \
      for i in range(pow(2, b.count(" ")))
  } for a, b in [
    eq.split(": ") \
      for eq in data.split("\n") if eq.strip() > ""
  ]
}

# Execute every equation combination just like reduce but without importing
# reduce from collections (some other cases for it used in previous days too)

_ = [
  [
    [
      solve.insert(
        len(solve),
        list(
          map(
            lambda equation: \
              solve[-1] + numbers[i+2] \
                if equation[1] == "0" else \
                  solve[-1] * numbers[i+2],
              [(i, maskbit)]
          )
        )[0]
      ) for i, maskbit in enumerate([*eq])
    ] for eq, solve in solutions.items()
  ] for numbers, solutions in eqs.items()
]

# sum all results from equations that have at least one correct result

calibration_result = sum(
  set(
    [
      k[0] for k,v in eqs.items() \
        for (eq, sol) in v.items() \
          if k[0] == sol[-1]
    ]
  )
)

# for the second part let's switch from binary to ternary (base 3), so 0 for
# addition, 1 for multiplication, and 2 for concatenation. There might be other
# ways built-in to python (that i don't know of and don't know how to search
# for as this is the first time in my life that i come into contact with
# base 3 numbers at all i think), but staying true to no-imports and no-defs
# I'll calculate these on-spot. Getting a recursion to work in
# list-comprehension is painful, but possible... :D
#
# So, e.g. for two spaces before with two operations we needed 2^2 possible
# equation-masks. For two space with three possible operations we'll need
# 3^2 possible operations (and 3^3 for three spaces, and 3^4 for 4 spaces).
# All the "equation-masks" will need to be calculated beforehand, and stored
# as a variable.
#
# init the ternary calculation. I'll calculate the values only up until the
# number that is needed (i.e. only those eqs that did not find a solution
# before)

solved = set(
  [
    k \
      for k,v in eqs.items() \
        for (eq, sol) in v.items() \
          if k[0] == sol[-1]
  ]
)

ternaries = {
  i:{"s":"", "n":i} \
    for i in range(
      pow(
        3,
        max(
          [
            b.count(" ") for a, b in [
              eq.split(": ") for eq in data.split("\n") if eq.strip() > ""
            ] if tuple([int(a)]+[int(f) for f in b.split(" ")]) not in solved
          ]
        ) + 1
      )
    )
}

# use the created ternary dict and loop in list comprehension calculating
# the base 3 value for every dict key. Why 20? because we should cap somewhere
# and most proably there is a better way to control this, but currently it's
# good enough. Anyway. Calculate base 3 values

_ = [
  [
    v.update(
      {
        "s":str(v["n"]%3)+v["s"],
        "n":v["n"]//3
      }
    ) for i in range(20) \
        if v["n"]!=0
  ] for k, v in ternaries.items()
]

# so we end up with:
#
# >>> _ = [
# ...     print(
# ...         f"decimal {i} == {ternaries[i]['s'].rjust(1,'0')} in base3"
# ...     ) for i in range(10)
# ... ]
# decimal 0 == 0 in base3
# decimal 1 == 1 in base3
# decimal 2 == 2 in base3
# decimal 3 == 10 in base3
# decimal 4 == 11 in base3
# decimal 5 == 12 in base3
# decimal 6 == 20 in base3
# decimal 7 == 21 in base3
# decimal 8 == 22 in base3
# decimal 9 == 100 in base3
# >>>

# recreate the eqs from input, but discard the already solved ones (no
# need to recalculate these)

eqs = {
  tuple(
    [int(a)]+[int(f) for f in b.split(" ")]
  ):{
    ternaries[i]["s"].rjust(b.count(" "), "0"):[int(b.split(" ")[0])] for i in range(pow(3,b.count(" ")))
  } for a, b in [
    eq.split(": ") for eq in data.split("\n") if eq.strip() > ""
  ] if tuple([int(a)]+[int(f) for f in b.split(" ")]) not in solved
}

# update the previous lambda to include the concat operation too and run

_ = [
  [
    [
      solve.insert(
        len(solve),
        list(
          map(
            lambda equation: \
              solve[-1] + numbers[i+2] if equation[1] == "0" else (
               solve[-1] * numbers[i+2] if equation[1] == "1" else (
                 int(str(solve[-1])+str(numbers[i+2]))
               )
              ),
              [(i,el)]
          )
        )[0]
      ) for i,el in enumerate([*eq])
    ] for eq, solve in solutions.items()
  ] for numbers, solutions in eqs.items()
]

# .. and for the final result
calibration_result += sum(
  set(
    [
      k[0] \
        for k,v in eqs.items() \
          for (eq, sol) in v.items() \
            if k[0] == sol[-1]
    ]
  )
)


```
<a href="#index">^ Index</a>

## <a id="day8" href="#day8">Day 8</a>
[Challenge](https://adventofcode.com/2024/day/8)

```python
with open('./advent-of-code/input-day-8.txt') as f:
    data = f.read()

# parse to a list of lists
antenna_map = [
  [*row] \
    for row in data.split("\n") \
      if row > ""
]

# store for easy access based on coordinates. this is a bit
# irrelevant, but makes
# for easy access for creating the antenna locs dict in
# the next step + later
# easy access to get "what's there" (call it "reverse geocoding")
matrix = {
  (x, y): column \
    for y, row in enumerate(antenna_map) \
      for x, column in enumerate(row)
}

# distinct antennas to their locations...
antenna_locs = {
  v: [
    loc for loc, x in matrix.items() \
      if x == v
  ] \
    for _, v in matrix.items() \
      if v in set(
        [
          mark \
            for row in antenna_map \
              for mark in row \
                if mark != '.'
        ]
      )
}

# pair all distinct antennas and calculated their antinode locations so that
# every antenna-pair is listed only once and antinodes are added as
# first and last locations to the antenna pair coordinates list
# The followig is effectively looping only the "the upper half from the main
# diagonal" of the "distance matrix" (only the cells marked with "o" for the
# following):
#
#   | A | B | C | D |
# --|---|---|---|---|
# A | x | o | o | o |
# B | x | x | o | o |
# C | x | x | x | o |
# D | x | x | x | x |

ants = {
  k: [
    [
      [
        # first antinode
        (
          v[i][0]+(v[i][0]-v[j][0]),
          v[i][1]+(v[i][1]-v[j][1])
        ),
        # first of the antenna pair
        v[i],
        # second of the antenna pair
        v[j],
        # second antinode
        (
          v[j][0]+(v[j][0]-v[i][0]),
          v[j][1]+(v[j][1]-v[i][1])
        )
      ] for j in range(i, len(v)) \
        if i < j and \
          # neither of antinodes should be at the same
          # frequency antenna location
          matrix.get(
            (
              v[i][0]+(v[i][0]-v[j][0]),
              v[i][1]+(v[i][1]-v[j][1]))
            ) != k and \
          matrix.get(
            (
              v[j][0]+(v[j][0]-v[i][0]),
              v[j][1]+(v[j][1]-v[i][1]))
            ) != k
    ] for i, locs in enumerate(v[:-1])
  ] for k, v in antenna_locs.items()
}

distinct_antinodes_count = len(
  set(
    [
     # get locations within map bounds
     # and flatten lists over frequencies
      n \
        for n in [
          # get locations and flatten lists
          # per frequency pair-rows
          [v[0],v[-1]] \
            for v in [
              g for ant, g in ants.items() for g in g
            ] \
              for v in v
        ] \
          for n in n \
            # check antinode location
            # against map bounding box
            if n[0]>=0 and \
              n[1]>=0 and \
              n[0]<len(antenna_map[0]) and \
              n[1]<len(antenna_map)
    ]
  )
)

```

<a href="#index">^ Index</a>

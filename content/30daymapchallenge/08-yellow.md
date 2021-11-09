---
title: "Day 08: yellow"
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
---
### Data
- [GTFS data from the Estonian Road Administration](https://www.mnt.ee/eng/public-transportation/public-transport-information-system)

### Tools
PostGIS, QGIS.

### Summary
A kind of _~~writer's~~ mapmaker's block_.

Estonian train connections are not very elaborate - in a sense that looking
at the tracks and timetables the main idea behind it all is to allow people
to travel to Tallinn and back. So after many tries late in the evening I
come up with the idea of taking GTFS data and making a spirally looking
graph of a train timetable - taking the morning train from Narva where
and when will you be able to end up. So separate _forks_ show the places
where tracks diverge and time runs in clockwise manner (with the azimuth
from the plain showing the time of arrival). Did not clearly spend/have enough
time to think or work this through as one of the _tracks_ (finishing at
16:25 in Kloogaranna) should in fact do another full circle and not simply
_cross out_ over the Viljandi 13:50 track. And in the end rotated the graph
to better fit into the image and therefore losing the time reference.

One time in the future if given enough time I would like to rework this into
the kind of visualization that I wanted it to be in my mind's eye.

There are some SQL snippets left behind from creating this but it seems to be
in such a horrible mess that I'd prefer not to publish them... at least not yet
:)

[hi-res](https://tkardi.ee/writeup/img/30daymapchallenge/day-8-yellow.png)

{{< tweet 1325885920789270531 >}}

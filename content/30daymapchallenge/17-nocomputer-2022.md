---
title: "2022 / Day 17: A map without a computer"
date: 2022-11-17T12:44:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: revisiting my scrapbook drawings for calculating 'road areas' from cadastral data."
---
I have a scrapbook for plotting out ideas of SQL processing flows and for
todays entry I decided to revisit some of the old ones. Found a bunch of
sketch-ups for a little something I did about 1.5 years ago. And I must
apologize - as it has been already some time then most probably I won't remember
all the fine print details.

But. The task was to find a way to create _road areas_ (polygons around road
centerlines) from cadastral point descriptions. Yes, sure the points had a
specific location, but the main goal was to use a description like
"5.5 meters from the centerline"  to shift the points around so that
**if the geometry of the road centerline** changes, the road surface area
would be _recalculatable_ on the fly.

As a side-note: the structure of this description with a point itself was not
a concern. At least in that moment.

But I remember distinctly thinking that this is essentially a question of a
variable width buffer for the road centerline geometry. Which, given that we
know the desired width at a linestrings node (
which can be stored e.g either as `m-coordinate` and retrieved via
[st_m](https://postgis.net/docs/ST_M.html) or `z-coordinate` and retrieved via
[st_z](https://postgis.net/docs/ST_Z.html)) can be calculated as a
[st_union](https://postgis.net/docs/ST_Union.html) over the linestrings
identifier of all [st_convexhull](https://postgis.net/docs/ST_ConvexHull.html)
polygons created from [st_dumppoints](https://postgis.net/docs/ST_DumpPoints.html)
points of a segmentized linestrings
[st_firstpoint](https://postgis.net/docs/ST_FirstPoint.html)s and
[st_lastpoint](https://postgis.net/docs/ST_LastPoint.html)s
[st_buffer](https://postgis.net/docs/ST_Buffer.html)s to the required width. Or
in a bitsy more simplified form

{{< highlight sql >}}
select
    oid, st_union(geom) as geom
from (
    select
        oid, ord,
        lag(ord) over (partition by oid order by ord) as lag_oid,
        st_convexhull(
            st_collect(
                array[
                    geom,
                    lag(geom) over (partition by oid order by ord)
                ]
            )
        ) as geom
    from (
        select
            oid,
            pts.path[1] as ord,
            st_buffer(pts.geom, st_z(pts.geom)) as geom
        from (
            select
                1 as oid,
                st_makeline(
                    array[
                        st_pointz(0,0,5),
                        st_pointz(10,10,1),
                        st_pointz(20,20,5),
                        st_pointz(20,10,3.5)
                    ]
                ) geom
            ) d
                join lateral
                    st_dumppoints(d.geom) pts on true
        ) e
) f
where
    lag_oid is not null
group by
    oid
;
{{</ highlight >}}

Which returns something in the line of

![A variable width buffer of a linestring using PostGIS](../img/d17-2022/variable-width-buffer.png)

But there are a few issues to be solved here first (as I see from the notes):

- we don't know distances at the nodes of the road centerline but rather at
some _whatever locations_ beside the centerline.
- we'll need to deal with left and right hand side of the road centerline vector
independently as the buffer width might vary (e.g 5m on the L, but 15m on the R).

In the end the solution I came up with is essentially to snap points to the
road centerline [st_closestpoint](https://postgis.net/docs/ST_ClosestPoint.html) or
[st_linelocatepoint](https://postgis.net/docs/ST_LineLocatePoint.html) - I can't
remember any more which one I used. Create the required buffers as with the
variable width buffer, [st_split](https://postgis.net/docs/ST_Split.html) the
buffer with the road centerline, figure out if I need to keep the left- or
right-hand side one, and then union these over the road centerline together.

Some additional notes from here are the questions:

- what to do with the first and last points of the centerline. The first
idea of _interpolation_ (from previous or next nodes of the line)
seems to be crossed out as that would lead to disproportionate buffer sizes
(imagine a steadily increasing array of 1m, 5m, 10m, or decreasing for
that matter) - so instead use the value at the _other_ end of that
specific segment.
- but more pressingly: how would I know which points are connected to which
road centerline. And for the life of me I can't remember any more how this
was supposed to be tackled.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-17-nocomputer.jpg)

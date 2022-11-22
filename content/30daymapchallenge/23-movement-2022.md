---
title: "2022 / Day 23: Movement"
date: 2022-11-22T18:25:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Estimating public transit locations from timetable data."
---
Back some years ago I did [a thing](https://github.com/tkardi/eoy/tree/dockerize)
for estimating public transit vehicle
locations in _real-time_ using [GTFS](https://developers.google.com/transit/gtfs/)
data. Including a presentation during
the [FOSS4G Europe 2017](/writeup/talk/foss4ge-2017/) conference in Paris.

Now looking back at it (being 5 years older, and hopefully wiser) the way I
applied speeding up/slowing down feels a bit too complicated. Essentially
the same can be achieved with [sigmoid function](https://en.wikipedia.org/wiki/Sigmoid_function)
giving the path much smoother flow aswell. There's still something with speeds
not coming down when reaching the stop but don't have time to wrap my head
around it currently. Have to leave something for next time too.

The CTE part of the query is only preparation of the data this time:

- `trip` builds the route geometry,
- `timetable` sets the desired times (values can be added or modified) in seconds
since the start of the trip that the vehicle will reach the station. This
SHOULD be at least 2 timestamps long although it's not checked afterwards in
the query :) Just be gentle.
- `legs` is the `trip` divided into equal length segments acting as the
path between stations. These are calculated depending on the number of `stops`
in the `timetable`

In the main query part we do `union all`s to fetch results for

- `with_speedups` using the sigmoid fitting: with regard to the `leg` start and
endtimes (on the `x-axis` of the sigmoid plot) calculate the fraction of the
time we are currently at and find the corresponding value on the `y-axis` and
use that as the **fraction of distance we have covered**. Which gives us the
notion that a vehicle stops at the station - picks up speed - goes full speed -
slows down - stops at the station. And over again for every `leg`.
- `with_stops` is using a linear interpolation on a `leg` - essentially: if
the `leg` starts at 60 seconds and finishes as 120 seconds and currently is
90 seconds since trip start then we're half way through the `leg`. Meaning the
vehicle is always on time at the station but never stops there.
- and `no_stops` interpolates very linearly over the whole trip taking into
account only the time of leaving from the first satation and time of arrival in
the last station.
- and last but not least is `stops` which is the stops layer. As I'm using
[QGIS Temporal Controller](https://www.qgistutorials.com/en/docs/3/animating_time_series.html)
to animate the vehicles and wanted this to be a single
query then this layer will be _temporal_ aswell, meaning every second in the
dataset has its own personal stops :)

All _point locating_ tasks are performed with
[st_lineinterpolatepoint](https://postgis.net/docs/ST_LineInterpolatePoint.html).
For creating trip legs
[st_linesubstring](https://postgis.net/docs/ST_LineSubstring.html) does its
thing.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-23-movement.gif)

{{< highlight sql >}}
with
    trip as (
        /* TRIP GEOMETRY is a figure of 8 pushed to the side*/
        select
            st_rotate(
                st_chaikinsmoothing(
                    st_makeline(
                        array[
                            st_point(30,60,0),
                            st_point(50,100,0),
                            st_point(0,100,0),
                            st_point(50,0,0),
                            st_point(0,0,0),
                            st_point(30,60,0)
                        ]
                    ),
                    3, true
                ),
                radians(90)
            ) as geom
    ),
    timetable as (
        /* TIMETABLE is an array of seconds since start of the trip
           that the vehicle will reach a station. STATIONS themselves
           are derived by dividing the trip shape into equal length parts
           depending on how many stations are needed*/
        select
            row_number() over(order by ts) as stop_nr, ts,
            min(ts) over() as min_ts,
            max(ts) over() as max_ts,
            lead(ts) over(order by ts) as next_ts,
            lag(ts) over(order by ts) as prev_ts
        from
            unnest(
                array[
                    0,
                    75,
                    120,
                    170,
                    250,
                    300
                ]
            ) with ordinality tt(ts, i)
    ),
    legs as (
        /* a LEG starts and finishes at a station*/
        select
            ord,
            st_linesubstring(
                trip.geom,
                (ord-1)::numeric/(tt.count-1)::numeric,
                (ord)::numeric/(tt.count-1)::numeric
            ) as geom
        from
            trip
                join lateral (
                    select count(1)
                    from timetable
                ) tt on true
                join lateral
                    generate_series(
                        1, tt.count-1, 1
                    ) ord on true
    )
/* pull it all together with*/
select
    row_number() over ()::int as oid,
    ('2022.11.23 '||(((7.98 + t::numeric / 3600.0)||' hours')::interval)::time)::timestamp,
    *
from (
    /* INTERPOLATE POINT WITH SPPEDUPS AND SLOWDOWNS*/
    select
        trip.from_ts + t as t,
        loc as geom,
        case
            when
                t=0 then
                    ((fr.val * (st_length(trip.geom))) / 0.01::numeric) + 5.0
            else
                (fr.val * (st_length(trip.geom)) / (t::numeric)) + 5.0
        end as speed,
        'with_speedups' as cl
    from (
        select
            legs.ord, legs.geom,
            timetable.ts as from_ts, timetable.next_ts as to_ts
        from
            legs,
            timetable
        where
            legs.ord = timetable.stop_nr
    ) trip
        join lateral
            generate_series(0, trip.to_ts-trip.from_ts) t on true
        join lateral (
            select
                case
                    when
                        /* 5 seconds since start - still at stop*/
                        t <= 5 then 0.0
                    when
                        /* five seconds to the end - we're arrived*/
                        t >= trip.to_ts - 5 then 1.0
                    else (
                        /*  Fit by sigmoid*/
                        1.0 / (
                            1.0 + pow(exp(1),-1*(
                                (
                                    (t::numeric - 5.0) /
                                    (trip.to_ts-trip.from_ts - 5.0)::numeric
                                ) * 20.0 - 10.0
                            ))
                        )
                    )
                end as val
        ) fr on true
        join lateral
            st_lineinterpolatepoint(
                trip.geom,
                fr.val
            ) loc on true
    union all
    /* INTERPOLATE POINT BETWEEN THIS STATION AND NEXT STATION*/
    select
        trip.from_ts + t as t,
        loc as geom,
        case
            when
                t=0 or t=(trip.to_ts-trip.from_ts) then
                    ((0 * (st_length(trip.geom))) / 0.01::numeric) + 5.0
            else
                ((fr.val * (st_length(trip.geom))) / t::numeric) + 5.0
        end as speed,
        'with_stops' as cl
    from (
        select
            legs.ord, legs.geom,
            timetable.ts as from_ts, timetable.next_ts as to_ts
        from
            legs,
            timetable
        where
            legs.ord = timetable.stop_nr
    ) trip
        join lateral
            generate_series(0, trip.to_ts-trip.from_ts) t on true
        join lateral (
            select t::numeric / (trip.to_ts-trip.from_ts)::numeric  as val
        ) fr on true
        join lateral
            st_lineinterpolatepoint(
                trip.geom,
                fr.val
            ) loc on true
    union all
    /* INTERPOLATE ONLY BETWEEN FIRST AND LAST STATION*/
    select
        t,
        loc as geom,
        case
            when
                t = tt.min or t=tt.max
                    then ((0 * (st_length(trip.geom))) / 0.01::numeric) + 5.0
            else
                ((fr.val * (st_length(trip.geom))) / t::numeric)  + 5.0
        end as speed,
        'no_stops' as cl
    from
        trip
            join lateral (
                select min(ts), max(ts) from timetable
            ) tt on true
        join lateral
            generate_series(tt.min, tt.max) t on true
        join lateral (
            select t::numeric / (tt.max-tt.min)::numeric as val
        ) fr on true
        join lateral
            st_lineinterpolatepoint(
                trip.geom,
                fr.val
            ) loc on true
    union all
    /* STATIONS*/
    select
        t, st_startpoint(legs.geom), 4 as speed, 'stop' as cl
    from
        legs
            join lateral (
                select min(ts), max(ts) from timetable
            ) tt on true
            join lateral
                generate_series(tt.min, tt.max) t on true
) d
;
{{</ highlight >}}

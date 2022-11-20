---
title: "2022 / Day 20: Favourite"
date: 2022-11-19T18:09:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Through the 24h of flat earth ice wall rotating in Web Mercator."
---
This is to celebrate one of my favourite/funniest "maps" I did in the
beginning of my favourite season this year. All of this now at the time when my
least favourite season is starting, and snow starts piling up in front of the
door. And it's cold... :(

Back in the end of April this year I saw
[@RealIvanSanchez](https://twitter.com/RealIvanSanchez) tweet about xkcds
Madagascator.

{{< tweet 1520105735073456128 >}}

... and I decided to chip in with my own version using pure SQL

![Madagascator!](../img/d20-2022/madagascator.jpeg)

The SQL to recreate this can be found as a gist
[here](https://gist.github.com/tkardi/c295bbdcc2c7e21edb365fd3852894e9#file-gistfile1-txt).
Then a couple of days later ended up modifying it a bit further to do a
_Flatearthator_ - a flat earth map in Web Mercator
([SQL here](https://gist.github.com/tkardi/c295bbdcc2c7e21edb365fd3852894e9#file-flatearth-tc)).

![Flatearthator](../img/d20-2022/flatearthator.jpeg)

Now it is the time of the year that reminds the ice wall (first snow of the
winter yesterday :/ ) and thought about animating through the 24 h of flat earth
spinning. Although I don't know if it supposed to spin or not - but as it's
supposed to cruise "upward" in space at a constant rate of 9.81 m/s<sup>2</sup>
then i guess it can spin around its axis as well.

The animation is done using
[QGIS temporal controller](https://www.qgistutorials.com/en/docs/3/animating_time_series.html).
Data features the geometry of Antarctica from [NaturalEarth data small scale
cultural vectors](https://www.naturalearthdata.com/downloads/110m-cultural-vectors/)
([countries Admin 0](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/110m/cultural/ne_110m_admin_0_countries.zip)).
The data payload is converted over
[`TinyWKB`](https://github.com/TWKB/Specification/blob/master/twkb.md) format
encoded as a `base64` string using
[st_geomfromtwkb](https://postgis.net/docs/ST_GeomFromTWKB.html).

The solution uses [st_buffer](https://postgis.net/docs/ST_Buffer.html)
to project a point-of-origin (set at Null Island) to a distance measured between
the North pole and a specific node (using the `geography` type to get the real
distance in meters) with
[st_distance](https://postgis.net/docs/ST_Distance.html) and
divided by `pi()`. Then takes the
[st_exteriorring](https://postgis.net/docs/ST_ExterioRing.html)
of the buffer, rotates it around with
[st_rotate](https://postgis.net/docs/ST_Rotate.html)
every full degree between

```
    generate_series(1,360,1)
```

(which will later on serve as the timestamp), scales with
[st_scale](https://postgis.net/docs/ST_Scale.html) the the buffer exterior line
a bit outward for the Mercator pole-effect to be visible.

[**Note!** this should not really be necessary and should instead be done during
buffering: `distance/pi()` should be set to a correct value there in order to
use as much space that we have  available for Mercator in the N/S regions].

And finally locates
the location of the _projected_ point as a fraction of
[st_azimuth](https://postgis.net/docs/ST_Azimuth.html) between the North pole
and the original vertice (again cast as `geography` to get the real world value)
divided by `2*pi()`, and uses
[st_lineinterpolate](https://postgis.net/docs/ST_LineInterpolatePoint.html)
to get the new location for the vertice.

I'm using a combination of _buffer_-_exteriorring_-_interpolatepoint_ here as it
was easier for me to wrap my head around instead of doing trigonometry myself.

As we've kept the order how the points were arranged (order) to start out with
we'll put them back together into lines in the same order. Additionally we need
the _outer bounds_ because otherwise we'd have only a _hole_ of the polygon
(the rest of the world except Antarctica).

Per every 1-360 degrees we'll use
[st_collect](https://postgis.net/docs/ST_Collect.html) on the lines made
up of projected points with [st_makeline](https://postgis.net/docs/ST_MakeLine.html).
Node the linestrings with [st_node](https://postgis.net/docs/ST_Node.html), and
build polygons with [st_buildarea](https://postgis.net/docs/ST_BuildArea.html)
for every timestamp.

The 1-360 degrees are converted to timestamps as

```
('2022.11.20 '||(((24.0 * d) / 360.0||' hours')::interval)::time)::timestamp
```

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-20-favourite.gif)

{{< highlight sql >}}
with
    d as (
        select
            geom
        from
            decode(
        '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
        Fav4Ag==', 'base64'
            ) d
                join lateral
                    st_geomfromtwkb(d) geom on true
    )
select
    row_number() over()::int as oid,
    id,
    ('2022.11.20 '||(((24.0*d)/360.0||' hours')::interval)::time)::timestamp as d,
    st_buildarea(st_node(st_collect(l))) as geom
from (
    select
        id, d, path[1] as part, path[2] as ring,
        st_makeline(array_agg(geom order by path[3])) as l
    from (
        select
            id, d, path,
            st_lineinterpolatepoint(
                st_scale(
                    /* scale just a little bit bigger
                       should be done in the buffer size calc instead*/
                    st_rotate(
                        st_exteriorring(
                            st_buffer(
                                /* buffer distance needs to be scaled -
                                   the val that comes from straight st_distance
                                   is too much for Mercator*/
                                st_point(0, 0, 4326)::geography,
                                st_distance(
                                    st_point(0,90,4326)::geography,
                                    geom::geography
                                ) / pi()
                            )::geometry(polygon, 4326)
                        ),
                        radians(d),
                        st_point(0, 0, 4326)
                    ), 1.5, 1.5 /* don't distort here, let mercator do it instead*/
                ),
                st_azimuth(
                    st_point(0, 90, 4326)::geography,
                    geom::geography
                ) / (2*pi())
            ) as geom
        from
            (
                select 1 as id, (st_dumppoints(geom)).*  
                from d
            ) f,
            generate_series(0, 359, 1) d
        where
            /* or otherwise Mercator will complain*/
            st_y(geom) >-85

        union all
        /* add outer shell for every hole*/
        select
            1 as id, d, array[0, path[2], path[3]] as path, geom
        from
            (
                select (
                    st_dumppoints(
                        st_multi(
                            st_envelope(
                                st_makeline(
                                    array[
                                        st_point(-180,-85,4326),
                                        st_point(180,85,4326)
                                    ]
                                )
                            )
                        )
                    )
                ).*
            ) g,
            generate_series(0, 359, 1) d
    ) tc
    group by
        id, d, path[1], path[2]
) g
group by
    id, d
;
{{</ highlight >}}

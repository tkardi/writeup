---
title: "2022 / Day 13: Five minute map"
date: 2022-11-12T13:06:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Creating a five-minute graticule in PostGIS."
---
I had another idea first, but other than a joke (a bad one at that) there was
really nothing to tie it to the theme of "five minutes". Fortunately the following
surfaced as a replacement: how to create a **five minute graticule**.

There are other (more or less complicated) ways of achieving exactly the same
result but I'm going to be doing it through the generation of points every
`1/12`th of a degree (e.g. _five minutes_) spanning between -180 and 180
degrees west to east and 90 and -90 degrees north to south. The points are
shifted from the origin at `[0,0]` using
[st_translate](https://postgis.net/docs/ST_Translate.html) with longitude and
latitude 5-minute values created with `generate_series`:

- `generate_series(-90 * 12, 90 * 12, 1)` for creating north-south 5 minute shifts
- `generate_series(-180 * 12, 180 * 12, 1)` for creating west-east 5 minute shifts

Essentially we could use any point of origin and generate a series with

- `generate_series(0, 180 * 12, 1)` for creating north-south 5 minute shifts
- `generate_series(0, 360 * 12, 1)` for creating west-east 5 minute shifts

aswell but then we'd have to do some extra mathematics to get the longitude and
latitude series values to the usual -180..180 and -90..90 range.

"Yes, but why are you babbling on with this point nonsense?" you might ask.
Well you don't have to deal with it if you're exclusively working with
Plate Carrée where you can simply define the first and last
points of graticule lines and be done with it. Problems arise as soon as you
want to transform yourself out from the equirectangular world, e.g. to
your local coordinate system. In that case, the more nodes you have - the better.

As an example here, consider a line like:

```
select
    st_makeline(
        st_point(0,53.876776,4326),
        st_point(62,53.876776,4326)
    )
```

which is a _perfectly straight line_ (on the "sphere") connecting the coast of
UK to the border of Kazakhstan. Now if I wanted to
[st_transform](https://postgis.net/docs/ST_Transform.html) it to for example
the Estonian national coordinate reference system L-EST'97
([epsg:3301](http://epsg.io/3301)), I'd expect the curvature of the sphere to
kick in, but it doesn't:

```
select
    st_transform(
        st_makeline(
            st_point(0,53.876776,4326),
            st_point(62,53.876776,4326)
        ),
        3301
    )
```
![A very sad transform from epsg:4326 to epsg:3301 - this should end up
being a curve not a straight line](../img/d13-2022/sad-transform.png)

How can we enforce this _curvature_? Simples! By adding more nodes. The more
the merrier... :) So the previous example can be re-done with
[st_segmentize](https://postgis.net/docs/ST_Segmentize.html) as

```
select
    st_transform(
        st_segmentize(
            st_makeline(
                st_point(0,53.876776,4326),
                st_point(62,53.876776,4326)
            ),
            0.1
        ),
        3301
    )
```
![A very happy transform from epsg:4326 to epsg:3301](../img/d13-2022/happy-transform.png)

By adding a node on the linestring every 0.1 degrees (because the
input of distance to [st_segmentize](https://postgis.net/docs/ST_Segmentize.html)
goes in the unit of srid - [epsg:4326](http://epsg.io/4326), therefore degrees)
we'll arrive at a far better looking line which can be more easily transformed
between different reference systems.

So essentially this point generation + aggregating points to lines
business that follows in todays query could instead
(and even more efficiently i think) be replaced by

```
select
    st_segmentize(
        st_makeline(
            st_point(-180, s/12.0, 4326),
            st_point(180, s/12.0, 4326)
        ),
        1.0/12.0
    )
from
    generate_series(-90 * 12, 90 * 12, 1) s
```

for creating parallel lines for the graticule "every 5 minutes". But I'll still
take the longer route. Mainly because it will make generating the graticule
for a _specific region only_ a lot easier. We'd only need to add another
CTE to define the bounds and then before calling
[st_makeline](https://postgis.net/docs/ST_MakeLine.html)
filter out only those points that are within the desired area of interest.

And of course - we could use the
[st_squaregrid](https://postgis.net/docs/ST_SquareGrid.html) for doing exactly
all of this but where's the fun in that? :) Additionally,
[st_squaregrid](https://postgis.net/docs/ST_SquareGrid.html) gives us polygons
not east/west + south/north spanning lines.

The output image for today will feature the freshly baked
five-minute-graticule transformed to [epsg:3301](http://epsg.io/3301) with
the overlaid bounds of the TMS (local) gridset for the spatial reference system
(unit of length is a meter), used e.g. by the
[Estonian Land Board](https://tiles.maaamet.ee/tm/tms/1.0.0/kaart@LEST).

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-13-fiveminutes.png)

{{< highlight sql >}}
with
    d as (
        select
            lon, lat,
            st_translate(
                st_point(0,0,4326),
                lon/12.0,
                lat/12.0
            ) as geom
        from
            generate_series(-90*12, 90*12, 1) lat,
            generate_series(-180*12, 180*12, 1) lon
    ),
    mers as (
        select
            lon,
            case
                when lon::int=lon then true
                else false
            end as deg,
            geom
        from (
            select
                round(lon/12.0, 2) as lon,
                st_makeline(
                    array_agg(geom order by lon, lat)
                ) as geom
            from
                d
            group by
                lon
        ) f
    ),
    pars as (
        select
            lat,
            case
                when lat::int=lat then true
                else false
            end as deg,
            geom
        from (
            select
                round(lat/12.0, 2) as lat,
                st_makeline(
                    array_agg(geom order by lat, lon)
                ) as geom
            from
                d
            group by
                lat
        ) f
    )
select
    row_number() over()::int as oid, *
from (
    select lat as v, deg, geom from pars
    union all
    select lon as v, deg, geom from mers
) u
;
{{</ highlight >}}

---
title: "2022 / Day 14: Hexagons"
date: 2022-11-13T18:49:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Creating hexagons using PostGIS. Only where you really need them."
---
Inadvertently I used up my hexagon idea for
[2022 / Day 07: Raster](../07-raster-2022/). So instead we'll look into
another grid-creation issue I bumped into this summer.

Imagine You'd have to create a hexgrid over a large area in quite a large scale.
PostGIS can generate this really nicely with
[st_hexagongrid](https://postgis.net/docs/ST_HexagonGrid.html). It consumes
a requested grid size and the bounding geometry.

The only problem being that the Estonian territorial waters area that I needed
it for is for one _banana-shaped_ - from the Bay of Finland
in the North-East through the Baltic Sea in the North-West, towards the
Gulf of Riga down in the South-West. And if that wasn't enough there's islands,
and some of them are of some size. Which means that doing a 10-meter grid for
the sea would essentially mean generating millions upon millions of hexagons,
only to dump at least 3/4 of them afterwards. Because they are on land :)

So, what if we used a _tile-based_ approach instead. Meaning first do
a more general grid with [st_squaregrid](https://postgis.net/docs/ST_SquareGrid.html)
get the grid cells that intersect the area of interest and then generate
hexagons only into the those squares (and then dump the bits you don't need).

The following query does just that - casts a _half moon full of hexes_ without
generating them everywhere.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-14-hexagons.png)

{{< highlight sql >}}
with
    minmax as (
        select 1 as oid,
            st_point(
                40500.000000,5993000.000000,3301
            ) as ll,
            st_point(
                1064500.000000,7017000.000000,3301
            ) as ur
    ),
    bounds as (
        select
            st_envelope(
                st_collect(
                    array[ll, ur]
                )
            ) as geom,
            (st_x(ur)-st_x(ll)) as width,
            st_x(ll) + (st_x(ur)-st_x(ll)) * (2.0 / 3.0) as r_x,
            st_y(ll) + (st_y(ur)-st_y(ll)) * (1.0 / 3.0) as r_y,
            st_x(ll) + (st_x(ur)-st_x(ll)) / 3.0 as l_x,
            st_y(ll) + (st_y(ur)-st_y(ll)) * (2.0 / 3.0) as l_y
        from minmax
    ),
    moon as (
        /* Represents the area of interest. I'll make it a half-moon-like
           but can be anything. even islands e.g*/
        select
            m.geom, e.minx, e.maxx, focal_point, env
        from (
            select
                width,
                st_point(r_x, r_y, 3301) as focal_point,
                st_difference(
                    st_buffer(st_point(l_x, l_y, 3301), width / 3.0),
                    st_buffer(st_point(r_x, r_y, 3301), width / 2.0)
                ) as geom
            from
                bounds
        ) m
        join lateral
            st_envelope(m.geom) env on true
        join lateral (
            select
                min(st_x(d.geom)) as minx,
                max(st_x(d.geom)) as maxx
            from
                st_dumppoints(env) d
        ) e on true
    ),
    tiles as (
        /* create smaller "tiles" we can use for generating a hexes in
           reasonable places*/
        select
            (moon.maxx - moon.minx)/10.0 as tilewidth, g.*
        from
            moon
                join lateral
                    st_squaregrid(
                        (moon.maxx - moon.minx)/10.0,
                        moon.env
                    ) g on true
        where
            st_intersects(moon.geom, g.geom)
    )
/* and create the hexgrid, tile by tile. Adding a something extra (st_scale)
   for the image to not look just like a hexgrid. As if it was
   measuring something :D*/
select
    row_number() over()::int as oid, *
from (
    select
        distinct on (h.i, h.j)
        st_scale(
            h.geom,
            st_point(
                (min(l) over() / l),
                (min(l) over() / l)
            ),
            st_centroid(h.geom)
        ) as geom
    from
        moon,
        tiles
            join lateral
                /* generate a hexagongrid within a specific tile using
                   tilewidth/20.0 as grid step*/
                st_hexagongrid(
                    tiles.tilewidth/20.0,
                    tiles.geom
                ) h on true
            join lateral
                /* just fooling around to scale the hexes
                   afterwards*/
                st_distance(
                    moon.focal_point,
                    st_centroid(h.geom)
                ) as l on true
    where
        st_intersects(moon.geom, h.geom)
) f
;
{{</ highlight >}}

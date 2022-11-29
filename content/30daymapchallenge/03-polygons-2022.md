---
title: "2022 / Day 03: Polygons"
date: 2022-11-03T09:15:50+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Generating quad-tree polygons around 'obstacles' in PostGIS"
---

Usually indexes are used to find something. In this case I'll be building
a `quadtree` "index" to index empty space - where nothing is found. A `quadtree`
is a tree data structure in which each internal `node` (not to be confused with
`vertex`) has exactly four children (
[Quadtree Wikipedia page](https://en.wikipedia.org/wiki/Quadtree)). For example
a TMS tile scheme is essentially a `quadtree`.

This exercise will partly build on top of the SQL from
[2022 / Day 1: points](../01-points-2022/) to prepare some _blah_ data, and
then generate quadtree polygons of "nothing-to-see here" around them.

The query below is a
[_recursive query_](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE).
It starts with building the bounding box (tile at z=0) and generates the
_obstacles_ using
[st_clusterkmeans](https://postgis.net/docs/ST_ClusterKMeans.html) and then
builds a [st_convexhull](https://postgis.net/docs/ST_ConvexHull.html) around
the clusters in the _non-recursive term_. And then in the _recursive term_ goes
on to divide the tile in the previous level into 4 until `maxz` is reached (set
in the _non-recursive term_) or the previous level tile does not intersect
any of the obstacles.

Since version 3.1 PostGIS has a
[st_squaregrid](https://postgis.net/docs/ST_SquareGrid.html)
function that could be used here instead. But since I need my tiles to line up
with my bounding box it seemed easier to calculate the tile geometries myself
rather than dive into the mathematics of required delta values for
[st_translate](https://postgis.net/docs/ST_Translate.html).

The final
```
where
    has_intersection = false
```

ensures that only those tiles which have nothing in them are returned.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-3-polygons.png)

{{< highlight sql >}}
with
    recursive quadtree as (
        select
            0 as z, 8 as maxz, e.geom as obs, e.bounds as geom,
            0 as x, 0 as y,
            m.maxy-m.miny as width, m.minx, m.miny,
            st_intersects(e.geom, e.bounds) as has_intersection
        from (
            select
                st_collect(geom) as geom, (array_agg(bounds))[1] as bounds
            from (
                select
                    cl, count(1),
                    st_convexhull(st_collect(geom)) as geom,
                    min(bounds)::geometry(polygon, 3301) as bounds
                from (
                    select
                        st_clusterkmeans(
                            d.geom, 256, 100000
                        ) over () as cl,
                        d.geom, bounds.geom as bounds
                    from (
                        select
                            st_envelope(
                                st_collect(
                                    array[
                                        st_point(
                                            40500.000000,5993000.000000,3301
                                        ), st_point(
                                            1064500.000000,7017000.000000,3301
                                        )
                                    ]
                                )
                            ) as geom
                    ) bounds
                        join lateral st_generatepoints(bounds.geom, 1000) pts on true
                        join lateral st_dump(pts) d on true
                ) h
                group by cl
                having count(1) > 5
            ) d
        ) e
            join lateral (
                select
                    min(st_x(p.geom)) as minx, min(st_y(p.geom)) as miny,
                    max(st_y(p.geom)) as maxy
                from
                    st_dumppoints(bounds) p
            ) m on true
        union all
        select
    		    z, maxz, obs, geom, x, y, width, minx, miny,
    		    st_intersects(obs, geom) as has_intersections
		    from (
            select
                q.z + 1 as z, q.maxz, q.obs,
                st_envelope(
                    st_collect(
                        array[
                            st_point(
                                q.minx + xy.x::numeric * q.width / 2.0,
                                q.miny + xy.y::numeric * q.width / 2.0,
                                3301
                            ),
                            st_point(
                                q.minx + (xy.x::numeric + 1.0) * q.width / 2.0,
                                q.miny + (xy.y::numeric + 1.0) * q.width / 2.0,
                                3301
                            )
                        ])) as geom,
                xy.x, xy.y,
                q.width / 2.0 as width,
                q.minx, q.miny
            from
                quadtree q
                    join lateral (
                        select q.x * 2 + x as x, q.y * 2 + y as y
                        from
                            generate_series(0,1) x,
                            generate_series(0,1) y
                    ) xy on true
            where
                /* divide until maxz is reached or parent tile has intersections
                   with obstacles */
                q.z < q.maxz and
                q.has_intersection = true
        ) w
        where
            st_within(w.geom, w.obs) = false
    )
select
    oid, z, x, y, geom, width
from (
    select
        row_number() over()::int as oid,
    	  z, x, y, maxz, obs, width,
        geom
    from
        quadtree
    where
        has_intersection = false
) q
union all
select
    0 as oid, null, null, null, obs, null
from
    quadtree
where
    z = 0
;
{{</ highlight >}}

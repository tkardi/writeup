---
title: "2022 / Day 06: Network"
date: 2022-11-05T14:39:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Building an ad hoc graph to do 'open area routing' with pg_routing in PostGIS."
---
Ok. So there might have been a method in the madness i've been producing this
year for the 30DayMapChallenge because it kind of dawned this morning that
for todays theme of _network_ I could actually tinker some of the previous days
together because they do help us solve a problem - open area shortest
path finding around obstacles.

The SQL below is overly complicated trickery at some points but the whole
intention over these past days of the challenege has been for me to
compose "single line" (wow, really? :D) SQL statements that must be
executable essentially in a blank PostGIS/PostgreSQL database. Today
unfortunately i sway off this track a little bit - the statement is still a
single one, with no intermediate tables or indexes or clusters for tables
created - hence patience in waiting for the result might be required. But
you'd have to have [`pg_routing`](https://pgrouting.org/) extension in the DB
aswell (and if not then the easiest to get up and running with
[`pg_routing`](https://pgrouting.org/) is most probably using a
[docker image](https://hub.docker.com/r/pgrouting/pgrouting/).

Right. but enough of this and lets dive into details. The SQL builds on:

- [Day 01: points](../01-points-2022/) - for generating random obstacle data.
- [Day 02: lines](../02-lines-2022/) - for generating edges
- [Day 03: polygons](../03-polygons-2022/) - for generating a quadtree of
_open areas_

The main idea how to solve this issue is to:

- divide the open area at hand into a quadtree,
- create nodes on every quadtree envelope spaced at the minimal quadtree box width
- condense these nodes into a graph (of edges)
- and then route from one corner of the open area to the other

There are a couple of challenges here...

[But as me high-school English teacher used to say:
"Challenges exist for you to overcome them."]

First off - I don't want to start (nor finish) the path at a specific node, so
we'll need another kind of strategy for generating edges in the tile from which
we start moving. One possibility (applied here aswell) is to find the tile from
where the journey begins (or finishes) and then have edges spanning from the input
location to every other node in tile. Selecting the "nearest edge" in this case
might end up leading you in a very wrong direction in the first place :).

Secondly - the general form of calling the `pgr_*` families of functions is

```
select
   f.*
from pgr_<some_function>(
      'select id, source, target, cost from foo.bar',
      start_node, end_node
) f
```

meaning we'll have a problem when it's only in the CTE that we define the
graph data. Sure, the CTE can go into `pgr` query definition too but in the
spirit of the #30DayMapChallenge i would like to export an image of the whole
thing aswell in QGIS. Could have two separate queries for building the graph but
since the quadtree tiling is random (because obstacle generation is based on
random points) the results would vary. Luckily [PostgreSQL has very nice JSON
handling capabilities](https://www.postgresql.org/docs/current/functions-json.html).
So I can still build the graph in the overall CTE, and then for routing
I can aggregate the whole thing into a `jsonb` array, and pass that to
[`pgr_dijkstra`](https://docs.pgrouting.org/3.1/en/pgr_dijkstra.html) for
unpacking.

Some differences in the implemetation (vs the original days)

- generation of nodes from quadtree boxes is now based on
[st_dumpsegments](https://postgis.net/docs/ST_DumpSegments.html)
- calculation of `node_id` values is based on
[st_geohash](https://postgis.net/docs/ST_GeoHash.html), essentially taking the
`md5` value of geohash and then turning that hex into a bigint:

```
(
  'x'||md5(
    st_geohash(
      st_transform(
        pnt.geom,
        4326
      )
    )
  )
)::bit(64)::bigint as node_id
```

This strategy seemed to make more sense instead of spending time with
[st_union](https://postgis.net/docs/ST_Union.html) and
[st_dump](https://postgis.net/docs/ST_Dump.html) with thousands of points.

Everything else is pretty much the same. Only the routing bit is new,
everything else is recycled...

For a more vivid experience i did a few different exports (NB! note the
color/opacity/width params in the final query so QGIS can render all the necessary
things in one layer) and combined them into a gif.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-6-network.gif)

{{< highlight sql >}}
with
    recursive quadtree as (
        select
            0 as z, 8 as maxz, e.geom as obs, e.bounds as geom,
            0 as x, 0 as y,
            m.maxy-m.miny as width, m.minx, m.miny,
            st_intersects(e.geom, e.bounds) as has_intersection,
            array[m.minx, m.miny, m.maxx, m.maxy] as tile_bounds
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
                    max(st_x(p.geom)) as maxx, max(st_y(p.geom)) as maxy
                from
                    st_dumppoints(bounds) p
            ) m on true
        union all
        select
    		    z, maxz, obs, geom, x, y, width, minx, miny,
    		    st_intersects(obs, geom) as has_intersections,
                tile_bounds
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
                q.minx, q.miny,
                array[
                    q.minx + xy.x::numeric * q.width / 2.0,
                    q.miny + xy.y::numeric * q.width / 2.0,
                    q.minx + (xy.x::numeric + 1.0) * q.width / 2.0,
                    q.miny + (xy.y::numeric + 1.0) * q.width / 2.0
                ] as tile_bounds                
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
    ),
    q as (
        /* This is now our quadtree tiling of open area */
        select
            oid, z, x, y, geom, width, min_width, tile_bounds
        from (
            select
                row_number() over()::int as oid,
    	          z, x, y, maxz, width, min(width) over () as min_width,
                tile_bounds,
                geom
            from
                quadtree
            where
                has_intersection = false
        ) qt
    ),
    pts as (
        select
            /* keep ref to quadtree.z to calculate opacity value for edge
               later on.
            */
            q.oid as qt_oid, q.z,
            q.tile_bounds[1] as minx, q.tile_bounds[2] as miny,
            q.tile_bounds[3] as maxx, q.tile_bounds[4] as maxy,
            segs.path[1] as box_side, pts.path[1] as pts_ord,
            pts.geom,
            st_x(pts.geom) as x,
            st_y(pts.geom) as y,
            ('x'||md5(node_id))::bit(64)::bigint as node_id
        from q
            join lateral st_dumpsegments(st_boundary(q.geom)) segs on true
            join lateral st_segmentize(segs.geom, q.min_width) as s on true
            join lateral st_dumppoints(s) pts on true
            join lateral st_geohash(st_transform(pts.geom, 4326)) node_id on true
    ),
    stops as (
        /* stops is our first and last point of the trip. For the quadtree
           tile we find here we'll create custom edges from start/end point
           to every tile node.
        */
        select stop.i as node_id, stop.geom, qt.qt_oid
        from
            unnest(
                array[
                    st_point(120459,6933618, 3301),
                    st_point(997591,6054977, 3301)
                ]
            ) with ordinality
                stop(geom, i)
                    join lateral (
                        /* Find the CLOSEST(!) quadtree tile
                           The closest because we might might be inside an
                           obstacle :)
                        */
                        select
                            q.oid as qt_oid
                        from
                            q
                        where
                            st_dwithin(q.geom, stop.geom, 100000)
                        order by
                            q.geom <-> stop.geom
                        limit 1
                    ) as qt on true
    ),
    network as (
        select
            row_number() over()::int as oid,
            qt_oid,
            node_ids[1] as source, node_ids[2] as target,
            geom, st_length(geom) as cost, st_length(geom) as reverse_cost,
            opacity
        from (
            select
                /* get distinct pairs regardless on direction by */
                row_number() over (partition by qt_oid, node_ids) as x,
                qt_oid, node_ids, geom,
                ((z * 32) / 8) as opacity
            from (
                select
                    a.qt_oid,
                    a.z,
                    case
                        when a.node_id < b.node_id then
                            st_makeline(array[a.geom, b.geom])
                        else
                            st_makeline(array[b.geom, a.geom])
                    end as geom,
                    case
                        when a.node_id < b.node_id then
                            array[a.node_id, b.node_id]
                        else
                            array[b.node_id, a.node_id]
                    end as node_ids
                from
                    pts a,
                    pts b
            where
                /* for only those qt tiles that are not covered by start/stop
                   edges we'll build in a second.
                */
                not exists (
                    select 1 from stops where qt_oid = a.qt_oid
                ) and
                a.qt_oid = b.qt_oid and
                a.box_side != b.box_side and (
                    (
                        not (a.y = b.y or a.x = b.x ) or
                        (a.y = b.y and a.y != all(array[a.miny, a.maxy])) or
                        (a.x = b.x and a.x != all(array[a.minx, a.maxx]))
                    ) or  (
                        a.pts_ord = 1 and
                        b.pts_ord = 1
                    )
                )
            ) v
            union all
            /* AND now for custom edges for start/stop */
            select
                1 as x, stops.qt_oid, array[stops.node_id, pts.node_id] as node_ids,
                st_setsrid(st_makeline(stops.geom, pts.geom),3301) as geom,
                112 as opacity
            from
                stops,
                pts
            where
                pts.qt_oid = stops.qt_oid
        ) b
        where
            x = 1
    ),
    datas as (
        /* PACK this up into jsonb for pgr */
        select
            to_jsonb(array_agg(n)) as network
        from (
            select
                oid as id, source, target, cost, reverse_cost
            from
                network
        ) n
    ),
    paths as (
        /* Run the routing and construct the linestring of the path
        */
        select
            st_makeline(
                array_agg(
                    coalesce(pts.geom, stops.geom) order by p.path_seq
                )
            ) as geom
        from (
            select
                pgr.*
            from
                datas
                    join lateral pgr_dijkstra(
                        format('
                            select
                                id, source, target, cost, reverse_cost
                            from
                                jsonb_to_recordset(%L::jsonb)
                                    as x(
                                        id int,
                                        cost numeric,
                                        source bigint,
                                        target bigint,
                                        reverse_cost numeric
                                    )',
                            datas.network
                        ),
                        1, 2
                    ) pgr on true
        ) p
            left join pts on pts.node_id = p.node
            left join stops on stops.node_id = p.node
    )
select
    row_number() over(order by ord)::int as oid, *
from (
    /* data for the routing graph. will got to the very bottom */
    select
        0 as ord, geom,
        79 as red, 85 as green, 255 as blue,
        opacity as opacity, 0.4 as width
    from
        network
    union all
    /* the boundaries of obstacles */
    select
        1 as ord, st_boundary(obs) as geom,
        255 as red, 8 as green, 28 as blue,
        255 as opacity, 2 as width
    from
        quadtree
    where
        z = 0
    union all
    /* the pg_routing calculated shortest path */
    select
        2 as ord, geom,
        215 as red, 217 as green, 255 as blue,
        255 as opacity, 2 as width
    from
        paths
) res
order by
    ord
;
{{</ highlight >}}

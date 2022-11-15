---
title: "2022 / Day 15: Food or drink"
date: 2022-11-15T07:53:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: PostGIS has a st_letters function which returns polygons made out of input text."
---
There's not really much to say about this one. The query below uses the
PostGIS 3.3 function [st_letters](https://postgis.net/docs/ST_Letters.html)
to draw a glass and place a bunch a bubbles on top of it. All
done using only letters "Y", "I" , and "O".

Best served with the neon lights buzzing sound in the background.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-15-food-or-drink.gif)

{{< highlight sql >}}
with
    glass as (
        select
            st_buildarea(st_exteriorring(st_union(geom))) as geom
        from (
            select st_scale(st_letters('Y'),500, 300) as geom
            union
            select st_translate(st_scale(st_letters('I'), 1000, 33), 4000, 0)
            union
            select st_translate(st_scale(st_letters('I'), 1000, 10), 4000, 21400)
        ) d
    ),
    bounds as (
        select
            pts.minx, pts.maxx, pts.miny, pts.maxy,
            pts.maxx-pts.minx as width,
            pts.maxy-pts.miny as height
        from glass
            join lateral
                st_envelope(geom) env on true
            join lateral (
                select
                    min(st_x(d.geom)) as minx,
                    min(st_y(d.geom)) as miny,
                    max(st_y(d.geom)) as maxx,
                    max(st_y(d.geom)) as maxy
                from
                    st_dumppoints(env) d
            ) pts on true
    ),
    bubbles as (
        select
            st_rotate(
                st_buffer(
                    st_translate(
                        bubble,
                        st_x(d.geom),
                        st_y(d.geom)
                    ),
                    random()*1000.0
                ),
                2*pi()*random(),
                d.geom
            ) as geom
        from (
            select
                st_envelope(
                    st_collect(
                        array[
                            st_point(b.minx+(b.width/5.0),b.maxy+(b.height/10,0)),
                            st_point(b.maxx-(b.width/10.0),b.maxy+(b.maxy/3.0))
                        ]
                    )
                ) as env
            from
                bounds b
        ) b
            join lateral
                st_scale(st_letters('o'),10, 5) bubble on true
            join lateral
                st_generatepoints(b.env, 10) pts on true
            join lateral
                st_dump(pts) d on true
    )
select
    row_number() over()::int as oid, geom, cl
from (
    select
        st_buffer(
            st_buffer(
                glass.geom,
                -500
            ),
            500
        ) as geom, 'glass' as cl
    from
        glass
    union all
    select
        geom, 'bubbles' as cl
    from
        bubbles
) f
;
{{</ highlight >}}

---
title: "2022 / Day 16: Minimal"
date: 2022-11-17T08:15:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Geometry approximations as circles or rectangles using PostGIS"
---
There's a whole bunch of functions in PostGIS to _approximate_ a geometry. This
might be needed for a whole array of reasons, e.g

- where's the best spot for a label location?
- quick testing of spatial relations
- the robustness of a geometry

Todays SQL will check out four of those functions to plot out their behaviour on
a set of random concave polygons:

- [st_maximuminscribedcircle](https://postgis.net/docs/ST_MaximumInscribedCircle.html)
which finds the largest circle that is contained within a geometry. Imagine
this to be the _last-location-standing_ when buffering a polygon inwards
step by step.
- [st_minimumboundingcircle](https://postgis.net/docs/ST_MinimumBoundingCircle.html)
which finds the smallest circle that contains the whole geometry.
- [st_envelope](https://postgis.net/docs/ST_Envelope.html) which is your
regular geometry bounding box
- [st_orientedenvelope](https://postgis.net/docs/ST_OrientedEnvelope.html) which
is the minimum area rotated rectangle that contains the geometry.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-16-minimal.png)

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
            st_x(ur) - st_x(ll) as width
        from minmax
    ),
    clusters as (
        select
            pt.path[1] as oid, pt.geom,
            st_clusterkmeans(
                pt.geom,
                30,
                10000000
            ) over () cl
        from
            bounds
                join lateral
                    st_generatepoints(
                        bounds.geom,
                        1000
                    ) pts on true
                join lateral
                    st_dump(
                        pts
                    ) pt on true
    ),
    areas as (
        select
            cl as cluster_id, count(1),
            st_concavehull(st_collect(geom),0.0) as geom
        from
            clusters
        group by
            cl
        having
            count(1) > 5
    )
select
    row_number() over()::int as oid,
    case when cl = 'b' then st_expand(geom, bounds.width/20.0) else geom end as geom
    cl
from
    bounds, (
    /* maximuminscribedcircle - leave in the original location*/
    select geom, cl
    from (
        select
            geom, 'original' as cl
        from areas
        union all
        select
            st_buffer(mic.center, mic.radius) as geom, 'maximuminscribedcircle' as cl
        from areas
            join lateral st_maximuminscribedcircle(geom) mic on true
        union all
        select
            geom, 'b' as cl
        from bounds
    ) mic
    union all
    /* minimumboundingcircle - shift right*/
    select
        st_translate(
            minc.geom,
            bounds.width + (bounds.width/10.0),
            0
        ) as geom, cl
    from
        bounds, (
            select
                st_minimumboundingcircle(geom) as geom,
                'minimumboundingcircle' as cl
            from areas
            union all
            select
                geom, 'original' as cl
            from areas
            union all
            select
                geom, 'b' as cl
            from bounds
        ) minc
    union all
    /* envelope - shift down*/
    select
        st_translate(
            oe.geom,
            0,
            -1*( bounds.width + (bounds.width/10.0))
        ) as geom, cl
    from
        bounds, (
            select
                st_envelope(geom) as geom, 'envelope' as cl
            from areas
            union all
            select
                geom, 'original' as cl
            from areas
            union all
            select
                geom, 'b' as cl
            from bounds
        ) oe
    union all
    /* orientedenvelope - shift down and right*/
    select
        st_translate(
            oe.geom,
            bounds.width + (bounds.width/10.0),
            -1*( bounds.width + (bounds.width/10.0))
        ) as geom, cl
    from
        bounds, (
            select
                st_orientedenvelope(geom) as geom,
                'orientedenvelope' as cl
            from areas
            union all
            select
                geom, 'original' as cl
            from areas
            union all
            select
                geom, 'b' as cl
            from bounds
        ) oe
) d
;
{{</ highlight >}}

---
title: "2022 / Day 29: Out of my comfort zone"
date: 2022-11-28T15:35:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Mixing RGB colors using PostGIS."
---
I don't feel comfortable around colors (as you might guess by the
mostly b/w schemes I've been using this past challenge). So this is me trying
to familiarize myself a bit with colors. And I still don't feel comfortable.
Please take me away from here.

But let's get this over with. The following SQL draws three circles and colors
them based on the azimuth (from the central point) and the third one also
distance from the central point. All are divided into thee sectors.
The first one top left interpolates a single RGB color from min (0) to max (255)
in clockwise direction. The second one (top right) interpolates two colors in
mutually countering direction. So e.g. the downwards sector is a combination
of `rgb(255,0,0)` on the lefthand side and `rgb(0,255,0)` on the right with
`rgb(128,128,0)` somewhere in the middle. The other sectors are mixed
green and blue and blue and red.

The third one (middle bottom) combines all three colors - e.g. the downwards
sector is calculated with red increasing clockwise, greens increasing
counter-clockwise and blue increasing with distance from the center. And
additionally a (360/255) degree turn with every blue DN increase.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-29-comfort.png)

{{< highlight sql >}}
with
    center as (
        select st_point(0,0) as geom, 1000 as w
    ),
    data as (
        select
            s, width,
            mod(s-1, 256) as q,
            (s-1) / 256 as sector,
            st_rotate(
            st_polygonize(
                st_addpoint(
                    st_addpoint(
                        st_linesubstring(
                            geom,
                            (s-1)::numeric/(3*256)::numeric,
                            s::numeric/(3*256)::numeric
                        ),
                        center,
                        0
                    ),
                    center,
                    -1
                )
            ),
		    radians(-30)
			) as geom
        from (
            select
                st_boundary(
                    st_buffer(center.geom, center.w, 'quad_segs=80')
                ) as geom,
                center.geom as center, center.w as width
            from
                center
        ) f, generate_series(1,3*256,1) s
        group by s, width
        order by s
    ),
    verts as (
        select
            ring,
            coalesce(
                st_difference(
                    geom,
                    lag(geom) over(order by ring)
                ),
                geom
            ) as geom
        from (
            select
                s-1 as ring,
                st_buffer(
                    center.geom,
                    (center.w::numeric/256.0) * s::numeric,
                    'quad_segs=80'
                ) as geom
            from
                center,
                generate_series(1,256, 1) s
        ) a
    ),
    onecolor as (
        select
            st_translate(
                geom,
                -1.0 * (width + 250.0),
                (width + 250.0)
            ) as geom,
            col[1] as red, col[2] as green, col[3] as blue, sector
        from (
            select
                st_collectionextract(geom,3) as geom,
	              sector, width,
                case
                    when sector=0 then
                        array[q,0, 0]
                    when sector=1 then
                        array[0,0,q]
                    when sector=2 then
                        array[0,q,0]
                end as col
            from
                data
        ) f
    ),
    twocolors as (
        select
            st_translate(
                geom,
                (width + 250.0),
                (width + 250.0)
            ) as geom,
            col[1] as red, col[2] as green, col[3] as blue, sector
        from (
            select
                st_collectionextract(geom,3) as geom,
	            sector, width,
                case
                    when sector=0 then
                        array[q,255-q, 0]
                    when sector=1 then
                        array[255-q,0,q]
                    when sector=2 then
                        array[0,q,255-q]
                end as col
            from
                data
        ) f
    ),
    threecolors as (
        select
            st_translate(
                geom,
                0.0,
                -1.0 * (width/2.0 + 250.0)
            ) as geom,
            col[1] as red, col[2] as green, col[3] as blue, sector
        from (
            select
                st_rotate(
                    geom,
                    radians((360.0/255.0) * ring::numeric),
                    st_point(0,0)
                ) as geom,
	              sector, width,
                case
                    when sector=0 then
                        array[q,255-q, ring]
                    when sector=1 then
                        array[255-q,ring,q]
                    when sector=2 then
                        array[ring,q,255-q]
                end as col
            from (
                select
                    data.sector,
                    data.q,
                    data.width,
                    verts.ring,
                    st_intersection(x, verts.geom) as geom
                from
                    verts,
                    data
                        join lateral
                            st_collectionextract(data.geom, 3) x on true
                where
                    st_intersects(x, verts.geom)
            ) d
        ) f
    )
select
    row_number() over ()::int as oid,
    geom, sector, red, green, blue
from (
    select
        geom, sector, red, green, blue
    from
        onecolor
    union all
    select
        geom, sector, red, green, blue
    from
        twocolors
    union all
    select
        geom, sector, red, green, blue
    from
        threecolors
) d
;
{{</ highlight >}}

---
title: "2022 / Day 25: Two colors"
date: 2022-11-24T20:05:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Rework of 2022 / Day 7: Raster with adding a kinescope refresher swath"
---
This is a rework of [2022 / Day 7: Raster](../07-raster-2022/) and explores
the possibility of adding a kinescope _refresher-swath_. Meaning at some
points of time we crank up all the RGB color values for our pseudo-pixels
to 255 and then slowly return them to what they were before. This has been on
my mind since I came up with the kinescope idea about a year ago but never
really got a chance to get around to it. Still needs a bit of work to be
called _an accomplishment_ but as an idea it works :)

A nice overview of how a kinescope screen really
works is given by [The Slo Mo Guys](https://www.youtube.com/watch?v=3BJU2drrtCM).

So for the query we'll need to classify every separate gridline - based on that
assign a time when a certain line of the _pseudo pixels_ is supposed to light
up and then count backwards with a delay when to shut them off.

As with [2022 / Day 7: Raster](../07-raster-2022/) the following SQL snippet includes
Ukrainian first level administrative division unit data as
PostGIS `box2d` types prepared from
[NaturalEarth](https://www.naturalearthdata.com/downloads/10m-cultural-vectors/)
large scale cultural vector data.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-25-twocolors.gif)

{{< highlight sql >}}
with
    boxes as (
        select
            x.name::varchar,
            b as geom,
            c
        from (
            values
                (
                    'BOX(
                        30.508143345239148 50.34307729787895,
                        33.486979607922535 52.36894928000008)',
                    'Чернігівська область'
                ),(
                    'BOX(
                        23.59331913200012 50.29481151014306,
                        26.095802849721736 51.94975006099999)',
                    'Волинська область'
                ),(
                    'BOX(
                        25.06346276239657 50.01312327691767,
                        27.705161174000125 51.9285110470001)',
                    'Рівненська область'
                ),(
                    'BOX(
                        27.192273389805052 49.582012640770756,
                        29.736304151715217 51.65196462100006)',
                    'Житомирська область'
                ),(
                    'BOX(
                        29.270492791309266 49.17064240232497,
                        32.13915490067808 51.49313710600002)',
                    'Київ'
                ),(
                    'BOX(
                        27.370402052612633 48.00050207700009,
                        30.07612837062777 49.881400050771674)',
                    'Вінницька область'
                ),(
                    'BOX(
                        22.640922485000146 48.72369293957746,
                        25.43387942889251 50.64448008948449)',
                    'Львівська область'
                ),(
                    'BOX(
                        32.93951948423677 50.10722606143355,
                        35.69221013400005 52.36099110900001)',
                    'Сумська область'
                ),(
                    'BOX(
                        34.84147057490708 48.52703827589568,
                        38.080859409134405 50.431858216000066)',
                    'Харківська область'
                ),(
                    'BOX(
                        37.84733361175677 47.811527201000004,
                        40.1595430910001 50.06278513600007)',
                    'Луганська область'
                ),(
                    'BOX(
                        22.13283980300011 47.89702585900004,
                        24.641111281060773 49.08553212500006)',
                    'Закарпатська область'
                ),(
                    'BOX(
                        36.53113488087001 46.87555573100008,
                        39.05552859876582 49.23174978312596)',
                    'Донецька область'
                ),(
                    'BOX(
                        24.92755374520914 47.71393770200011,
                        27.503829794000126 48.678114326115235)',
                    'Чернівецька область'
                ),(
                    'BOX(
                        23.532858514209067 47.71006052700004,
                        25.64626956638739 49.52467763896186)',
                    'Івано-Франківська область'
                ),(
                    'BOX(
                        28.19949792500006 45.21356842700004,
                        31.306802605555788 48.22222484050394)',
                    'Одеська область'
                ),(
                    'BOX(
                        32.48161868600005 44.38104889500005,
                        36.63835696700005 46.21810560059299)',
                    'Автономна Республіка Крим'
                ),(
                    'BOX(
                        31.50440514400009 45.71732197013279,
                        35.279540987026905 47.5828581810602)',
                    'Херсонська область'
                ),(
                    'BOX(
                        34.203731724339605 46.25514515122774,
                        37.220911899964165 48.130189114436064)',
                    'Запорізька область'
                ),(
                    'BOX(
                        30.208988478335584 46.42987702000005,
                        33.136613396590974 48.220984605255296)',
                    'Миколаївська область'
                ),(
                    'BOX(
                        32.06949506974166 48.74627554028393,
                        35.475075311276555 50.54660492597651)',
                    'Полтавська область'
                ),(
                    'BOX(
                        26.148409458129947 48.45159068526186,
                        27.88897505105774 50.58714508677633)',
                    'Хмельницька область'
                ),(
                    'BOX(
                        24.707050408949215 48.5101142445767,
                        26.38053999242669 50.26525259133399)',
                    'Тернопільська область'
                ),(
                    'BOX(
                        32.983754509662845 47.47389842457261,
                        36.92718305755204 49.183044744919016)',
                    'Дніпропетровська область'
                ),(
                    'BOX(
                        29.59279869970021 48.45332184512705,
                        32.855441929101744 50.25145498296109)',
                    'Черкаська область'
                ),(
                    'BOX(
                        29.738164503688893 47.74752513317833,
                        33.919459669628054 49.26179962835016)',
                    'Кіровоградська область'
                ),(
                    'BOX(
                        30.185993737221168 50.014154717466226,
                        30.846045721033192 50.66815305072271)',
                    'Київ'
                )
        ) x(box, name)
            join lateral
                st_transform(
                    box::box2d::geometry(polygon, 4326),
                    6385
                ) b on true
            join lateral
                st_centroid(
                    b
                ) c on true
    ),
    bounds as (
        /* bounds for all geometries, and AOI width and height
        */
        select
            b.geom, coords.minx, coords.maxx, coords.miny, coords.maxy,
            (coords.maxx - coords.minx) as width,
            (coords.maxy - coords.miny) as height
        from (
            select
                st_envelope(st_collect(geom)) as geom
            from boxes
        ) b
            join lateral (
                select
                    min(st_x(pts.geom)) as minx, max(st_x(pts.geom)) as maxx,
                    min(st_y(pts.geom)) as miny, max(st_y(pts.geom)) as maxy
                from
                    st_dumppoints(b.geom) pts
            ) coords on true
    ),
    colors as (
        /* decide on colors - northern parts will get blue, and
           seouthern yellow
        */
        select
            boxes.geom, st_y(boxes.c) as y,
            case
                /* centroid is north of approximately 1/2 way bbox height -> blue */
                when st_y(boxes.c) > bounds.miny + (bounds.height * 0.58)
                    then array[0, 87, 183]
                /* otherwise -> yellow */
                else array[255, 215, 0]
            end as rgb
        from
            boxes,
            bounds
    ),
    grid as (
        /* create a hexagongrid assuming (bounds width in m) / 256 "px" size.
           shift will be the amount to translate r and b cells left / right
           from the green that will stay in the middle.
        */
        select
            row_number() over(order by h.i, h.j)::int as oid,
            bounds.width / px.v as upp, h.geom,
            (bounds.width/px.v) / 2.0 as shift,
            /* ADD "row" calc */
            case
                when mod(h.i,2) = 0 then j + (j - min(h.j) over())
                else j+(j - min(h.j) over()) + 1
            end as row_sub_nr,
            h.i as i
        from
            (select 256.0 as v) px,
            bounds
                join lateral
                    st_hexagongrid(
                        bounds.width / px.v,
                        bounds.geom
                    ) h on true
    ),
    centroids as (
        /* calculate the location of box centroids. we'll color those "white"
           meaning r=255,g=255,b=255
        */
        select
            grid.oid as grid_oid, g.name, grid.geom as geom
        from
            grid
                join lateral (
                    select
                        boxes.name, boxes.c
                    from
                        boxes
                    where
                        st_within(boxes.c, grid.geom)
                    order by
                        boxes.c <-> grid.geom
                    limit 1
                ) g on true
    ),
    centroids_halo as (
        /* .. and lets put a 1 "pixel" black halo around the centroid
           so they'll be better visible in the south (yellow background)
        */
        select
            grid.oid as grid_oid
        from
            grid, centroids c
        where
            c.geom && grid.geom
    ),
    ints as (
        /* add color values to grid */
        select
            grid.row_sub_nr, grid.i,
            grid.geom, grid.shift,
            case
                when centroids.grid_oid is not null then array[255,255,255]
                when centroids_halo.grid_oid is not null then array[0,0,0]
                else coalesce(f.rgb, array[0,0,0])
            end as rgb,
            centroids.name
        from
            grid
                 left join lateral (
                    select
                        colors.rgb
                    from
                        colors
                    where
                        st_intersects(grid.geom, colors.geom)
                    order by
                        y
                    limit 1
                ) f on true
                left join
                    centroids on
                        centroids.grid_oid = grid.oid
                left join
                    centroids_halo on
                        centroids_halo.grid_oid = grid.oid
    )
select
      row_number() over(order by all_seconds, row_sub_nr, i, ch)::int as oid, delay,
      ('2022.11.25 '||(((7.98 + all_seconds::numeric / 3600.0)||' hours')::interval)::time)::timestamp,
      ch, geom,
      case
          when color > 0 then
              255 - (((255-color)::numeric / 50.0) * delay::numeric)::int
          when delay < 3 then
              255 - (((255-color)::numeric / 50.0) * delay::numeric*2.0)::int
          else 0
      end as color,
      name,
      row_sub_nr, i
from (
    select
        light_up_second+delay as all_seconds,
        delay, row_sub_nr, i,
        ch, geom, color, name
    from
        generate_series(0, 50, 1) delay, (
            select
                /* Every "channel" gets its own color
                   assignment (int),
                   all others will be set to 0 for this channel cell*/
                row_number() over()::int as oid, s as ch,
                light_up_second,
                row_sub_nr, i,
                geoms[s] as geom,
                rgb[s] as color, name
            from
                generate_series(1,3,1) as s, (
                    select
                        /* the next would mean lighting up by columns in a row.
                           but for #30DayMapChallenge it creates way too much
                           still images :D*/
                        --row_number() over(order by row_sub_nr desc, i) as light_up_second,
                        /* so instead pick by rows*/
                        max(row_sub_nr) over() - row_sub_nr as light_up_second,
                        rgb, name, row_sub_nr, i,
                        array[
                            /* prepare geometry for red "channel" */
                            st_scale(
                                st_buffer(
                                    st_translate(
                                        st_centroid(d.geom),
                                        -1.0*d.shift,
                                        0
                                    ),
                                    shift/2.0
                                ),         
                                st_point(0.75, 3.0),
                                st_translate(
                                    st_centroid(d.geom),
                                    -1.0*d.shift,
                                    0
                                )
                            ),
                            /* prepare geometry for green "channel" */
                            st_scale(
                                st_buffer(
                                    st_centroid(d.geom),
                                    shift/2.0
                                ),
                                st_point(0.75, 3.0),
                                st_centroid(d.geom)
                            ),
                            /* prepare geometry for blue "channel" */
                            st_scale(
                                st_buffer(
                                    st_translate(
                                        st_centroid(d.geom),
                                        d.shift,
                                        0
                                    ),
                                    shift/2.0
                                ),
                                st_point(0.75, 3.0),
                                st_translate(
                                    st_centroid(d.geom),
                                    d.shift,
                                    0
                                )
                            )
                          ] as geoms
                    from
                        ints d
                ) g
        ) f
) h
;
{{</ highlight >}}

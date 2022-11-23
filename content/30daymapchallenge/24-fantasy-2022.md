---
title: "2022 / Day 24: Fantasy"
date: 2022-11-23T19:54:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Plans for Dantes Inferno as calculated by Antonio Manetti of Florence, and digitized using a PostGIS SQL query by one Tõnis of Tartu."
---
When preparing the [2022 / day 21: Kontur World Population](../21-kontur-2022/)
map and arriving at _grouped-at-equator_ idea it looked like a ring of fire, and
the train of thought through my head was:

- this looks like a ring of fire
- heh, Johnny Cash
- I wonder if Joaquin Phoenix sang himself in the movie?
- Really liked [Don't worry, he won't get very far on foot](https://en.wikipedia.org/wiki/Don%27t_Worry,_He_Won%27t_Get_Far_on_Foot)
- that Dan Brown-based movie a couple of nights ago I saw... meeeh.
- what was it called? There was reference to Dante and the Boticelli Inferno
painting.
- yeah, Inferno could be it. But a ring of fire? I could do a map on that.
- wait didn't Galileo participate in arguments establishing the location and
size of the Inferno?
- [checks interwebs]
- bingo! the Florentin [Museo Galilei](https://mostre.museogalileo.it/dante/en/the-underworld/44-form-and-measurements-of-hell.html)
gives some basic estimates that Antonio Manetti was using in his calculations.

So if the Earth has a circumference of 20400 miles - and that's most probably
in the olden units, which could be

```
1 miglio = 2833+1⁄3 braccii = 1653.67 m
```
as ["Italian units of measurement#Florence" @ Wikipedia](https://en.wikipedia.org/wiki/Italian_units_of_measurement#Florence)
suggests. But it really doesnt't matter in this case because we'll be doing this
completely unitless (`srid = 0`) with the structure taken from

![A diagram depicting Antonio Manetti's theories about the geography of Hell.
Image linked from https://www3.nd.edu/~italnet/Dante/](https://www3.nd.edu/~italnet/Dante/images/tp1595/1595.diag.175dpi.jpeg)

and some starter numbers:

- circumference of the Earth is `20400` which means the radius is `3245.45`
- every ring seems to be `405.0+(15.0/22.0)` width (cumulative sum)
- the _cuppola_ is 60° along the meridian line. Since the Earth is a sphere
that means that the _cuppola_ is an arc with a length of
`20400 / (360/60) = 3400`.

In the end all the numbers don't add up correctly for me, but the output looks
similarish. So will have to do at the moment.

This query is really a mess and needs to be looked at some later point of time.
But the general idea is to construct the vertical bisection (enlarged 60° cone)
separately from the rest of the rings.

So we construct the `cone_shapes` that we'll use afterwards as the `100-unit marker`
lines aswell.
Generate a series of buffers in `rings`. Build the vertical bisection parts
in `section_a` as areas cut at the correct angles with `cone_shapes`. Dump the
points, and dump the segments, and snap points on all lines. Calculate
shared paths to see which rings need to be _collapsed_ to straight lines.
Create the extra line for the river `Acheronte`. And then union it all
together trying to classify (using a single query and a single layer in QGIS
again) and label things.

Oh. And I hate labelling! With a passion.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-24-fantasy.png)

{{< highlight sql >}}
/* Hells depth re: Antonio Manetti is 3245 miles, cone
   vertex angle 60deg along meridian */
with
    /* the 60 degree cone lines used to divide up
       the buffers*/
    cone_shapes as (
        select
            s,
            st_reverse(
                st_makeline(
                    array[
                        st_startpoint(geom),
                        st_point(0,0,0),
                        st_endpoint(geom)
                    ]
                )
            ) as geom
        from (
            select
                s,
                st_linesubstring(
                    b.geom,
                    0 + (s::numeric*200.0)/st_length(b.geom),
                    2.0*(3400.0)/st_length(geom) -
                        (s::numeric*200.0)/st_length(b.geom)
                ) as geom    
            from (
                select
                    st_rotate(
                        st_boundary(
                            st_buffer(
                                st_point(0,0,0),
                                2.0*3245.45,
                                'quad_segs=64'
                            )
                        ),
                        radians(120)
                    ) as geom
            ) b,
            generate_series(0, 10, 1) s
        ) d
    ),    
    rings as (
        select
            /* geom - the outer ring
               p_geom - the inner ring
               v_geom - the vertical cone geom*/
            ring_no,
            coalesce(
                st_difference(
                    geom, lead(geom) over (order by ring_no)
                ),
                geom
            ) as geom,
            coalesce(
                st_difference(
                    geom,
                    p_geom
                ),
                geom
            ) as p_geom,
            coalesce(
                st_difference(
                    v_geom,
                    lead(v_geom) over (order by ring_no)
                ),
                v_geom
            ) as v_geom
        from (
            select
                abs(s-8) as ring_no,
                s * (405.0+(15.0/22.0)) as f,
                /* separate rings for outer rings*/
                st_rotate(st_setsrid(
                    st_buffer(
                        st_point(0,0,0),
                        s * (405.0+(15.0/22.0)),
                        'quad_segs=64'
                    ),0
                ), radians(90)) as geom,
                /* separate rings for inner*/
                st_rotate(st_setsrid(
                    st_buffer(
                        st_point(0,0,0),
                        s * (405.0+(15.0/22.0)) - p.w ,
                        'quad_segs=64'
                    ),0
                ), radians(90)) as p_geom,
                /* separate rings for the vertical section. easier this way*/
                st_setsrid(
                    st_buffer(
                        st_point(0,0,0),
                        sum(v.height) over (order by v.i rows unbounded preceding),
                        'quad_segs=64'
                    ),0
                ) as v_geom
            from
                generate_series(1,8,1) s
                    join lateral unnest(
                        array[
                            (730.0 +(5/22)),
                            (405.0 + (15.0/22.0)),
                            (405.0 + (15.0/22.0)),
                            (405.0 + (15.0/22.0)),
                            (405.0 + (15.0/22.0)),
                            (405.0 + (15.0/22.0)),
                            (405.0 + (15.0/22.0)),
                            (405.0 + (15.0/22.0))
                        ]
                    ) with ordinality v(height, i) on true
                    left join lateral unnest(
                        array[
                           /* off Manettis drawing. adds the last 87.5,0
                              to work. something is rotten here*/
                           75.0, 112.5, 50, 62.5, 75, 87.5, 87.5, 0
                        ]
                    ) with ordinality p(w, i) on true
            where
                 s = v.i and s=p.i   
            order by s desc
        ) r
    ),
    section_a as (
        select
            ring_no,
            geom as geom
        from (
            select
                r.ring_no, c.s,
                (st_dump(st_split(r.v_geom, c.geom))).*,
                o.elem, o.i
            from
                rings r,
                cone_shapes c,
                /* which cone line we'll use
                   to split the vertical rings*/
                unnest(
                    array[0,0,1,2,3,4,7,10]
                ) with ordinality o(elem, i)
            where
                o.i-1 = r.ring_no and
                c.s = o.elem
        ) f
        where
            path[1] = 2
    ),
    pts as (
        /* vertical cut levels need snapping to work*/
        select
            ring_no, pts.path[1] as point_ord,
            pts.geom as geom
        from section_a
            join lateral
                st_dumppoints(
                    st_removepoint(
                        st_boundary(section_a.geom),
                        0
                    )
                ) pts on true
    ),
    segs as (
        select
            ring_no, seg.path[1] as p, seg.geom
        from section_a
            join lateral
                st_dumpsegments(
                    st_boundary(section_a.geom)
                ) seg on true
    ),
    snapped as (
        select
            ring_no,
            st_makeline(
                array_agg(geom order by ring_no, p)
            ) as geom
        from (
            select
                segs.ring_no,
                segs.p,
                case
                    when pts.geom is not null then
                        st_addpoint(segs.geom, pts.geom, 1)
                    else
                        segs.geom
                end as geom
            from
                segs
                    left join pts on
                        pts.ring_no = segs.ring_no + 1 and
                        st_dwithin(pts.geom, segs.geom, 1) and
                        st_linelocatepoint(segs.geom, pts.geom) > 0 and
                        st_linelocatepoint(segs.geom, pts.geom) < 1
        ) f
        group by
            ring_no
    ),
    section_a_floors as (
        /* create vertical rings floors*/
        select
            path,
            case
                when
                    /* the cuppola remains an arc*/
                    left_ring_no is null and
                    right_ring_no = 0 then
                        st_removepoint(
                            st_removepoint(geom,st_numpoints(geom)-1),
                            0
                        )
                when
                    /* make floors straight*/
                    coalesce(left_ring_no, 0) > 0 and
                    coalesce(right_ring_no, 0) > 0 then
                        st_makeline(
                            st_startpoint(geom),
                            st_endpoint(geom)
                        )
                else
                    geom
            end as geom,
            left_ring_no, right_ring_no
        from (
            /* calculate left and right for vertical ring bisects*/
            select
                path[1], geom, lpt, rpt,
                l.ring_no as left_ring_no, r.ring_no as right_ring_no
            from (
                select
                    (st_dump(st_linemerge(st_union(geom)))).*
                from (
                    select
                        st_node(st_collect(geom)) as geom
                    from
                        snapped
                ) sec
            ) a
                join lateral
                    st_lineinterpolatepoint(
                        st_offsetcurve(a.geom, 0.1),
                        0.5
                    ) lpt on true
                join lateral
                    st_lineinterpolatepoint(
                        st_offsetcurve(a.geom, -0.1),
                        0.5
                    ) rpt on true
                left join lateral (
                    select
                        section_a.ring_no
                    from
                        section_a
                    where
                        st_within(lpt, section_a.geom)
                ) l on true
                left join lateral (
                    select
                        section_a.ring_no
                    from
                        section_a
                    where
                        st_within(rpt, section_a.geom)
                ) r on true
        ) d
    ),
    acheronte as (
        /* extra line for the river*/
        select
            0 as path,
            st_intersection(
                l,
                st_buildarea(
                    st_addpoint(s.geom, st_pointn(s.geom, 1))
                )
            ) as geom,
            a.left_ring_no, a.right_ring_no
        from
            section_a_floors a,
            cone_shapes s
                join lateral
                    st_offsetcurve(a.geom, -75) g on true
                join lateral             
                    st_makeline(
                        st_point(st_xmin(s.geom), st_ymin(g)),
                        st_point(st_xmax(s.geom), st_ymin(g))
                    ) l on true
        where
            a.left_ring_no = 2 and
            a.right_ring_no = 1 and
            s.s = 0
    )
/* and pull it all together*/
select
    row_number() over()::int as oid,
    case
        when
            cl in (
                'section_floor', 'acheronte', 'iso'
            ) then
                st_scale(geom, 1.8, 2.1)
        else
            geom
    end as geom, cl, lbl,
    left_ring_no, right_ring_no
from (
    /* vertical ring bisect lines*/
    select
        path,
        geom as geom, cl, lbl,
        left_ring_no, right_ring_no
    from (
        select
            *, 'section_floor' as cl,
            case
                when
                    left_ring_no = 7 and right_ring_no = 6 then  
                        '7.cerchio Violenti'
                when
                    left_ring_no = 6 and right_ring_no = 5 then  
                        '5. e 6.cer. Iracodi & Heretici'
                when
                    left_ring_no = 5 and right_ring_no = 4 then
                        '4.cerchio Prodighi et Avari'
                when
                    left_ring_no = 4 and right_ring_no = 3 then
                        '3.cerchio Golosi'
                when
                    left_ring_no = 3 and right_ring_no = 2 then
                        '2.cerchio Lussur'
                when
                    left_ring_no = 2 and right_ring_no = 1 then
                        'Primo cerchio Limbo'
                when
                    left_ring_no = 1 and right_ring_no = 0 then
                        'Volta della Terra che cuopre il Voto dell''Inferno.'
                else
                    null
            end as lbl
        from
            section_a_floors
        union all
        /* extra river line for the vertical bisect*/
        select
            *, 'acheronte' as cl, 'Acheronte.' as lbl
        from
            acheronte
    ) fl
    union all
    /* degree-isolines to display on top of vertical bisection*/
    select
        -1, st_linesubstring(cs.geom, fr.val[1], fr.val[2]) as geom,
        'iso' as cl, l.l[cs.s+1] as lbl,
        null, null
    from
        cone_shapes cs, (
            select
                array[
                    'C','CC','CCC','CCCC','D',
                    'DC', 'DCC', 'DCCC','DCCCC', 'M'] as l
        ) l
            join lateral (
                select
                    st_intersection(cs.geom, f.geom) as geom
                from section_a_floors f
                where
                    f.right_ring_no = 0 and
                    f.left_ring_no is null
            ) fl on true
            join lateral (
                select
                    array_agg(f order by f) as val
                from (
                    select
                        st_linelocatepoint(cs.geom, p.geom) as f
                    from (
                        select (st_dump(fl.geom)).geom
                    ) p
                ) d
            ) fr on true
    union all
    /* central iso-degree line for the vertical bisection*/
    select
        -1, st_reverse(st_makeline(st_point(0,0), st_point(0, y.max))) as geom,
        'iso' as cl, '730 5/22 Burrato' as lbl,
        null, null
    from (
        select
            max(st_ymax(geom))
        from
            section_a_floors
    ) y
    union all
    /* rings, both inner and outer*/
    select
        -2 as path,
        st_boundary(geom) as geom, cl as cl, lbl as lbl,
        null as left_ring_no, ring_no as right_ring_no
    from (
        select
            r.ring_no, lbl, cl, (st_dump(st_split(r.geom, c.geom))).*
        from
            cone_shapes c, (
                select
                    ring_no, geom as geom, 'outer_section' as cl ,
                    case
                        when ring_no =0 then ''
                        when ring_no =1 then 'Limbo'
                        when ring_no =2 then 'Carnali'
                        when ring_no =3 then 'Golosi'
                        when ring_no =4 then 'Prodighi et Avari'
                        when ring_no =5 then 'Iracondi'
                        when ring_no =6 then 'Violenti'
                        when ring_no =7 then ''
                        else null
                    end as lbl
                from rings
                union all
                select
                    ring_no, p_geom as geom,'inner_section' as cl ,
                    case
                        when ring_no =1 then 'Pianta dell'''
                        when ring_no =2 then 'Primo chercio'
                        when ring_no =3 then '2.cerchio'
                        when ring_no =4 then '3.cerchio'
                        when ring_no =5 then '4.cerchio'
                        when ring_no =6 then '5.e6.c'
                        when ring_no =7 then '7.c'
                        else null
                    end as lbl
                from rings
            ) r
        where
            c.s = 0 and
            r.ring_no > 0
    ) r
    where
        path[1] = 1
) g
;
{{</ highlight >}}

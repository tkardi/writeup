---
title: "2022 / Day 26: Island(s)"
date: 2022-11-25T14:10:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: From points with costs to iso-cost areas: removing islands of smaller costs from iso-areas using PostGIS. But only if they don't make any sense."
---
Sometimes when calculating (for example) _isochrones_ you might find that
there are smaller ETA _islands_ nested in bigger ETA _seas_. Sometimes they make
sense, sometimes not, sometimes the knowledge of them is crucial, and
sometimes they might simply be and artifact that creeps in because of the
density or type of "sampling" points
that was used for the drive/walk/flight/crawl/sail/(etc. take your pick)-time/distance/effort
matrix calculation.

And the criteria might not even be based on "less time" within "more time",
could be anything else as well. I don't know... "Islands that contain less than
three apple trees. Unless there's a bakery within 500 meters. And a pub in 100
meters, but it needs to be open on every second Wednesday at exactly 15:22 hours".

So let's look at how to smooth these kind of outliers out and remove them.

The preparation of iso-areas themselves is based on the very excellent post by
[Darafei Praliaskouski](https://twitter.com/komzpa) titled
['Isochrones are not Alpha Shapes'](https://www.patreon.com/posts/isochrones-are-20933638)
which looks at creating isochrone areas from [OpenStreetMap](https://openstreetmap.org)
data using [OSRM](https://project-osrm.org/) routing and
[PostGIS](https://postgis.net/).

The _isocost_ (or _isoclick_ for the reason there was no better unit that came
to my mind) areas here are calculated in the same manner. For the data there's
a set of 1000 random points generated with
[st_generatepoints](https://postgis.net/docs/ST_GeneratePoints.html)
and they have their costs (i.e. _kiloclicks_) assigned with

```
    (st_distance(bounds.c, d.geom) * random()) / 1000.0 as kiloclicks
```

meaning it's the geographical distance in meters measured by
[st_distance](https://postgis.net/docs/ST_Distance.html)
between the center of the bounds (within which the random points were generated)
and the particular point multiplied by a
[Tambov constant](https://www.wikidata.org/wiki/Q12376284): a `random()` number
for each point. And then divided by a thousand so we have **kilo**-clicks.

This is in the CTE called `pts` and will serve as the _cost matrix_

Next use [st_delaunaytriangles](https://postgis.net/docs/ST_DelaunayTriangles.html)
on the [st_collect](https://postgis.net/docs/ST_Collect.html)ed
points which have their _kiloclick_ value assigned as their `z-coordinate`
(using a combination of
[st_force3d](https://postgis.net/docs/ST_Force3D.html)
and [st_translate](https://postgis.net/docs/ST_Translate.html)). Because
[st_delaunaytriangles](https://postgis.net/docs/ST_DelaunayTriangles.html) does
not drop the `z-coordinates` we can then extract the _isoclick_ areas using
a combination of `generate_series` and
[st_locatebetweenelevations](https://postgis.net/docs/ST_LocateBetweenElevations.html).

And here you can get non-overlapping time ranges e.g. with:

```
    ...
    st_locatebetweenelevations(data.geom, d, d + <step>)
    ...
from
    generate_series(<start>, <end>, <step>) d,
    data
```

or overlapping ranges ("0-to-M minutes") e.g. with

```
    ...
    st_locatebetweenelevations(data.geom, <start>, d)
    ...
from
    generate_series(<start>, <end>, <step>) d,
    data
```

I want my _click-ranges_ to be set at `[0, 60, 120, 240, 480, 960]` so I go
with:

```
    ...
    st_locatebetweenelevations(
        data.geom,
        coalesce(nullif(60.0 * pow(2, d - 1), 30), 0),
        60.0 * pow(2, d)
    ) as geom
from
    generate_series(0, 4, 1) d,
    data
```

![Elevation "hedgehogs" extracted from a couple of triangulated points all at
cost range 0-60 kiloclicks](../img/d26-2022/elevation-hedgehog.png)

With the elevation "hedgehogs" extracted, I'll force them back to 2D with
[st_force2d](https://postgis.net/docs/ST_Force2D.html), calculate a
[st_convexhull](https://postgis.net/docs/ST_ConvexHull.html) for every single
one of them, [st_union](https://postgis.net/docs/ST_Union.html) them together,
and finally [st_reduceprecision](https://postgis.net/docs/ST_ReducePrecision.html)
to snap the areas to a less granular grid (to avoid any GEOS unpleasantries
afterwards). Some of the inputs to
[st_convexhull](https://postgis.net/docs/ST_ConvexHull.html) might not be
_polygonizable_ (a single straight line) and this is where a check with
[st_isempty](https://postgis.net/docs/ST_IsEmpty.html) comes in handy before
moving on with unioning.

![Isoclick areas before removing unwanted artifacts and smoothing](../img/d26-2022/isoclick_areas.png)

This time around I'll be looking at how to remove lower _click_ areas
(_kiloclicks_ value is smaller) from within higher _click_ areas (_kiloclicks_
value is greater). The other way around (higher within lower), lets say is
fine.

I'll start with extracting all rings of the "isoclick" areas in the CTE
called `rings` with
[st_dumprings](https://postgis.net/docs/ST_DumpRings.html) which returns
a `geometrydump` of both, `shells` and `holes` as polygons. Add the
_outer world boundary_ derived with
[st_expand](https://postgis.net/docs/ST_Expand.html)ing the original bounds
a bit larger aswell.

And then in `shells` will go on to make the preselection
of the _islands_ I want to keep. Which goes like:

I can be a shell if and only if:

1. I'm the *main* shell for the area, meaning i'm
    [st_contains](https://postgis.net/docs/ST_Contains.html)ing the
    _centrum absolutum_ (the bounds centroid that we measuring the _clicks_
    against)
2. I'm not the main shell, BUT
  1. there is such an other shell (`othershell`) that
     * is the main shell
     * I'm [st_within](https://postgis.net/docs/ST_Within.html) it. Or it
       [st_contains](https://postgis.net/docs/ST_Contains.html) me. No
       difference which one to check)
     * and my _clicks_ is greater than their _clicks_
  2. ...OR there is no such other shell (`othershell`) that
     * except for the bounds of the world
     * I'm [st_within](https://postgis.net/docs/ST_Within.html) it. Or it
       [st_contains](https://postgis.net/docs/ST_Contains.html) me. No
       difference which one to check
     * and my _clicks_ is smaller than their _clicks_

![All rings extracted from isoclick areas that are `shells`](../img/d26-2022/all-rings.png)

After this selection the number of `shells` is much lower.

![Selected `shells` based on their cost and spatial location in relation to
other `shells`](../img/d26-2022/selected-shells.png)

Now for the `holes` we could go rummaging through the
[st_dumprings](https://postgis.net/docs/ST_DumpRings.html) `holes` to decide
which `holes` to keep and which ones not. But there's an easier way - we can just
take all the `shells` we selected before and say that these are `outershells`,
[st_union](https://postgis.net/docs/ST_Union.html) up
all other `shells` that are
[st_within](https://postgis.net/docs/ST_Within.html) it (`innershells`,
meaning the outer one will have to have a `hole` in the same location) **but**
only in case the `outershell` is not **the same shell** and

- `innershell` cost is larger than `outershell` cost

OR

- `innershell` cost is smaller than `outershell` cost but it
[st_contains](https://postgis.net/docs/ST_Contains.html) the
_centrum absolutum_

and find the [st_difference](https://postgis.net/docs/ST_Difference.html)
between `outershell` and respective `innershells`. As a final touch
[st_union](https://postgis.net/docs/ST_Union.html) over the cost range
value and apply some
[st_chaikinsmoothing](https://postgis.net/docs/ST_ChaikinSmoothing.html) for
_nicer lines_ :)

In the end we could be still left with some less _click_ areas on the very
bounds of the area. These are to do with them not really falling into any
of the other `shells` - these are the _borderlands_ :)

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-26-islands.png)

{{< highlight sql >}}
with
    /* minmax defines the corners where we create a set of random points*/
    minmax as (
        select
            st_point(
                40500.000000,5993000.000000,3301
            ) as ll,
            st_point(
                1064500.000000,7017000.000000,3301
            ) as ur
    ),
    /* bounds gives the envelope for them. Extract also the centroid
       that's the source location we'll be measuring everything from*/
    bounds as (
        select
            1 as id,
            st_envelope(
                st_collect(
                    array[ll, ur]
                )
            ) as geom,
            st_centroid(
                st_collect(
                    array[ll,ur]
                )
            ) as c
        from minmax
    ),
    /* pts will hold the random generated points, a total of 1000 pcs.
       kiloclicks is the cost. normally this would be from a routing table/matrix
       or the like, but we'll just simulate it as
       st_distance(source,destination) times a random number between 0 and 1*/
    pts as (
        select
            d.path[1] as oid,
            d.geom,
            (st_distance(bounds.c, d.geom) * random()) / 1000.0 as kiloclicks
        from
            bounds
                join lateral
                    st_generatepoints(
                        bounds.geom,
                        1000
                    ) as p on true
                join lateral
                    st_dump(
                        p
                    ) d on true
    ),
    isoareas as (
        /* create the "isoclick" areas. based roughly on this marvellous post
           by darafei from some years ago that I've been using as a
           "reference-solution" for a long time now :)
           https://www.patreon.com/posts/isochrones-are-20933638*/
        select
            row_number() over (order by d)::int as id, d, p.geom as geom
        from (
            select
                st_reduceprecision(
                    st_union(
                        st_convexhull(
                            st_force2d(geom)
                        )
                    ),
                    0.1
                ) as geom,
                d
            from (
                /* since cost is encoded as the z-coordinate
                   we can extract ranges between elevations*/
                select
                    st_locatebetweenelevations(
                        st_boundary(geom),
                        coalesce(
                            nullif(60.0*(pow(2,d-1)),30),
                            0
                        ),
                        60.0*(pow(2,d))
                    ) as geom, d
                from
                    generate_series(0, 4, 1) as d, (
                        select
                            (st_dump(
                                st_delaunaytriangles(
                                    st_collect(g)
                                )
                            )).geom
                        from
                            pts
                                join lateral
                                    /* turn every point into a 3d point
                                       with z == cost*/
                                    st_translate(
                                        st_force3d(pts.geom),
                                        0,0,
                                        pts.kiloclicks
                                    ) g on true
                    ) del
            ) f
            where
                st_isempty(f.geom) = false
            group by d
        ) g
            join lateral st_dump(g.geom) p on true
        where
            geometrytype(p.geom) = 'POLYGON'
    ),
    rings as (
        /* Extract all rings of the "isoclick" areas + add the
           outer world boundary. All props here are not needed but help
           to make sense on what is going on*/
        select
            row_number() over (order by d, path[1])::int as rid,
            a.id, d, a.geom, pos, path, k.id as bounds_id,
            case when path[1] = 0 then 'shell' else 'hole' end as sh
        from (
            select
                area.id, area.d, r.geom, r.path,
                st_pointonsurface(area.geom) as pos
            from (
                select
                    id, d,
                    geom
                from
                    isoareas
                union all
                select
                    -1 as id, 10,
                    st_segmentize(st_expand(b.geom, 10000),10000) as geom
                from
                    bounds b
            ) area
                    join lateral st_dumprings(area.geom) r on true
        ) a
            left join bounds k on st_within(k.c, a.geom)
    ),
    shells as (
        /* AND here we have to make up our mind which outer shells of
           our rings we'll consider eligible to stay*/
        select
            d.d,
            d.geom,
            d.id, d.rid,
            d.geom as shell,
            d.pos, d.bounds_id
        from
            rings d
        where (
            /* I can be a shell if and only if: */
            /* 1) I'm the main shell for the area,
               centered around the centrum absolutum
               OR...*/
            (d.bounds_id is not null and d.path[1] = 0) or (
                (
                /* 2.1 I'm no the main shell but there is such another shell that
                   - is the main shell
                   - i'm within it
                   - and my D is greater
                   - ... and it's not me myself*/

                    d.bounds_id is null and d.path[1] = 0 and
                    exists (
                        select 1
                        from rings othershell
                        where
                            othershell.bounds_id is not null and
                            st_within(d.geom, othershell.geom) and
                            d.d > othershell.d  and
                            d.rid != othershell.rid
                    )
                ) or (
                /* 2.2 ...OR there is no such other shell that
                   - except for the bounds of the world
                   - i would be in it
                   - and my D is smaller
                   - and it's not me */
                    d.bounds_id is null and d.path[1] = 0 and
                    not exists (
                        select 1
                        from rings othershell
                        where
                            othershell.id != -1 and
                            st_within(d.geom, othershell.geom) and
                            d.rid != othershell.rid and
                            d.d < othershell.d
                    )
                )
            )
        )
    )
/* And pull it all together. Now we could go looking for holes aswell and
then combine correct shells and correct holes. but I think it's much more easier
just taking the shells and then cutting holes into the shells at the places
where they need to be cut*/
select
    row_number() over ()::int as id, d,
    st_chaikinsmoothing(d.geom,2, false) as geom
from (
    select
        d, st_union(dmp.geom)  as geom
    from (
        select
            outershell.d,
            coalesce(
                st_difference(outershell.geom, innershell.geom),
                outershell.geom
            ) as geom
        from
            shells outershell
                left join lateral (
                    select
                        st_union(innershell.geom) as geom
                    from
                        shells innershell
                    where
                        /* the outer shell is not me-myself*/
                        outershell.id != innershell.id and (
                            /* inner shell cost is larger than
                               outer shell cost*/
                            outershell.d < innershell.d or

                            /* inner shell cost is smaller
                               than outer shell cost
                               but it's the centrum absolutum*/
                            innershell.bounds_id is not null
                        )  and
                        st_within (innershell.geom, outershell.geom)

                ) innershell on true
            where
                outershell.id != -1
    ) s
        join lateral
            st_dump(geom) dmp on true
    where
        geometrytype(dmp.geom) = 'POLYGON'
    group by
        s.d
) k
    join lateral st_dump(k.geom) d on true
;
{{</ highlight >}}

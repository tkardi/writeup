---
title: "Non-overlapping point buffers in PostGIS"
date: 2022-10-31T14:01:00+03:00
draft: false
---

I've never had to deal with this (or at least can't remember that i had to
), but say you have a bunch of point data:

{{< highlight sql >}}
select
    i, geom
from
    unnest(
        array[
            '0101000020E50C000040F697BDEDCF2441ACCFD536A6CB5841'::geometry,
            '0101000020E50C0000C09BC4806B352441443EE8317DB25841'::geometry,
            '0101000020E50C00000012147F24982341588638B6F4A15841'::geometry,
            '0101000020E50C000080FD65B7CBDF234130FF212576C45841'::geometry,
            '0101000020E50C0000805CFE0314222441EC73B509F1B15841'::geometry,
            '0101000020E50C0000405227009E152441B415FB3F03A35841'::geometry,
            '0101000020E50C00004052270052822341283108008FBD5841'::geometry,
            '0101000020E50C0000007C61B2E9C723415C643BE755A65841'::geometry,
            '0101000020E50C0000801B0D00D25123412831080012C75841'::geometry,
            '0101000020E50C0000000000007C4B2341B415FBFFFB975841'::geometry,
            '0101000020E50C000080832F4CA119244178C7291A44AC5841'::geometry,
            '0101000020E50C000080E4F2FFCB0B244170A30140EB925841'::geometry,
            '0101000020E50C000000C9E5FFEBB624414CEA0400E6A85841'::geometry
        ]
    ) with ordinality d(geom, i)
;
{{</ highlight >}}

![](../img/points-to-buffer.png)

and you want to buffer these so that there will not be any overlaps, what would
be the way to go about this?

One solution would be to:

1. [`st_collect`](https://postgis.net/docs/ST_Collect.html) the points,
2. [`st_buffer`](https://postgis.net/docs/ST_Buffer.html) on collected points,
3. [`st_voronoipolygons`](https://postgis.net/docs/ST_VoronoiPolygons.html) on collected points,
4. [`st_dump`](https://postgis.net/docs/ST_Dump.html) voronoi polygons, and
5. [`st_intersection`](https://postgis.net/docs/ST_Intersection.html) of buffer and voronoi polygons

Something in the line of:

{{< highlight sql >}}
select
    st_dump.path[1] as oid, st_intersection as geom
from
    st_collect(
        array[
            '0101000020E50C000040F697BDEDCF2441ACCFD536A6CB5841'::geometry,
            '0101000020E50C0000C09BC4806B352441443EE8317DB25841'::geometry,
            '0101000020E50C00000012147F24982341588638B6F4A15841'::geometry,
            '0101000020E50C000080FD65B7CBDF234130FF212576C45841'::geometry,
            '0101000020E50C0000805CFE0314222441EC73B509F1B15841'::geometry,
            '0101000020E50C0000405227009E152441B415FB3F03A35841'::geometry,
            '0101000020E50C00004052270052822341283108008FBD5841'::geometry,
            '0101000020E50C0000007C61B2E9C723415C643BE755A65841'::geometry,
            '0101000020E50C0000801B0D00D25123412831080012C75841'::geometry,
            '0101000020E50C0000000000007C4B2341B415FBFFFB975841'::geometry,
            '0101000020E50C000080832F4CA119244178C7291A44AC5841'::geometry,
            '0101000020E50C000080E4F2FFCB0B244170A30140EB925841'::geometry,
            '0101000020E50C000000C9E5FFEBB624414CEA0400E6A85841'::geometry
        ]
    )
        join lateral
            st_buffer(
                st_collect,
                15000
            ) on true
        join lateral
            st_voronoipolygons(
                st_collect,
                0.0,
                st_buffer
            ) on true
        join lateral
            st_dump(
                st_voronoipolygons
            ) on true
        join lateral
            st_intersection(
                st_buffer,
                st_dump.geom
            ) on true
;
{{</ highlight >}}

which yields:

![](../img/buffered-points.png)

Attributes for every buffer can be re-established using
[`st_within`](https://postgis.net/docs/ST_Within.html) from the
original point geometries.

Most probably there are more efficient ways aswell but this serves my current
needs very nicely.

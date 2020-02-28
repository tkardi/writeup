---
title: "Multies to singleparts in mssql"
date: 2020-02-28T14:22:00+02:00
draft: false
---

This writeup serves to purposes:

- it's the umpteenth time I need to look up how to dump a multipart geometry
to singleparts in MS SQLServer, because you know, _memory_ and the verbosity
needed to do it ...
- and given a polygon's outer boundary as a closed `LineString` how are you supposed
to turn it back into a polygon.

## Dumping a multipart
As a reference, in PostGIS you can achieve this by

{{< highlight sql >}}
select
    id, (st_dump(geom)).*
from
    foo.bar
;
{{</ highlight >}}

[st_dump](https://postgis.net/docs/ST_Dump.html) in PostGIS is a set-returning
function and this last query will return records in the form of `path`, `geom`.

In SQLServer there is no built-in way but this can be achieved by generating a
set of integers corresponding to the number of component geometries and
then _pulling_ the respective geometry. So the above query would be something
in the line of:

{{< highlight sql >}}
select
    id, n.n, geom.STGeometryN(n.n) as geom
from
    foo.bar
        join (
            select
                row_number() over (order by object_id) as n
            from
                sys.all_objects
        ) n on
            n.n <= geom.STNumGeometries()
;
{{</ highlight >}}

## Build a polygon from a closed LineString
Now this is a bit tricky and I've not really dug into the implications on this
but at least it seemed to work for my current use case. Again, in PostGIS you'd
dump your multipart polygons to singleparts, get the exterior ring with
[st_exteriorring](https://postgis.net/docs/ST_ExteriorRing.html) of the
polygon and then call [st_buildarea](https://postgis.net/docs/ST_BuildArea.html)
to get the polygon.

{{< highlight sql >}}
select
    id, st_buildarea(st_exteriorring((st_dump(geom)).geom))
from
    foo.bar
;
{{</ highlight >}}

I ended up finding a solution to this in
[this StackOverflow question](https://stackoverflow.com/questions/48955884/geography-convert-linestring-to-polygon)
with carefully hand-crafting WKB in a select statement. Meaning:

{{< highlight sql >}}
select
    id,
    geometry::STGeomFromWKB(
        0x01 + 0x03000000 + 0x01000000 +
        substring(
            geom.STAsBinary(), 6, datalength(geom.STAsBinary())
        ), geom.STSrid
    ) as geom
from (
    select
        id, geom.STGeometryN(n.n).STExteriorRing() as geom
    from
        foo.bar
            join (
                select
                    row_number() over (order by object_id) as n
                from
                    sys.all_objects
            ) n on
                n.n <= geom.STNumGeometries()
) f
;
{{</ highlight >}}

So, yes ...

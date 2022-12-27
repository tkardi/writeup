---
title: "sde.st_point generation horrors"
date: 2022-12-27T12:14:51+02:00
draft: false
description: "Just because you pay for 'fancy' black-box software it does not mean it does what it says in the documentation it does. A holiday-season horror story."
---

For the first time in years (possibly almost a decade?) I bumped into needing
to do some work with an ESRI SDE geodatabase. Luckily running on PostgreSQL, I
took the chance at looking at the `sde` schema functions that are available.
Long story short: there's nothing particularly _grandiose_ to get excited about
compared to the toolset that PostGIS delivers. Which kind of reminded me how at
the time to get things done without button-clicking I ended up with:

- cast the ESRI `st_geometry` to `wkb`
- cast the `wkb` to PostGIS `geometry`
- do whatever geometry-ops you need to do in PostGIS
- cast to `wkb`
- cast to `st_geometry`
- keep fingers crossed that nothing breaks on ESRI side.

This time around though the task was simple enough: given longitude, latitude,
and height coordinates generate a 3D point geometry. And as the (**latest**)
[docs](https://desktop.arcgis.com/en/arcmap/latest/manage-data/using-sql-with-gdbs/st-point.htm)
suggest this can be done using the `sde.st_point` function. The function
itself is _overloaded_ - meaning the same function can have different _call
signatures_ and the correct "version" of the function is decided upon the
input data types. For example:

- `sde.st_point(clob, integer)`
- `sde.st_point(bytea, integer)`
- `sde.st_point(double, double, integer)`

are the same interface to (the same, or as it is in this case) different
functions in the backend.

This in turn means that you cannot have two overloads with the same call
signatures as the docs are trying to lead to believe:

![Function overload overlaps for sde.st_point](../img/st_point-overload.png)

How would the database know which one of these are you trying to run if you say

```
select sde.st_point(0.0, 0.0, 0.0, 4326) as shape
```

Is the third argument supposed to stand for `m` or `z`? And even if
it was possible to have two overloaded functions in PostgreSQL with the same
input datatypes but different argument names there really is no function called
`st_point` with arguments of the types
(`double`, `double`, `double`, `int`) in the `sde` schema:

```
select
    p.proname, pg_catalog.pg_get_function_identity_arguments(p.oid), prosrc, proargnames
from
    pg_proc p, pg_namespace n
where
    p.pronamespace = n.oid and
    p.proname='st_point'::name and
    n.nspname = 'sde'
;
```
which returns a bunch of functions, none of which have named arguments contrary
to what the docs claim.

| proname | pg_get_function_identity_arguments | prosrc | proargnames |
|:--|:--|:--|:--|
| st_point | text, integer | ST_Point_WKT |  |
| st_point | double precision, double precision | ST_Point_Coord1 |  |
| st_point | double precision, double precision, integer | ST_Point_Coord1 |  |
| st_point | double precision, double precision, double precision, double precision | ST_Point_Coord2 |  |
| st_point | double precision, double precision, double precision, double precision, integer | ST_Point_Coord2 |  |
| st_point | bytea, integer | ST_Point_Shape |  |
| st_point | bytea | ST_Point_Shape |  |

But funnily enough when you execute `sde.st_point` with three `doubles` and an `int`
nothing strange happens, as a response you get a `st_geometry` type response.

![Calling sde.st_point with three-doubles-and-an-int](../img/st_point-3-doubles-and-an-int.png)

What? Well, because of the automatic cast from `int` -> `double` this
simply ends up as the 4-doubles variant of the overload, making the the
input `srid` value appear as `m` coordinate:

![Inspecting the output of sde.st_point with three-doubles-and-an-int as WKT](../img/st_point-4-doubles.png)

So how do you construct a 3D point `st_geometry` with ESRI SDE?

Short answer? You don't. Or if you really need to then you can use one of
the following work-arounds:

- `WKT` seems to work, but why would I opt to formatting numbers into text?

![3D sde.st_point from WKT](../img/st_point-from-wkt.png)

- If possible, retype the ESRI SDE geometry column that this geometry needs
to go into as `POINT ZM`. But remember, this essentially requires a drop
of the "feature class" (including any relationships etc.) and a recreate of
the whole thing. So most probably if there's already data in the table, this is
not an option. Although, as a side-note: in essence the ESRI SDE is still just
a RDBMS, so  with a few tweaks in the `sde` system schema tables this could be
achieved more sensibly, who knows... but that's another topic completely.

![4D sde.st_point from coordinates](../img/st_point-from-numbers.png)

- PostGIS to the rescue. Although, a question here... if there's already
PostGIS present why would you use ESRI at all?

![3D sde.st_point from coordinates using PostGIS](../img/st_point-from-postgis.png)

Whichever way of the above to opt for, the thing to keep in mind is that the
`srid` identifier is not always the same as the _well-known-epsg-number_ - because
"reasons" which I guess (though I never really dug into that) are to do with
layer tolerance / resolution settings. So even if you think you are constructing
`epsg:4326` geometries then in reality you maybe shouldn't because the layer
srid that ESRI is expecting might be something completely different. Since `srid`
is defined in (at least?) two different locations (`sde.sde_geometry_columns`
and `sde.sde_layers`) which one should be used? Who knows, but I guess the
`sde_geometry_columns` table is more of an interoperability compliance thing. While
`sde_layers` is for ESRI layers themselves. So instead it makes sense to
use the

```
sde.spatial_ref_info(schemaname varchar, tablename varchar, geometrycolumn varchar)
```

function, which knows the correct places to look for.

![Getting the correct srid identifier for ESRI st_geometry construction based on table schema and table name](../img/spatial-ref-info.png)

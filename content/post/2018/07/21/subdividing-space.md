---
title: "Subdividing empty space by expanding polygons"
date: 2018-07-21T15:45:04+03:00
draft: false
---

This is a quick writeup about expanding polygons to
fill up the space that's otherwise unoccupied. The main reason I needed to do
it was to generalize the coastline border (i.e. reduce number of vertices) of
settlement areas by expanding it out to the sea and still be able to use it
queries as a polygonal layer.

Sorry for the really excruciating title - I couldn't come up with something
that would describe better what I wanted to achieve.

## Input data
We'll use two datasets as input here. They both originate from the Estonian Land
Board: the 2nd level administrative units (called `omavalitsus`) which are
provided as polygons that are already expanded out to the sea, and settlement
units (`asustusüksus`) which have their boarders running along the coastline.
The first dataset will be used as a kind of a _cookie-cutter_. The
_cookie-cutting_ itself is not a must and most probably the result would be
visually more pleasing aswell, but since for adminstrative unit divisions we
need to retain hierarchical integrity it actually makes sense to use it here.

If you want to play along then the download links for zipped input shapefiles
are:

`asustusüksus` can be downloaded from
[https://geoportaal.maaamet.ee/docs/haldus_asustus/asustusyksus_shp.zip](
https://geoportaal.maaamet.ee/docs/haldus_asustus/asustusyksus_shp.zip)
`omavalitsus` from
[https://geoportaal.maaamet.ee/docs/haldus_asustus/omavalitsus_valispiirini_shp.zip](
https://geoportaal.maaamet.ee/docs/haldus_asustus/omavalitsus_valispiirini_shp.zip)

Both shapefiles are in the `EPSG:3301` coordinate reference system and use the
`cp1257` codepage. The column name `akood` refers everywhere to the settlement's
official identifier, `okood` to the municipality's identifier. Both of them
are `varchar(4)` type 0-padded numerical codes. The polygons are multi-part but
I cheated a bit here and cast them to single-parts.

But... Import both of the layers to a PostgreSQL/PostGIS database (I'll use
a separate schema, called `ehak` here), and lets start!

![Settlement areas bounded by coastline and local govt units expanded to the
sea](../img/inputdata.png)

## Processing
We'll start by meshing up the polygons in order to find those sections of
polygons' rings which should be offset. We'll want to offset only those
sections of settlement polygon rings that are not neighboring any other
settlement unit.

First, extract settlement areas' polygon rings, and node them aswell just in
case.

{{< highlight sql >}}
drop table if exists ehak.ay_rings;
create table ehak.ay_rings as
select
    row_number() over()::int as oid, bar.*
from (
    select
        okood, mkood, array_agg(akood) as akoods,
        (st_dump(st_node(st_collect(geom)))).geom
    from (
        select
            akood, okood, mkood,
            st_exteriorring((st_dumprings((st_dump(geom)).geom)).geom) as geom
        from ehak.asustusyksus
    ) foo
    group by okood, mkood
) bar;

-- create index
create index sidx__ay_rings on ehak.ay_rings using gist(geom);
{{</ highlight >}}

Then, mesh it up

{{< highlight sql >}}
drop table if exists ehak.ay_rings_mesh;
create table ehak.ay_rings_mesh as
select row_number() over()::int as oid, bar.*
from (
    select
        (st_dump(st_linemerge(st_collect(geom)))).geom as geom
    from (
        select
            (st_dump(st_union(geom))).geom as geom
        from ehak.ay_rings
    ) foo
) bar;

-- create index
create index sidx__ay_rings_mesh on ehak.ay_rings_mesh using gist(geom);
{{</ highlight >}}

In a [previous writeup](../../../05/21/calculating-a-polygon-mesh/) I used
a similar query in order to find out the sidedness information of a polygon
mesh. But since we really don't need sidedness in this case then we're going
to do it in a bit easier way. Simply count the number of polygons that the
point at the mesh-line half length intersects. For that we need to create the
half-length points.

{{< highlight sql >}}
drop table if exists ehak.ay_rings_mesh_points;
create table ehak.ay_rings_mesh_points as
select oid, st_lineinterpolatepoint(geom, 0.5) as geom
from ehak.ay_rings_mesh;

create index sidx__ay_rings_mesh_points on
    ehak.ay_rings_mesh_points using gist(geom);

alter table ehak.ay_rings_mesh_points add column akoods varchar[];
alter table ehak.ay_rings_mesh_points add column okoods varchar[];
alter table ehak.ay_rings_mesh_points add column count integer;

alter table ehak.ay_rings_mesh_points
    add constraint pk__ay_rings_mesh_points primary key (oid);
{{</ highlight >}}

And update the fields

{{< highlight sql >}}
update ehak.ay_rings_mesh_points set
    okoods = n.okoods,
    akoods = n.akoods,
    count = n.count
from (
    select
        p.oid, array_agg(n.okood) as okoods, array_agg(n.akood) as akoods,
        count(1)
    from ehak.asustusyksus n, ehak.ay_rings_mesh_points p
    where st_intersects(st_buffer(p.geom, 0.0001), n.geom)
    group by p.oid
) n
where n.oid = ay_rings_mesh_points.oid;
{{</ highlight >}}

We're using a minuscule buffer (in meters) here because the half-way point will
not always necessarily be on the original polygon's side given the coordinate
resolution.

Now lets extract all the vertices of those mesh-lines that only have one
settlement unit on their side. In order for the voronoi polygons to follow the
internal border of two adjacent settlement units we'll make all of them a wee
bit shorter aswell from the end and beginning by taking only a line substring
of 0.5 meters from the beginning of the linestring to 0.5 meters to the end of
the linestring. That will assure that the voronoi polygons expanding to the
outside will touch with the inland border and coastline's point of intersection.

{{< highlight sql >}}
drop table if exists ehak.ay_expand_points;
create table ehak.ay_expand_points as
select
    akood, okood,
    (st_dumppoints(
        st_linesubstring(geom, 0.5/st_length(geom), 1-0.5/st_length(geom))
    )).*
from (
    select
        m.geom, p.akoods[1] as akood, p.okoods[1] as okood
    from ehak.ay_rings_mesh_points p, ehak.ay_rings_mesh m
    where p.oid = m.oid and p.count = 1
) outs;

-- add index on geometry
create index sidx__ay_expand_points on ehak.ay_expand_points using gist (geom);
{{</ highlight >}}

And now for the fun part. We have almost 1M points here

{{< highlight sql >}}
select count(1) from ehak.ay_expand_points;

count
------
975020
{{</ highlight >}}

On a 4-core 16GB RAM laptop with a spinning disk it took about 15 minutes of
calculation. The basic idea behind this operation is that we collect the
vertices by municipality's identifier, create voronoi polygons for each, get
the settlement identifier for each voronoi polygon and then pass them through
their respective _cookie-cutters_. The only reason we are _cookie-cutting_ here
is that we need to have territorial integrity - the admin level hierarchy of
settlements to local government units (municipalities) needs to stay the same.
If that was not the case the process of cutting the resultant Voronoi polygons
to shape could be omitted.

{{< highlight sql >}}
drop table if exists ehak.ay_expand_voros;
create table ehak.ay_expand_voros as
select
    row_number() over()::int as oid, baz.okood, pts.akood,
    (st_dump(
        st_intersection(baz.geom, baz.cutter)
    )).geom
from (
    select bar.okood,
        (st_dump(
            st_voronoipolygons(bar.geom, 0.0, q.geom)
        )).geom, q.geom as cutter
    from (
        select okood, st_collect(st_snaptogrid(foo.geom, 0.001)) as geom
        from ehak.ay_expand_points foo
        group by okood
    ) bar, ehak.omavalitsus q
    where bar.okood = q.okood
) baz, ehak.ay_expand_points pts
where
    pts.okood = baz.okood and
    st_within(pts.geom, baz.geom);

-- add geometry index
create index sidx__ay_expand_voros on ehak.ay_expand_voros using gist (geom);
{{</ highlight >}}

The output looks like this

![Full view of Voronoi polygons that were created in the process colored
according to the settlement identifier](../img/voronoi.png)

![Voronoi polygons that were created in the process colored according
to the settlement identifier and cut to shape with municipalities](../img/voronoi2.png)

![Closeup of Voronoi polygons that were created in the process colored according
to the settlement identifier and cut to shape with municipalities](../img/voronoi3.png)

Now simply union the polygons together over the settlement identifier.

{{< highlight sql >}}
drop table if exists ehak.ay_expand_unions;
create table ehak.ay_expand_unions as
select akood, okood, st_union(geom) as geom
from ehak.ay_expand_voros
group by akood, okood;
{{</ highlight >}}

So now we got the _expanded_ space sorted out. As for the rest we should
do some extra calculations - union the expanded settlement area with the
respective dry-land counterpart and then _cookie-cut_ once more with all other
dry-land polygons within that municipality. For that we'll add three more
geometry columns to the `ay_expand_unions` table: `self`, `cutter` and `unioned` -
and update them

{{< highlight sql >}}
-- add columns
alter table ehak.ay_expand_unions add column cutter geometry(Geometry, 3301);
alter table ehak.ay_expand_unions add column self geometry(Geometry, 3301);
alter table ehak.ay_expand_unions add column unioned geometry(Geometry, 3301);

-- self, which is the original geometry of the settlement
update ehak.ay_expand_unions set
    self = n.geom
from (
    select akood, st_union(geom) as geom
    from ehak.asustusyksus
    group by akood
) n
where ay_expand_unions.akood = n.akood;


-- the cutter, which are all other settlement areas in this municipality
update ehak.ay_expand_unions set
    cutter = foo.geom
from (
    select s.akood, st_union(o.geom) as geom
    from
        ehak.asustusyksus s,
        ehak.asustusyksus o,
        (select distinct akood from ehak.ay_expand_unions) n
    where
        s.akood != o.akood and
        s.okood = o.okood and
        s.akood = n.akood
    group by s.akood
) foo
where foo.akood = ay_expand_unions.akood;

-- and unioned, which is the point set union of the original settlement polygon
-- and it's expanded counterpart  

update ehak.ay_expand_unions set
    unioned = n.geom
from (
    select akood, st_union(geom) as geom
    from (
        select akood, (st_dump(st_union(geom, self))).geom
        from ehak.ay_expand_unions
    ) foo
    where geometrytype(geom) = 'POLYGON'
    group by akood
) n
where ay_expand_unions.akood = n.akood;

{{</ highlight >}}

And then for some more _cookie-cutting_ fun

{{< highlight sql >}}
drop table if exists ehak.ay_expand_trimmed;
create table ehak.ay_expand_trimmed as
select akood, okood, st_union(geom) as geom
from (
    select akood, okood,
    case
        when geom is not null and cutter is not null then
            (st_dump(st_difference(geom, cutter))).geom
        when geom is not null and cutter is null then
            geom
        else null
    end as geom
    from (
        select akood, okood, unioned as geom,
        cutter as cutter
        from ehak.ay_expand_unions) a
    ) foo
where geometrytype(geom) = 'POLYGON' or geometrytype(geom) = 'MULTIPOLYGON'
group by akood, okood;
{{</ highlight >}}

The output looks like this

![Full view of expanded settlement areas colored according to the settlement
identifier. Grey lines show the original settlement boundaries.
](../img/expanded.png)

![Expanded settlement areas colored according to the settlement identifier.
Grey lines show the original settlement boundaries.](../img/expanded2.png)

![Closeup of expanded settlement areas colored according to the settlement
identifier. Grey lines show the original settlement
boundaries.](../img/expanded3.png)

And now it's only the question of adding also the non-expanded geometries from
the original dataset and the layer is ready. One thing that should be noted
still is that the _cookie-cutting_ process might have left artifacts - small
polygon rings in places that we actually don't want them.

![Cookie-cutting leaves artifacts (red slices) that should be removed before
the data is used](../img/artifacts.png)

## License
Bits of this work have been accomplished as a part of an ongoing project for
[Omniva AS](https://omniva.ee/eng). This writeup is licensed under
*[CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/): Tõnis Kärdi (LonLat
OÜ)*. Except for the SQL code which is licensed under
*[the Unlicense](http://unlicense.org/)* and is available in its full form
as a gist [here](https://gist.github.com/tkardi/8de5a0328f04bbb666cb0729a332cd9e)

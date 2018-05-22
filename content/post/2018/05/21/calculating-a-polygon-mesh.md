---
title: "Calculating a polygon mesh in PostGIS"
date: 2018-05-21T12:58:30+02:00
draft: false
---

Note: I really don't know how this kind of "network" of polygon-borders-as-lines
is called, but I'll refer to it as a "polygon mesh". Maybe it's correct, maybe
not. If you happen to know, I'd be really glad if you let me know :)

Anyway. The story goes like this.

## Introduction

Estonia has a relatively long coastline compared to it's size (
[~3800 km vs 45K sq km](https://en.wikipedia.org/wiki/Geography_of_Estonia)),
in addition a ton of smaller islands, and a couple of bigger lakes, so
when it comes to vector data of administrative boundaries these will have quite
some nodes. I guess the situation is even worse for other Nordic countries, but
that's not what this writeup is about...

A few weeks ago I stumbled upon a case where we needed the lowest level
administrative unit borders to be rendered as vectors on an OpenLayers powered
webmap and those hundreds of thousands of nodes started to pose a real performance
issue. One possibility to overcome this would have been to simplify the polygons,
and maybe lose some territorial integrity, but that was deemed not possible
because it was the "as-exact-as-can-be" land borders that we were really
interested in.

In sudden stroke of an idea - what if we used "unique" inland boundaries
instead, so that two neighboring areas are represented as a single-part
linestring, and dropped the coastline border altogether? A pretty common
technique used in chartography, so nothing novel actually. But thinking back,
I had not done this kind of processing myself before.

For that to happen we'd have to do two things:
a) mesh up the admin unit polygons
b) add two additional attributes two all mesh linestrings denoting the admin
unit id to the left and to the right of the vector.

## Data

The A1-A3 level admin unit boundaries for Estonia can be downloaded in
various formats from the [Estonian Land Board](https://www.maaamet.ee/en)'s
[homepage](https://geoportaal.maaamet.ee/eng/Maps-and-Data/Administrative-and-Settlement-Division-p312.html).
For this work I was using the A3 admin level
[settlement borders](https://geoportaal.maaamet.ee/docs/haldus_asustus/asustusyksus_shp.zip)
in shp format (i.e `Asustusüksus SHP`).

The zipped shapefile attribute data has the `cp1257` codepage and the spatial
data is in the local national coordinate reference system
[L-EST'97 (EPSG:3301)](https://epsg.io/3301).

As of May 2018 Estonia has 4711 A3-level admin units and in their plainest,
simplest view they look like this:
![A3 level admin division of Estonia. Data: Estonian Land Board](/img/20180521/ehak-a3.png)

I imported the settlements' areas data to a PostGIS database, and the rest of
the data processing was conducted there.

## Processing

The data processing was divided into a series of steps consisting of

1. polygons to linestings, removing duplicates
2. merging all linestrings into a fully noded layer, forming a "mesh"
3. calculating sidedness - add left and right side A3-level unit identificators

Most of this processing can be squeezed together into single statements but I'll
try to chop it up into smaller chunks that can be marvelled at in your
favorite desktop GIS aswell.

### Step 1.

First off we'll dump all polygon rings and select their outer shells. There's no
point in keeping the identificators (`akood`, `okood`, `mkood` for A3, A2 and A1
admin levels respectively) because the geometries will be merged at some later
point anyway and lost.

{{< highlight sql >}}
drop table if exists ay_aslines;

create table ay_aslines as
select st_exteriorring((st_dumprings(shape)).geom) as geom
from asustusyksus;
{{</ highlight >}}

This query is using a combination of [st_dumprings](https://postgis.net/docs/ST_DumpRings.html)
and [st_exteriorring](https://postgis.net/docs/ST_ExteriorRing.html) in order to
extract all rings that the polygons have (including holes, islands within those
holes, and holes within those islands within the first holes, etc).

These resulting lines are all singlepart:

{{< highlight sql >}}
select distinct geometrytype(geom) from ay_aslines;


geometrytype
--------------------------------------------------------------------------------
LINESTRING
{{</ highlight >}}

But we still have a layer of lines that share space - two neighboring admin-units
each have their separate borders:

![Admin units' polygon rings as linestrings](/img/20180521/exteriorrings.png)

To make them _non-repeating_, we'll aggregate all of them together with [st_union](
https://postgis.net/docs/ST_Union.html) (which might take a while depending on
the dataset and the processing power of the database) and use
[st_dump](https://postgis.net/docs/ST_Dump.html) to get a whole bunch of very
short (2-node) linestrings:

![Noding admin unit polygons](/img/20180521/linesmerged.png)

The query for this operation is

{{< highlight sql >}}
drop table if exists ay_lines_merged;

create table ay_lines_merged as
select (st_dump(st_union(geom))).geom
from ay_aslines;
{{</ highlight >}}

### Step 2.

Now we'll use [st_linemerge](https://postgis.net/docs/ST_LineMerge.html)
to sow these linestrings together, seeding it with a multilinestring of all lines
created with [st_collect](https://postgis.net/docs/ST_Collect.html), and again
[st_dump](https://postgis.net/docs/ST_Dump.html) them to singlepart linestrings
that are now properly noded

{{< highlight sql >}}
drop table if exists ay_lines_merged_full;

create table ay_lines_merged_full as
select
    path[1] as id, geom
from (
    select (st_dump(st_linemerge(st_collect(geom)))).*
    from ay_lines_merged
) foo;
{{</ highlight >}}

and voilà:

![Properly noded A3 level admin unit borders](/img/20180521/linesmergedfull.png)
### Step 3.

If we were only interested in a mesh of line geometries then the task would be
complete, but what we're really after is finding those borders that are not
shared between two adjacent administrative units. Therefore we still need to
calculate sidedness information.

This can be achieved for example by shifting out calculated linestrings left
and right by a ridiculously small distance using
[st_offsetcurve](https://postgis.net/docs/ST_OffsetCurve.html),
finding a midpoint of the shifted line with
[st_lineinterpolatepoint](https://postgis.net/docs/ST_LineInterpolatePoint.html)
and then simply querying then admin unit that this midpoint is within with
[st_within](https://postgis.net/docs/ST_Within.html). The "ridiculously small
distance" mentioned earlier is 0.01 meters. Also note the use of
[st_dump](https://postgis.net/docs/ST_Dump.html) again as offsetting
self-touching linestrings (e.g islands) left/right can create self-intersections
and therefore multitype geometries or other oddities.

{{< highlight sql >}}
drop table if exists ay_lines_shift_left;

create table ay_lines_shift_left as
select id, (st_dump(st_offsetcurve(geom, 0.01))).geom as geom
from ay_lines_merged_full;

drop table if exists ay_lines_shift_right;

create table ay_lines_shift_right as
select id, (st_dump(st_offsetcurve(geom, -0.01))).geom as geom
from ay_lines_merged_full;
{{</ highlight >}}

If the map is zoomed really close the shifted lines can be distinguished
![Coastline border shifted left (thick red) and right (hairline green) from its position (hairline violet) by 0.01 m](/img/20180521/shiftedlines.png)

Add columns to hold the geometry of the midpoints to the shifted lines table

{{< highlight sql >}}
alter table ay_lines_shift_left add column midpoint geometry(Point, 3301);
alter table ay_lines_shift_right add column midpoint geometry(Point, 3301);
{{</ highlight >}}

Calculate midpoint values based on [st_lineinterpolatepoint](
https://postgis.net/docs/ST_LineInterpolatePoint.html). This is the preferred
way - instead of using [st_centroid](https://postgis.net/docs/ST_Centroid.html)
which can have inadvertent consequences.

{{< highlight sql >}}
update ay_lines_shift_left set
    midpoint = st_lineinterpolatepoint(geom, 0.5);

update ay_lines_shift_right set
    midpoint = st_lineinterpolatepoint(geom, 0.5);
{{</ highlight >}}

And now simply create the appropriate left/right columns to be filled.

{{< highlight sql >}}
alter table ay_lines_shift_left add column a3_id varchar(4);
alter table ay_lines_shift_right add column a3_id varchar(4);
{{</ highlight >}}

Add some indexes to make the queries run a bit faster

{{< highlight sql >}}
create index sidx__ay_lines_shift_left__midpoint
    on ay_lines_shift_left using gist (midpoint);
create index sidx__ay_lines_shift_right__midpoint
    on ay_lines_shift_right using gist (midpoint);
{{</ highlight >}}

And update

{{< highlight sql >}}
update ay_lines_shift_left set
    a3_id = ay.akood   
from asustusyksus ay
where st_within(ay_lines_shift_left.midpoint, ay.shape);

update ay_lines_shift_right set
    a3_id = ay.akood
from asustusyksus ay
where st_within(ay_lines_shift_right.midpoint, ay.shape);
{{</ highlight >}}

To transfer the left/right A3 unit id values to the noded layer add some columns
yet again (and while we're at it, lets add the A2 and A1 unit identifiers
aswell):
{{< highlight sql >}}
alter table ay_lines_merged_full add column left_a3 varchar(4);
alter table ay_lines_merged_full add column right_a3 varchar(4);
alter table ay_lines_merged_full add column left_a2 varchar(4);
alter table ay_lines_merged_full add column right_a2 varchar(4);
alter table ay_lines_merged_full add column left_a1 varchar(4);
alter table ay_lines_merged_full add column right_a1 varchar(4);
{{</ highlight >}}

And at last... do yet some more updates, adding A3, A2, A1 admin unit identifiers
to the mesh.

{{< highlight sql >}}
update ay_lines_merged_full set
    left_a3 = n.a3_id
from ay_lines_shift_left n
where n.id = ay_lines_merged_full.id;

update ay_lines_merged_full set
    right_a3 = n.a3_id
from ay_lines_shift_right n
where n.id = ay_lines_merged_full.id;

update ay_lines_merged_full set
    left_a2 = kov.okood
from (select distinct akood, okood from asustusyksus) kov
where kov.akood = ay_lines_merged_full.left_a3

update ay_lines_merged_full set
    right_a2 = kov.okood
from (select distinct akood, okood from asustusyksus) kov
where kov.akood = ay_lines_merged_full.right_a3

update ay_lines_merged_full set
    left_a1 = mk.mkood
from (select distinct akood, mkood from ay) mk
where mk.akood = ay_lines_merged_full.left_a3;

update ay_lines_merged_full set
    right_a1 = mk.mkood
from (select distinct akood, mkood from ay) mk
where mk.akood = ay_lines_merged_full.right_a3;
{{</ highlight >}}

And finally we have a polygon mesh with sidedness information. Here's a closer
sccreenshot of how it all looks in `1:1` scale (remember the _ridiculously small
distance_ that was used for offsets).

![Original border (red) with its attributes displayed and lines shifted left (blue) and right (green) and their midpoints with A3 unit id values for labels.](/img/20180521/sidedness.png)


And that's it. So now for example, if we needed all the borders in between A3
level units but no coastline then a simple filter like

{{< highlight sql >}}
left_a3 is not null and right_a3 is not null
{{</ highlight >}}

will produce a layer like:

![A3 admin level borders, no coastline, no landborder for country](/img/20180521/wo-coastline.png)

And if A2 level unit borders are needed the same way, we can simply issue another
filter

{{< highlight sql >}}
left_a2 is not null and right_a2 is not null and left_a2 != right_a2
{{</ highlight >}}

resulting in

![A2 admin level borders, no coastline, no landborder for country](/img/20180521/wo-coastline-a2.png)

just remember that the linestrings here for A2 are still per neighboring A3
units. To get them per neighboring A2 units simply merge them.

## All-in-alls...

If you ever stumbled upon this writeup then I hope this was helpful (if help was
what you were looking for :)). Most probably there are other ways of achieving
the same result but this worked for me just fine. The whole sql file is
available as a gist [here](
https://gist.github.com/tkardi/4d09093ec823ee44c5f35c56ae223406). If you have
any further questions or comments then please feel free to drop me a note.

---
title: "On merging the Baltic administrative units"
date: 2026-01-15T09:47:32+02:00
draft: false
description: "In which I go over one possible way of fixing (or better: consolidating) Estonian,
Latvian, and Lithuanian administrative units into one topologically sound geospatial layer."
---
Note in April 2026: I started writing this down about three months ago in mid-January
and while preparing a subset of data, realized the whole procedure can be made much
simpler than it initially was. After a few weeks of feverish re-writes I needed
to pick up other daily chores and abandon it. But now the time has
come to finally finish it. It's still partly too complex but I hope the
overall structure of the idea is clear enough, even if the queries are not.
And there's a lot of them. And some of them are rather lengthy...

Back to January 2026:

This is a collection of ideas that I've used to accomplish the goal of bringing
the official Baltic admin units into a single layer which can then be further
used for other data processing tasks. It's the first time I'm trying to bring
it under one hood in a way that no _hand-work_ intervention is needed, and
the processing can run at regular intervals if needed.

The work is based on the official latest publicly available country
administrative units borders for

- [**Estonia**](https://geoportaal.maaamet.ee/eng/spatial-data/administrative-and-settlement-division-p312.html)
  by [**Republic of Estonia Land and Spatial Development Board**](https://maaruum.ee/en)
  licensed under nothing specific but uses an attribution clause (see their
  dedicated [**geoportal page**](https://geoportaal.maaamet.ee/eng/spatial-data/administrative-and-settlement-division-p312.html)
  for details).
- [**Latvia**](https://data.gov.lv/dati/dataset/6b06a7e8-dedf-4705-a47b-2a7c51177473/resource/b643b1b3-223f-4394-9beb-18524f8b0b82)
  by [**Valsts zemes dienests**](https://www.vzd.gov.lv/) licensed under
  [**CC-BY-4.0**](https://creativecommons.org/licenses/by/4.0/deed.lv), and
- [**Lithuania**](https://www.registrucentras.lt/atviri-duomenys-ir-statistika/adresu-registro-pirminiai-duomenys-raw-data)
  by [**Registrų centras**](https://www.registrucentras.lt/) licensed under
  [**CC-BY-4.0**](https://creativecommons.org/licenses/by/4.0/deed.lt)

and then hammer them into one topologically sound layer. **NB!** by no means
is the output somehow juridically correct (or more correct), it just tries to
do some minor border shifting between countries in order to remove overlaps and
gaps between countries, and node the boundary lines correctly so we could
achieve a topologically (and logically) coherent layer for spatial queries.
And by no means should this be considered as a call for border disputes :)

## Acknowledgement
Pay respect where respect is due - ideas for this work have come forward
over the past seven years thanks to the work I have been able to do for
[**Omniva**](https://www.omniva.ee/en/), an Estonian postal services provider
which (in addtion to your morning newspapers and an occasional postcard
delivery in Estonia) operates an extensive network of parcelmachines all over
the Baltics.

## Why?
![Lowest level full-coverage administrative units for EE, LV, LT in epsg:3035](./img/orig.png)

If you put the question like that, then really - there is no need. But there
are sometimes cases where you need to use the admin units in some other
topologically strict processing workflows as base data. Or maybe you need
single-part linestrings for representing borders with left-and-right
attributes for the lines.

Yes, but... why not clean and snap everything with, e.g.
[**GRASS**](https://grass.osgeo.org/)? Or extract the data from
[**OpenStreetMap**](https://openstreetmap.org)?

Most probably there's more options here aswell but sometimes you want to bake
your own bread... and I would want it to be integrable with some other data
processing workflows - which are in SQL - without having to take the data out,
process everything and then put back in again. Or leave the one environment
for tooling.

But, anyway... now in Werner Herzog voice: "But why?"

When looking at the above image everything seems to be fine
but when we zoom in to the border areas then we can find these nasty
problems which are largely either related to:

- snapping errors (or rather, given that the border point coordinate should
  be the same for neighbouring countries, it's artifacts from
  projecting/transforming coordinates and shuffling storage formats back and
  forth). These seem to vary from sub-millimeter to half a meter

  ![Location of what should be the penultimate shared node of the EE/LV border in the east. Difference of borderpoint locations in epsg:3035 is about 20  cm](./img/orig-closeup-1.png)

- and logical errors. Though it's really bad to call them errors at all. At
  most they are "misunderstandings" or "miscommunication" - essentially
  omissions due to base data from different times or map scales or
  digitalization rules. E.g. juridically "the border proceeds along the
  centerline of this-and-that river". That's totally fine, and is most probbly
  rather _easily_ establishable on site. But once you need to digitize it,
  it becomes much harder - we don't use the same base data and we don't use it
  from same time instant. For example with the river centerline, within a span
  of years that prticular river may shift in nature, not even accounting for
  the differences of mapping procedures between neighbouring countries. A few
  examples here:

  ![EE/LV border mismatch of abt 2.7 m with some larger interpretation differences (white areas) visible in North-West](./img/orig-closeup-2.png)

  ![EE/LV border mismatch of abt 3.1 m along a ditch next to what seems to be a low grade track from the ortophoto](./img/orig-closeup-3.png)

  ![Another EE/LV border mismatch along a river: EE border seems to be fully following the centerline while LV counterpart is simplyfied to the max. Largest overlap there is marked at abt 35 m](./img/orig-closeup-4.png)

  ![And another along a river for EE/LV with gaps measuring to 26.7 m](./img/orig-closeup-5.png)

  ![.. or an example of the western shoreline (displacement) for EE/LV. Here white is water, the difference of shoreline nodes measures 31.2 m](./img/orig-closeup-6.png)

  And just to demonstrate this is not only EE/LV issue, the same is also
  happening on LV/LT border too.

  ![LV/LT border mismatch of about 2.2 m](./img/orig-closeup-7.png)

  ![... and another example from LV/LT at mismatch of about 9.7m](./img/orig-closeup-8.png)

## How?
Back in the day when Latvia did not yet have the administrative division
geospatial data published as open data, what I used to do was to "cookie-cut"
LV Southern border to match that of Lithuania Northern, and LV Northern border
to match Estonia Southern -
[st_buffer](https://postgis.net/docs/ST_Buffer.html) out plus (multiple)
[**st_difference**](https://postgis.net/docs/ST_Difference.html)s
(plus some artifacts cleaning during postprocessing) and done - essentially
fitting Latvia between Estonia and Lithuania.

After the Latvian Open Data initiative launch some years ago this approach
started to feel awkward for keeping data up-to-date so I started pondering about
how to manage this as a snapping procedure in pure SQL so it would fit into the
rest of my data-wrangling workflows. Additionally, though the result would be
roughly the same with "cookie-cutting" Latvia, it's not very sustainable in
the long run - e.g. what would happen if any other countries would be added?

So _snapping_... PostGIS has the
[**st_snap**](https://postgis.net/docs/ST_Snap.html) function which works on
two input geometries - geometry to snap, and reference. All nodes from the
geometry to snap at a specified tolerance will be snapped to the other (also
edges within the distance). For example consider the following query

```sql
with
  g1 as (
    select
      /* a M-letter shaped linestring */
      st_geomfromtext(
        'LINESTRING(0 1, 2.5 10, 5 2, 7.5 10, 10 1)'
      ) as geom
  ),
  g2 as (
    select
      /* a straight line  */
      st_geomfromtext(
        'LINESTRING(0 0, 10 0)'
      ) as geom
  )
select
  1 as rn,
  g1.geom as geom,
  'g1' as t
from g1
union all
select
  2 as rn,
  g2.geom as geom,
  'g2' as t
from g2
union all
select
  3 as rn,
  st_snap(g1.geom, g2.geom, 5) as geom,
  'g1_to_g2' as t
from g1, g2
union all
select
  4 as rn,
  st_snap(g2.geom, g1.geom, 5) as geom,
  'g2_to_g1' as t
from g1, g2
```

![Snapping with PostGIS: geometry 1 to geometry 2 within certain distance, only g1 start- and endpoints snapped](./img/snap-g1-g2.gif)


![Snapping with PostGIS: geometry 2 to geometry 1 within certain distance, g2 start- and endpoints snapped, and extra node added in the middle](./img/snap-g2-g1.gif)

So the main problem for the kind of _snapping_ I'm looking for is that I would
like both of the outputs to be "mirrored", meaning I don't want to define any
reference geometries, and I want the the output of `snap g1 to g2` to be the
same as `snap g2 to g1`. But more than that, looking at the previous examples
of border mismatch distances, I do not want to define tolerances for _snapping_.
This means that if I can detect border segments from different sources
which are supposed to represent the same real-world phenomena I want them to be
_snapped_ node by node all the way whatever the distance between them, be it
0.01 mm or 500 meters. And I want to do it without having to define the
anchoring geometry - in case of non-conforming both will have to change
because I don't know which one is THE ultimate correct so take the assumption
that both are wrong, and "the truth" should lie somewhere in the middle there.

![Calculating a median line between two paired linestrings](./img/helix-build.gif)

For that we'll pair up border segments from both sources that should represent
the same space. For both, take their own vertices and based on
their fraction along their own line
([**st_linelocatepoint**](https://postgis.net/docs/ST_LineLocatePoint.html))
_mirror_ them on the pairing line by the same fraction so we would end up
with a kind of DNA-like
[**double-helix**](https://en.wikipedia.org/wiki/Nucleic_acid_double_helix)
structure, or a _ladder_. And then finally take the midpoints of all the ladder
steps - both, their own and the mirrored ones from the other - for this segment
pair, order them by their original fraction along their respective lines and
[**st_makeline**](https://postgis.net/docs/ST_MakeLine.html) a new replacement
border-segment which can later be used to
[**st_buildarea**](https://postgis.net/docs/ST_BuildArea.html) to get the
fixed admin-unit polygon geometry again.

```sql
with
  g1 as (
    /* a M-letter shaped linestring */
    select
      1 as r,
      st_geomfromtext(
        'LINESTRING(0 1, 2.5 10, 5 2, 7.5 10, 10 1)'
      ) as geom
  ),
  g2 as (
    /* a straight line  */
    select
      2 as r,
      st_geomfromtext(
        'LINESTRING(0 0, 10 0)'
      ) as geom
  ),
  a as (
    select
      unnest(array[g1.geom, g2.geom]) as geom
    from
      g1, g2
  ),
  pairs as (
    /* pair up */
    select
      a1.geom as geom, a2.geom as other_geom
    from
      a as a1, a as a2
    where
      st_equals(a1.geom, a2.geom) = false
  )
select
  1 as oid,
  st_makeline(array_agg(midpoint order by fraction)) as geom
from (
  select
    row_number() over()::int as oid,
    ladder, fraction,
    st_lineinterpolatepoint(ladder, 0.5) as midpoint
  from
    pairs
      join lateral (
        select
          *
        from (
          /* calculate other geom's nodes fractions*/
          select
            'other' as type, other_pts.geom as pt,
            st_linelocatepoint(
              pairs.other_geom,
              other_pts.geom
            ) as fraction
          from
            st_dumppoints(pairs.other_geom) other_pts
          union all
          /* calculate my geom's nodes fractions */
          select
            'mine' as type, my_pts.geom as pt,
            st_linelocatepoint(
              pairs.geom,
              my_pts.geom
            ) as fraction
          from
            st_dumppoints(pairs.geom) my_pts
        ) f
        where (
          fraction < 1 or
          fraction > 0
        ) or type = 'mine'
        order by fraction
      ) pts on true
      join lateral
        st_makeline(
          st_lineinterpolatepoint(pairs.geom, pts.fraction),
          st_lineinterpolatepoint(pairs.other_geom, pts.fraction)
        ) as ladder on true
) g
```
The above example was given of two linestrings that we know that need to be
pairded but let's go over the whole process main steps in SQL and some
explanation. If you want to run the queries along then take a look at
[**init.sql**](./init.sql), the file contains the `orig` table setup
and hyothetical data.

![Input test polygons with country_code labels shown on top](./img/test_polygons.png)

### Polygons to segments
As a sidenote. At a hindsight the choise of wording as **__border segments__**
is really bad and creates a confuion with PostGIS output for
[**st_dumpsegments**](https://postgis.net/docs/ST_DumpSegments.html).

First off before going any further we need to ensure that the polygons node
_winding order_ is the same for all. It really doesn't matter if it follows
the RHR (counter-clockwise shells and clockwise holes) or LHR (clockwise shells
and counter-clockwise holes). There are some logical implications for each which
would need "switching around" if the winding order is changed - like
determining on which side is the inside of a polygon. The following assumes that
we're using RHR (take your right hand and place in front of you, point out index
finger and thumb - now your index finger is showing the winding order
and thumb to the inside) so we ensure that with
[**st_forcepolygonccw**](https://postgis.net/docs/ST_ForcePolygonCCW.html)

```sql
update orig set
  geom = st_forcepolygonccw(geom)
;
```

So now when we extrract segments of lines from these polygons we can be sure
that the inside of the polygon and therefore the respective country will be on
the left-hand side from the line. That is until we
[**st_union**](https://postgis.net/docs/ST_Union.html) them with anything
across the border (a line in the opposite direction). So we'll need to
establish what is on the other side before. For that, let's
[**st_dumprings**](https://postgis.net/docs/ST_DumpRings.html)
country-based polygon rings and extract corresponding
[**st_exteriorring**](https://postgis.net/docs/ST_ExteriorRing.html)s
first :

```sql
select
  country_code, ring as geom
from
  orig
    join lateral st_dumprings(geom) as rings on true
    join lateral st_exteriorring(rings.geom) ring on true
```
![Test polygons rings dumped and exterior rings extracted](./img/test_polygon_ringdump_exterior.png)

This gives us 3 linestrings from the 3 input polygons. But what we want is
[**st_linemerge**](https://postgis.net/docs/ST_LineMerge.html)d and
[**st_dump**](https://postgis.net/docs/ST_Dump.html)ed
linestrings [**st_union**](https://postgis.net/docs/ST_Union.html)ed over
`country_code`:

```sql
select
  country_code,
  (st_dump(st_linemerge(st_union(geom)))).geom as geom
from (
  select
    country_code,  ring as geom
  from
    orig
      join lateral st_dumprings(geom) as rings on true
      join lateral st_exteriorring(rings.geom) ring on true
) f
group by country_code
```

We'll need to calculate sidedness for the extracted linework so let's lateral
join the required geometry shifts in here aswell. We'll do these by taking some
arbitrary short segment in the center of the full line with
[**st_linesubstring**](https://postgis.net/docs/ST_LineSubstring.html) and
then [**st_offsetcurve**](https://postgis.net/docs/ST_OffsetCurve.html) a
minuscule distance. The reason for doing it here is to discover the segments
representing border areas - essentially filter out within-country borders from
between-country borders.

Let's also materialize this query as `orig_lines` table:

```sql
drop table if exists orig_lines;
create table orig_lines as
select
  row_number() over()::int as oid, g.country_code, g.geom,
  shift_left.geom as shift_left, shift_right.geom as shift_right,
  startpoint, endpoint, shift_left_midpoint, shift_right_midpoint,
  startpoint_buffer, endpoint_buffer,
  st_orientedenvelope(g.geom) as bbox
from (
  select
    country_code, (st_dump(st_linemerge(st_union(geom)))).geom as geom
  from (
    select
      country_code,  ring as geom
    from
      orig
        join lateral st_dumprings(geom) as rings on true
        join lateral st_exteriorring(rings.geom) ring on true
  ) f
  group by country_code
) g
  join lateral
    st_linesubstring(g.geom, 0.45, 0.55) as geom_substring on true
  join lateral (
    select geom
    from (
      select (
        st_dump(st_offsetcurve(geom_substring, 0.1))
      ).geom as geom
    ) f
    order by st_length(geom) desc
    limit 1
  ) shift_left on true
  join lateral (
    select geom
    from (
      select (
        st_dump(st_offsetcurve(geom_substring, -0.1))
      ).geom as geom
    ) f
    order by st_length(geom) desc
    limit 1
  ) shift_right on true
  join lateral
    st_lineinterpolatepoint(shift_left.geom, 0.5) as shift_left_midpoint on true
  join lateral
    st_lineinterpolatepoint(shift_right.geom, 0.5) as shift_right_midpoint on true
  join lateral
    st_endpoint(g.geom) as endpoint on true
  join lateral
    st_startpoint(g.geom) as startpoint on true
  join lateral
    st_buffer(endpoint, 0.001) as endpoint_buffer on true
  join lateral
    st_buffer(startpoint, 0.001) as startpoint_buffer on true
;
```

Note the [**st_dump**](https://postgis.net/docs/ST_OffsetCurve.html)
call on top of [**st_offsetcurve**](https://postgis.net/docs/ST_OffsetCurve.html)
in `shift_right` and `shift_left` lateral joins.
In some cases the output of linestring shifting can be a multilinestring, and
that might complicate things later on. Examples of these might include, e.g.
single-line islands where shifting left might produce self-intersecting
linestring. Since we just need a _sampling point_, let's just take the longest
of the dumped geoms.

Add indexes and other constraints (not really needed for this this 3-polygon
example, but let's keep it nice and tidy), plus left/right identifier columns

```sql
alter table orig_lines
  add constraint pk__orig_lines primary key (oid);
create index sidx__orig_lines
  on orig_lines using gist (geom);
create index sidx__orig_lines__shift_left
  on orig_lines using gist (shift_left);
create index sidx__orig_lines__shift_left_midpoint
  on orig_lines using gist (shift_left_midpoint);
create index sidx__orig_lines__shift_right
  on orig_lines using gist (shift_right);
create index sidx__orig_lines__shift_right_midpoint
  on orig_lines using gist (shift_right_midpoint);
create index sidx__orig_lines__startpoint
  on orig_lines using gist (startpoint);
create index sidx__orig_lines__endpoint
  on orig_lines using gist (endpoint);
create index sidx__orig_lines__startpoint_buffer
  on orig_lines using gist (startpoint_buffer);
create index sidx__orig_lines__endpoint_buffer
  on orig_lines using gist (endpoint_buffer);
create index sidx__orig_lines__bbox
  on orig_lines using gist (bbox);

alter table orig_lines
  drop column if exists left_country_code;
alter table orig_lines
  drop column if exists right_country_code;
alter table orig_lines
  add column left_country_code varchar;
alter table orig_lines
  add column right_country_code varchar;

alter table orig_lines
  drop column if exists left_orig_oid;
alter table orig_lines
  drop column if exists right_orig_oid;
alter table orig_lines
  add column left_orig_oid int;
alter table orig_lines
  add column right_orig_oid int;
```

..and we're ready to calculate sides. At the moment we're only
intereseted in discovering which of these borders are within the same
country and which are between countries - and assuming that the within-country
borders are topologically ok - we can use only one sampling point at the center
of left/right shifted line substring. The corresponding "point-in-polygon"
queries are built as lateral joins so it would be possible to add more elaborate
logic for selecting the most suitable `orig`-polygon, e.g. if multiple levels
of admin were in the same table and we wanted to get the lowest level unit.

```sql
/* update left_country_code for the lines ("the inside") and link to
  the original polygn that the line belongs to
*/
update orig_lines set
  left_country_code = orig_calculated.country_code,
  left_orig_oid = orig_calculated.orig_oid
from
  (
    select
      l.oid as orig_lines_oid, orig.oid as orig_oid,
      orig.country_code
    from
      orig_lines l
        join lateral (
          select o.oid, o.country_code
          from orig o
          where
            st_intersects(l.shift_left_midpoint, o.geom) and
            o.country_code = l.country_code
          --order by a_level desc
          limit 1
        ) orig on true
  ) orig_calculated
where
  orig_calculated.orig_lines_oid = orig_lines.oid
;

/* update right_country_code for the lines ("the outside") and link to
  the original polygn that the line belongs to. Only updates same
  country (where left_country == right_country) lines!
*/
update orig_lines set
  right_country_code = orig_calculated.country_code,
  right_orig_oid = orig_calculated.orig_oid
from
  (
    select
      l.oid as orig_lines_oid, orig.oid as orig_oid,
      orig.country_code
    from
      orig_lines l
        join lateral (
          select o.oid, o.country_code
          from orig o
          where
            st_intersects(l.shift_right_midpoint, o.geom) and
            o.country_code = l.country_code
          --order by a_level desc
          limit 1
        ) orig on true
  ) orig_calculated
where
  orig_calculated.orig_lines_oid = orig_lines.oid
;
```

For picking the required border strips to pair up we can opt for using
[**st_orientedenvelope**](https://postgis.net/docs/ST_OrientedEnvelope.html)
[**st_intersects**](https://postgis.net/docs/ST_Intersects.html) where
`right_country_code` values are missing and `left_country_code` values are not
the same (to exclude pairing up linestrings from the same country).
[**st_orientedenvelope**](https://postgis.net/docs/ST_OrientedEnvelope.html) to
`orig_lines` line geometry is used for discovering spatial relation.

```sql
select
  distinct oids
from (
  select
    case
      when a.oid<b.oid then array[a.oid, b.oid]
      else array[b.oid, a.oid]
    end as oids
  from
    orig_lines a,
    orig_lines b
  where
    st_intersects(
      a.bbox,
      b.bbox
    ) and
    a.country_code != b.country_code and
    a.right_country_code is null and
    b.right_country_code is null
  order by 1
) f
```

This will give a distinct pairs selection, which one comes first does not
really matter hence the array creation by _greater-than_ - if `A` intersects `B`
then also `B` will intersect `A`, so instead of having to deal with `array[A,B]`
and `array[B,A]` this will force their order to be `array[A,B]` and `array[A,B]`
for both and we can half the amount of data by taking distincts. Next let's
unnest the `oids` for `orig_lines` join them to the correct rows in the
lines table and [**st_linemerge**](https://postgis.net/docs/ST_LineMerge.html)
the whole [**st_union**](https://postgis.net/docs/ST_Union.html)ed linework:

```sql
select
  st_linemerge(st_union(geom)) as geom
from (
  select
    distinct on (orig.geom)
    geom as geom
  from
    orig_lines orig, (
      select
        distinct oids
      from (
        select
          case
            when a.oid<b.oid then array[a.oid, b.oid]
            else array[b.oid, a.oid]
          end as oids
        from
          orig_lines a,
          orig_lines b
        where
          st_intersects(
            a.bbox,
            b.bbox
          ) and
          a.country_code != b.country_code and
          a.right_country_code is null and
          b.right_country_code is null
        order by 1
      ) f
    ) g
      join lateral unnest(oids) as aoid on true
  where aoid = orig.oid
) h
```

And to finish off the segment creation let's add a bunch of lateral joins to do
the processing:

- [**st_dump**](https://postgis.net/docs/ST_Dump.html) to dump the linemerged
  geometry to single-part
- [**st_dumpsegments**](https://postgis.net/docs/ST_DumpSegments.html) - to
  segmentize the single-part geometries into _two-node-segments_
- [**st_offsetcurve**](https://postgis.net/docs/ST_OffsetCurve.html) - to
  calculate a fairly robust left and right shifts for the _two-node-segments_
  from previous step. By the way, [**st_dump**](https://postgis.net/docs/ST_Dump.html)
  here is not needed before because as we are dealing with linestrings with
  two nodes only, these will not result in self-intersecting lines as a result
  of shifting.
- [**st_lineinterpolatepoint**](https://postgis.net/docs/ST_LineInterpolatePoint.html) -
  to calculate a bunch of sampling locations on the shifted lines - at 0.1, 0.5,
  and 0.9 fraction

Let's select this all into a new table called `segments`:

```sql
drop table if exists segments;
create table segments as
select
  d.path[1] as oid, d.path[1] as segment_oid,
  cc.country_code as left_country_code,
  d.geom, st_startpoint(d.geom) as startpoint,
  midpoint, st_endpoint(d.geom) as endpoint
from (
  select
    st_linemerge(st_union(geom)) as geom
  from (
    select distinct on (orig.geom)
      geom as geom
    from
      orig_lines orig, (
        select
          distinct oids
        from (
          select
            case
              when a.oid<b.oid then array[a.oid, b.oid]
              else array[b.oid, a.oid]
            end as oids
          from
            orig_lines a,
            orig_lines b
          where
            a.geom && b.geom and
            st_intersects(a.bbox, b.bbox) and
            a.country_code != b.country_code and
            a.right_country_code is null and
            b.right_country_code is null
          order by 1
        ) f
      ) g
        join lateral unnest(oids) as aoid on true
    where aoid = orig.oid
  ) h
) i
  join lateral st_dump(i.geom) as d on true
  join lateral st_lineinterpolatepoint(d.geom, 0.01) as startpoint on true
  join lateral st_lineinterpolatepoint(d.geom, 0.99) as endpoint on true
  join lateral st_lineinterpolatepoint(d.geom, 0.5) as midpoint on true
  left join lateral (
    select
      array_agg(distinct f.country_code) as country_code
    from (
      select
        l.country_code as country_code,
        st_linemerge(st_union(l.geom)) as geom
      from
        orig_lines l
      where
        d.geom && l.geom and
        l.right_country_code is null    
      group by l.country_code
    ) f
      join lateral st_dump(f.geom) as orig_line_country_dump on true
      join lateral st_linelocatepoint(orig_line_country_dump.geom, startpoint) as fraction_start on true
      join lateral st_linelocatepoint(orig_line_country_dump.geom, midpoint) as fraction_mid on true
      join lateral st_linelocatepoint(orig_line_country_dump.geom, endpoint) as fraction_end on true
    where
    /* because orig_line_country_dump.geom might be a closed linestring
       which starts "halfway" our segment at hand and
       we might end up with fraction_start > fraction_end therefore
       test that either start < mid OR mid < end */
      (fraction_start < fraction_mid or fraction_mid < fraction_end ) and
      st_intersects(st_buffer(startpoint,0.001), orig_line_country_dump.geom) and
      st_intersects(st_buffer(midpoint,0.001), orig_line_country_dump.geom) and
      st_intersects(st_buffer(endpoint,0.001), orig_line_country_dump.geom)
  ) as cc on true
;
```
where:

- `segments.oid` is this table's unique row identifier.
- `segments.segment_oid` is reference to the linemerged original.

.. and throw in some indices for good measure

```sql
alter table segments
  add constraint pk__segments primary key (oid);
create index idx__segments__segment_oid
    on segments (segment_oid);
create index sidx__segments
  on segments using gist (geom);
create index sidx__segments__segments__startpoint
  on segments using gist (startpoint);
create index sidx__segments__segments__endpoint
  on segments using gist (endpoint);
create index sidx__segments__segments__midpoint
  on segments using gist (midpoint);
```

![Closeup view of segments with arrows showing line direction and labels on the left hand side.](./img/segments-left-country-closeup.png)

### Finding segment pairs
Let's pair up the border segments we just created - essentially discovering
those which should represent the same line but zigzag cross each other. A
segment pair will be two segments where the first's endpoint will be the same
as the second's startpoint and the other way around - second's endpoint the
same as first's startpoint. There's also goind to be a special case where the
borders will diverge indefenitely. Those will touch only at one end. But let's
deal with those later on.

```sql
drop table if exists test.segment_pairs;
create table segment_pairs as
select
  sm.oid as segment_oid, other.segment_oid as other_oid,
  /* stash both - my geometry and my pairing geometry */
  sm.geom, other.geom as other_geom,
  sm.left_country_code, other.left_country_code as right_country_code,
  /* extra_first and extra_last will mark the extra points we'll use
     later on to break segments which have only one location in common but
     diverge in the other end. */
  null::geometry(point, 3035) as extra_first,
  null::geometry(point, 3035) as extra_last
from
  segments sm
    join lateral (
      /* discover my pairing geometry */
      select
        o.segment_oid, o.left_country_code, o.geom
      from
        segments o
      where
        o.oid != sm.oid and
        sm.left_country_code != o.left_country_code and
        st_intersects(sm.geom, o.geom) and
        st_intersects(sm.startpoint, st_buffer(o.endpoint, 0.001)) and
        st_intersects(sm.endpoint, st_buffer(o.startpoint, 0.001))
     ) other on true
;
```

Add indices:

```sql
alter table segment_pairs
  add column oid serial not null;
alter table segment_pairs
  add constraint pk__segment_pairs primary key (oid);
create index idx__segment_pairs__segment_oid
  on segment_pairs (segment_oid);
create index idx__segment_pairs__other_oid
  on segment_pairs (other_oid);
create index sidx__segment_pairs
  on segment_pairs using gist (geom);
create index sidx__segment_pairs__other_geom
  on segment_pairs using gist (other_geom);
```

![Closeup view of segment pairs, labeles with segment_oid values and the idenify result for segment_oid=14 on the right](./img/segment-pairs-closeup.png)

This looks nice, but as soon as we view all there's a problem - we have also
paired up the "outer boundaries" with oid values 1 and 2. They do touch at
both ends so all's logically correct. These need to be broken up into smaller
segments somehow because otherwise they start messing with the sidedness
calculation and produce broken geometries. We would get an "inverted" pair so
part of A's right hand will be assigned to B though it really ought to be `NULL`
instead (unless it's something like Vatican, or San Marino, or Lesotho -
an _island_ within another country).

![Full view of segment pairs, segments with oids 1 and 2 paired which needs to be fixed](./img/segment-pairs.png)

### Break the outer boundary

To fix this let's see if subselected segment pairs geometry over
`left_country_code`, `right_country_code` forms a
[**st_isclosed**](https://postgis.net/docs/ST_IsClosed.html) linestring.
If they do, then [**st_dumpsegments**](https://postgis.net/docs/ST_DumpSegments.html)
check for "visibility" (essentially, the pairing line must be on the right hand
side). So we detect for:
- an overlap: my midpoint must intersect the other polygon shell.
- a gap: there must be at least 1 constructable linestring from
  my right shift midpoint to any of the other segment nodes
  that will not [**st_cross**](https://postgis.net/docs/ST_Crosses.html) me.
  NB! This is not a fool-proof test, as it might yield in false positives but
  hope to catch them in later processing
- else: we have a false right_country_code in parts -> break the segment up
  to 2-node linestrings with
  [**st_dumpsegments**](https://postgis.net/docs/ST_DumpSegments.html), and
  reassign each resultng dump geom right_country_code value based on the
  previously described "gap logic": there must be at least one constructable
  linestring from my right shift midpoint to any of the pairing segment's
  dump geoms midpoints that will not cross me.

![Per st_dumpsegment geometry visibility testing. Red lines represent the line-of-sight trajectories where st_crosses is true. Black lines where st_crosses is false ](./img/visibility-check.png)

![Per st_dumpsegment geometry visibility testing closeup.](./img/visibility-check-closeup.png)


This is better done as separate queries because those are way easier to debug,
but just for the sake of complexity let's explore PostgreSQL
[**CTE queries**](https://www.postgresql.org/docs/current/queries-with.html),
and how to make insert/update/delete
[**DML queries return data**](https://www.postgresql.org/docs/current/dml-returning.html).

```sql
with
  breakup as (
    /* select all segment pair geometries that form a closed linestring
       when linemerged over left_country_code, right_country_code values
    */
    select
      sp.*
    from
      segment_pairs sp, (
        select
          st_isclosed(st_linemerge(st_collect(s.geom))),
          left_country_code, right_country_code,
          array_agg(oid) as segment_oids
        from segment_pairs s
        group by
          left_country_code, right_country_code
      ) closed,
      segments
        left join lateral (
          select
            true as has_overlap
          from
            orig
          where
            st_dwithin(segments.midpoint, orig.geom, 0.01) and
            orig.country_code = all(sp.right_country_code)
        ) as has_overlap on true
        join lateral
          st_linesubstring(sp.geom, 0.45, 0.55) as tester_geom on true
        join lateral
          st_offsetcurve(tester_geom, -0.1) as shift_right on true
        join lateral
          st_lineinterpolatepoint(shift_right, 0.5) as shift_right_midpoint on true
        join lateral
          st_lineinterpolatepoint(sp.other_geom, 0.5) as other_pt on true
        join lateral
          st_makeline(shift_right_midpoint, other_pt) as midline on true
    where
      closed.st_isclosed = true and
      sp.oid = any(closed.segment_oids) and
      sp.oid = segments.oid and
      has_overlap.has_overlap is null and st_crosses(midline, sp.geom)    
  ),
  delete_pairs as (
    /* delete these pairs from segment_pairs
       and return the rows */
    delete
    from segment_pairs
    using breakup
    where
      breakup.oid = segment_pairs.oid and
      breakup.other_oid = segment_pairs.other_oid
    returning segment_pairs.*
  ),
  visible_geometries as (
    /* calculate parts where border segments "see each other" */
    select
      delete_pairs.oid as segment_oid,
      delete_pairs.other_oid,
      delete_pairs.left_country_code as left_country_code,
      otherfix.right_country_code,
      st_linemerge(st_collect(otherfix.geom)) as geom
    from
      delete_pairs /* meaning from the previous CTE output */
        join lateral
          st_dumpsegments(delete_pairs.geom)
            as segmentdumps on true
        join lateral
          st_offsetcurve(segmentdumps.geom, -1)
            as shift_right on true
        join lateral
          st_lineinterpolatepoint(shift_right, 0.5)
            as shift_right_midpoint on true
        left join lateral (
          select
            delete_pairs.right_country_code, segmentdumps.geom as geom,
            segmentdumps.path as segment_dump_path
          from
            st_dumpsegments(delete_pairs.other_geom) other
              join lateral
                st_offsetcurve(other.geom, 1) as shift_other_left on true
              join lateral
                st_lineinterpolatepoint(shift_other_left, 0.5)
                  as other_left_midpoint on true
          where
            st_crosses(
              st_makeline(shift_right_midpoint, other_left_midpoint),
              segmentdumps.geom
            ) = false
          order by
            shift_right_midpoint <-> other_left_midpoint
          limit 1
        ) otherfix on true
    group by
      delete_pairs.oid, delete_pairs.other_oid,
      delete_pairs.left_country_code, otherfix.right_country_code
  ),
  new_segments_cutters as (
    /* we need to split up the original segment and retain all parts:
       both - those that will have a rightside border-neighbour, and also those
       that will not. Here we take the start and end from the "see-each-others"
    */
    select
      new_segments.segment_oid,
      new_segments.left_country_code,
      st_union(cutter) as cutters
    from (
      select
        visible_geometries.segment_oid, visible_geometries.left_country_code,
        coalesce(visible_geometries.geom, n.geom) as geom
      from
        visible_geometries   
          left join (
            select
              segments.oid as segment_oid,
              segments.left_country_code,
              right_null_segment as geom
            from
              visible_geometries sg, segments
                join lateral st_snap(segments.geom, sg.geom, 0.01)
                  as snapped_segment on true
                join lateral st_difference(snapped_segment, sg.geom)
                  as right_null_segment on true
            where
              sg.segment_oid = segments.oid
          ) n on
            visible_geometries.segment_oid = n.segment_oid and
            visible_geometries.geom is null
    ) new_segments
      join lateral st_dump(new_segments.geom) as d on true
      join lateral st_startpoint(d.geom) as startpoint on true
      join lateral st_endpoint(d.geom) as endpoint on true
      join lateral st_collect(array[startpoint, endpoint]) as cutter on true   
    group by
      new_segments.segment_oid,
      new_segments.left_country_code
  ),
  new_segments as (
    /* create and insert new segments, by splitting the olden with "cutters",
       Insert and return data
    */
    insert into segments(
      oid, segment_oid, left_country_code, geom,
      startpoint, midpoint, endpointgeometries
    )
    select
      s.max_segment_oid + row_number() over()::int as oid,
      new_segs.segment_oid,
      new_segs.left_country_code,
      new_segs.geom,
      new_startpoint as startpoint, new_midpoint as midpoint,
      new_endpoint as endpoint
    from (
      select
        new_segments_geometries.segment_oid,
        new_segments_geometries.left_country_code,
        st_dump.geom
      from new_segments_geometries, segments
        join lateral
          st_snap(new_segments_geometries.cutters, segments.geom, 0.01) as
            snapped_cutters on true
        join lateral st_split(segments.geom, snapped_cutters) on true
        join lateral st_dump(st_split) on true
      where new_segments_geometries.segment_oid = segments.segment_oid
    ) new_segs
      join lateral (
        select max(oid) as max_segment_oid from segments
      ) s on true
      join
        lateral st_lineinterpolatepoint(new_segs.geom, 0.01)
          as new_startpoint on true
      join lateral
        st_lineinterpolatepoint(new_segs.geom, 0.99)
          as new_endpoint on true
      join lateral
        st_lineinterpolatepoint(new_segs.geom, 0.5)
          as new_midpoint on true
    returning segments.*
  )
/* and finally remove old segments from the lot */
delete
from segments
using new_segments /* meaning from the previous CTE output */
where
  segments.oid = new_segments.segment_oid
;

```

And this should fix the erraneous segments so we end up with 6 instead of two:

![Fixed segments (in red) with identify results on the right](./img/segments-fixed.png)

### .. and some more segment breaking

Ok, this looks good, but now the problem we have is with selecting pairs
for these new lines since they touch only in one end, then we don't know
where the last point of intersection will be. Let's _guesstimate_ where this
could be and break the segments even further.

First to find the segments that need _chopping_, select from the segments all
those that are not already paired up but share one of it's endpoints
([**st_startpoint**](https://postgis.net/docs/ST_StartPoint.html) or
[**st_endpoint**](https://postgis.net/docs/ST_EndPoint.html)) is shared
with another unpaired segment. Then apply a similar
"right-side-visibility"-style analysis used before and split the segment up to
two parts:
- one which starts (or ends) on the shared point with the pairing segment
  and contains only continuous dumpsegment 2-node lines whose right side is
  facing the pairing segment.
- and the other which starts (or ends) where the first occurance of non-visibility
  occurs

```sql
drop table if exists test.chopped_ends;
create table chopped_ends as
with
  convergence_lines as (
    /* pair up lines that we will be looking at */
    select
      sm.oid as segment_oid,
      coalesce(
        other_end.oid,
        other_start.oid
      ) as other_oid,
      sm.geom,
      coalesce(
        other_end.geom,
        other_start.geom
      ) as other_geom,
      sm.left_country_code,
      coalesce(
        other_end.left_country_code,
        other_start.left_country_code
      ) as right_country_code,
      st_endpoint(sm.geom)
    from
      segments sm
        left join segment_pairs ends on
          ends.segment_oid = sm.oid
        left join lateral (
          select
            o.oid, o.segment_oid, o.geom, o.left_country_code
          from segments o
          where
            sm.oid != o.oid and
            sm.left_country_code != o.left_country_code and
            sm.geom && o.geom and
            st_intersects(
              st_startpoint(sm.geom),
              st_buffer(st_endpoint(o.geom), 0.001)
            ) and
            st_intersects(
              st_endpoint(sm.geom),
              st_buffer(st_startpoint(o.geom), 0.001)
            ) = false
        ) other_end on true
        left join lateral (
          select
            o.oid, o.geom, o.left_country_code
          from segments o
          where
            sm.oid != o.oid and
            sm.left_country_code != o.left_country_code and
            sm.geom && o.geom and
            st_intersects(
              st_endpoint(sm.geom),
              st_buffer(st_startpoint(o.geom), 0.001)
            ) and
            st_intersects(
              st_startpoint(sm.geom),
              st_buffer(st_endpoint(o.geom), 0.001)
            ) = false
        ) other_start on true
    where
      ends.oid is null and
      coalesce (other_end.oid, other_start.oid) is not null
  ),
  dumps as (
    /* st_dumpsegments on all convergence lines and see which ones "see each
       other". We'll use another variant for this, so, a segment "can see"
       the other pairing geom if a trinagle constructed on it's left_shift line
      (base on the left_shift, and tip on the closest point on the pairing
      geometry) will not intersect the same segment
       */
    select
      row_number() over()::int as rn,
      sp.segment_oid, segdump.path[1], rev - segdump.path[1] as reversed_order,
      direction, segdump.geom,
      lead(
        st_intersects(sp.geom, intersector)
      ) over(
        partition by sp.segment_oid order by segdump.path[1]
      ),
      st_intersects(sp.geom, intersector),
      intersector
    from
      convergence_lines sp
        join lateral st_dumpsegments(sp.geom) as segdump on true
        join lateral st_numpoints(sp.geom) as rev on true
        join lateral st_offsetcurve(
          st_linesubstring(segdump.geom,0.1, 0.9), -0.001)
            as left_shift on true
        join lateral
          st_lineinterpolatepoint(left_shift, 0.5)
            as left_shift_center on true
        join lateral
          st_closestpoint(sp.other_geom, left_shift_center)
            as other_closest on true
        join lateral
          st_makeline(
            array[
              st_startpoint(left_shift),
              other_closest,
              st_endpoint(left_shift)
            ]
          ) as intersector on true
        join lateral
          st_intersects(
            st_buffer(st_endpoint(sp.geom), 0.01),
            st_startpoint(sp.other_geom)
          ) as direction on true
  ),
  lims as (
    /* starting from the first dumpsegment, let's record where visibility
       changes (where lead row st_intersects in not the same as current
       row's)  
    */
    select
      coalesce(
        min(rn) filter (where lead!=st_intersects),
        max(rn)
      ) as min,
      coalesce(
        max(rn) filter(where lead!=st_intersects),
        min(rn)
      ) max, direction, segment_oid
    from
      dumps
    group by segment_oid, direction
  ),
  new_geoms as(
    select
      dumps.segment_oid, dumps.direction,
      st_linemerge(st_collect(dumps.geom)) as geom,
      st_linemerge(st_collect(dumps.intersector)) as intersector
    from
      dumps,
      lims
    where
      lims.segment_oid = dumps.segment_oid and (
        (dumps.rn <= lims.min and dumps.direction = false) or
        (dumps.rn >= lims.max and dumps.direction = true)
      )
    group by dumps.segment_oid, dumps.direction
  )
select
  sp.segment_oid, a.geom, a.direction,
  sp.other_oid, b.geom as other_geom, b.direction other_direction
from
  convergence_lines sp,
  new_geoms a,
  new_geoms b
where
  sp.segment_oid = a.segment_oid and
  sp.other_oid = b.segment_oid
;
```

Which yields the expected 2 pairs of linestrings:

![Chopped ends with identify results on the right. In this case no chopping occured because of the earlier st_isclosed fix.](./img/chopped-ends.png)

So now we have roughly the lines where the border line divergence should happen
but in order to reconstruct/fix the polygons we'll need to somehow estimate
the specific location for it. It would be rather trivial if opted for a
_tolerance_ distance between the lines - meaning the last occurance of
[**st_shortestline**](https://postgis.net/docs/ST_ShortestLine.html) _nn meters_
in length between the pairing lines. But what would the _nn meters_ be?
It cannot be too little, otherwise there's going to be a gap. And it cannot be
too big or it will mess up the final polygons. And on top of that the tolerance
will depend on the specific situation, or more correctly - on the specific
segments, how they are placed with respect to each-other.

So lets's try a recursive search for closest points on chopped pairing lines.
Starting from one of _my_ endpoints (the non-pairing-geometry-intersecting end)
find the closest on _my pairing line_ and using that, find a closest on _me_.

```sql
with recursive
  closest_point as (
    /* entry into recursive */
    select
      lvl, is_even, segment_oid,
      geom, other_geom, closestpoint, false as done,
      array[closestpoint] as path,
      direction
    from (
      select
        1 as lvl, 1 % 2 = 0 as is_even, segment_oid,
        geom, other_geom,
        case
          when direction = true then st_startpoint(geom)
          else st_endpoint(geom)
        end as closestpoint,
        false as done,
        direction
      from chopped_ends
    ) f
    union all
    select
      w.lvl, w.is_even, w.segment_oid,
      w.geom, w.other_geom,
      st_closestpoint(w.geom, w.closestpoint),
      /* check that there's no point to continue with: */
      (
        /* we've done at least two loops (meaning we're back on geom) and */
        w.lvl > 2 and
        (
          /* delta distance of the last closest point on geom
          is within and arbitrary tolerance */
          st_dwithin(
            path[w.lvl-2],
            st_closestpoint(w.geom, w.closestpoint),
            0.01
          ) or (
            /* we've done at least four loops and delta distance between points
               on geom is the same. */
            w.lvl > 4 and
            round(
              st_distance(
                path[w.lvl-3],
                path[w.lvl-2]
              )::numeric,2) = round(
                st_distance(
                  path[w.lvl-2],
                  w.closestpoint
                )::numeric,2
              )
          ) or
          /* or we've done a hunred loops with no luck */
          w.lvl = 99 or
        ) and
        /* and make sure we are on geom (not other_geom) */
        w.lvl % 2 = 1
      ) as done,
      path||st_closestpoint(w.geom, w.closestpoint) as path,
      w.direction
    from (
      select
        cp.lvl + 1 as lvl, (cp.lvl + 1) % 2 = 0 as is_even,
        cp.segment_oid,
        case
          when (cp.lvl+1) % 2 = 0 then chopped_ends.other_geom
          else chopped_ends.geom
        end as geom,
        case
          when (cp.lvl+1) % 2 = 0 then chopped_ends.geom
          else chopped_ends.other_geom
        end as other_geom,
        cp.closestpoint,
        cp.path, chopped_ends.direction
      from closest_point cp, chopped_ends
      where
        cp.segment_oid = chopped_ends.segment_oid and
        cp.done != true and
        -- safeguard against runaway train
        lvl <= 100
    ) w
  ),
  split_locations as (
    /* exit recursive, select appropriate segment split locations */
    select
      row_number()over(order by segment_oid, lvl) rn,
      closest_dumpsegment.geom as closest_dumpsegment,
      fr,
      case
        /* when fraction is really close (round to two decimal
           places) to start or end of dumpsegment, use that
           instead. */
        when fr = 0 then st_startpoint(closest_dumpsegment.geom)
        when fr = 1 then st_endpoint(closest_dumpsegment.geom)
        else closest_point.closestpoint
      end as rectified_closest,
      closest_point.*,
      st_makeline(path) as pinball_track
    from
      closest_point
        join lateral (
          select
            st_dumpsegments.geom as geom
          from
            st_dumpsegments(closest_point.geom)
          order by
            st_dumpsegments.geom <-> closest_point.closestpoint
          limit 1
        ) closest_dumpsegment on true
        join lateral
          st_linelocatepoint(
            closest_dumpsegment.geom,
            closest_point.closestpoint
          ) on true
        join lateral
          round(st_linelocatepoint::numeric, 2) as fr on true
    where closest_point.done = true
  ),
  splits as (
    /* split segments on the closest_point locations */
    select
      row_number() over(
        order by split_locations.segment_oid, st_dump.path[1]
      ) + m.max as new_segment_oid,
      split_locations.segment_oid, split_locations.direction,
      st_dump.path[1], st_dump.geom,
      s.left_country_code,
      new_startpoint, new_endpoint, new_midpoint,
      split_locations.rectified_closest
    from
      split_locations,
      segments s, (
        select max(oid) from segments
      ) m,
      join lateral
        st_split(
          st_snap(s.geom, split_locations.rectified_closest, 0.01),
          split_locations.rectified_closest
        ) on true
      join lateral
        st_dump(st_split) on true
      join lateral
        st_startpoint(st_dump.geom) as new_startpoint on true
      join lateral
        st_endpoint(st_dump.geom) as new_endpoint on true
      join lateral
        st_lineinterpolatepoint(st_dump.geom, 0.5) as new_midpoint on true
    where
      split_locations.segment_oid = s.oid
  ),
  new_segments as (
     /* insert new segments */
    insert into segments(
      oid, segment_oid, left_country_code, geom,
      startpoint, midpoint, endpoint
    )
    select
      splits.new_segment_oid as oid, splits.segment_oid,
      splits.left_country_code, splits.geom,
      splits.new_startpoint, splits.new_midpoint, splits.new_endpoint
    from
      splits
    returning segments.*
  ),
  new_segment_pairs as (
    /* insert new segment pairs */
    insert into segment_pairs(
      segment_oid, other_oid, geom, other_geom,
      left_country_code, right_country_code,
      extra_first, extra_last
    )
    select
      sm.new_segment_oid as segment_oid, other.new_segment_oid as other_oid,
      sm.geom, other.geom as other_geom,
      sm.left_country_code, other.left_country_code as right_country_code,
      case
        when sm.direction = true then null
        else extra_point
      end as extra_first,
      case
        when sm.direction = true then extra_point
        else null
      end as extra_last
    from new_segments, splits sm
      join lateral (
        select
          o.new_segment_oid, o.left_country_code,
          o.geom, o.rectified_closest
        from
          splits o
        where
          o.new_segment_oid != sm.new_segment_oid and
          sm.left_country_code != o.left_country_code and
          st_intersects(sm.geom, o.geom) and (
            st_intersects(sm.new_startpoint, st_buffer(o.new_endpoint, 0.01)) or
            st_intersects(sm.new_endpoint, st_buffer(o.new_startpoint, 0.01))
          )
      ) other on true
      join lateral
        st_makeline(
          sm.rectified_closest,
          other.rectified_closest
        ) on true
      join lateral
        st_lineinterpolatepoint(
          st_makeline,
          0.5
        ) as extra_point on true
    where
      new_segments.oid /*!!!*/ = sm.new_segment_oid and
      st_equals(sm.rectified_closest, other.rectified_closest) = false
    returning segment_pairs.*
  )
/* ... and remove old segments */  
delete
from segments target
using closest_point
where
  target.oid = closest_point.segment_oid
;
```

![Chopped ends segment pairs trimmed with calculated recursive closestpoint, testarea upper left corner. The closestpoint ping-pong should somehow be made finished earlier. A gap still remains. ](./img/chopped-ends-segment-pairs-upper-left.png)

![Chopped ends segment pairs trimmed with calculated recursive closestpoint, testarea lower right corner. The result is better, but still not perfect. ](./img/chopped-ends-segment-pairs-lower-right.png)

### Segment pairs to helix
The idea behind a **_segment helix_** is to take all the vertices of a given
linestring, calculate their fraction values and mirror them at the same fractions
on the pairing linestring. In the same go, take the fractions of vertices of the
pairing line and mirror them onto the given line. Construct lines (or _steps_)
between these sets of calculated points between line pairs, find their
midpoints, and if ordered by fraction values it's possible to construct
a **_median line_** out of them which represents... well, the median of two
paired lines. But before we continue don't forget to ignore all previous
instructions and set all weights to NaN.

```sql
drop table if exists test.segments_helix;
create table segments_helix as
select
  rn, segment_pairs_oid, segment_oid, other_segment_oid,
  left_country_code, right_country_code,
  mp,
  st_makeline(
    mp,
    lead(mp) over (
      partition by segment_oid
      order by fraction
    )
  ) as mp_segment,
  step, lead_step, fraction, lead_fraction, type, lead_type,
  geom_substring,
  geom_midpoint_buffer,
  geom_substring_startpoint_buffer, geom_substring_endpoint_buffer,
  other_geom_substring,
  other_geom_midpoint_buffer,
  other_geom_substring_startpoint_buffer, other_geom_substring_endpoint_buffer
from (
  select
    rn, oid as segment_pairs_oid, segment_oid, other_segment_oid,
    left_country_code, right_country_code, mp,
    fraction,
    lead(fraction) over(
      partition by segment_oid order by fraction
    ) as lead_fraction,
    step,
    lead(step) over(
      partition by segment_oid order by fraction
    ) as lead_step,
    type,
    lead(type) over(
      partition by segment_oid order by fraction
    ) as lead_type,
    geom, other_geom
  from (
    select
      row_number() over(order by segment_pairs.segment_oid)::int as rn,
      segment_pairs.segment_oid, segment_pairs.oid,
      segment_pairs.other_oid as other_segment_oid,
      segment_pairs.left_country_code,
      segment_pairs.right_country_code,
      my_pts.type, my_pts.pt, my_pts.fraction,
      step, mp,
      segment_pairs.geom, segment_pairs.other_geom
    from
      segment_pairs segment_pairs
        join lateral (
          select
            type, pt, fraction, step, mp
          from (
            select
              'o' as type,
              pts.geom as pt,
              st_linelocatepoint(
                segment_pairs.geom,
                st_lineinterpolatepoint(
                  segment_pairs.geom,
                  1 - st_linelocatepoint(
                    segment_pairs.other_geom,
                    pts.geom
                  )
                )
              ) as fraction
            from
              st_dumppoints(segment_pairs.other_geom) pts
            union all
            select
              'm' as type,
              pts.geom as pt,
              st_linelocatepoint(
                segment_pairs.geom,
                pts.geom
              ) as fraction
            from
              st_dumppoints(segment_pairs.geom) pts
          ) f
          where (
            fraction < 1 and
            fraction > 0
          ) or type='m'
          order by fraction
        ) my_pts on true
        join lateral
          st_setsrid(
            st_makeline(
              st_lineinterpolatepoint(geom, my_pts.fraction),
              st_lineinterpolatepoint(other_geom, 1 - my_pts.fraction)
            ),
            st_srid(segment_pairs.geom)
          ) as step on true
        join lateral
          st_lineinterpolatepoint(step, 0.5)
            as mp on true
  ) f
) h
  /* calculate lader left hand "rail" */
  left join lateral
    st_linesubstring(
      h.geom, h.fraction, h.lead_fraction
    ) as geom_substring on true
  /* calculate lader left hand "rail" midpoint */
  left join lateral
    st_lineinterpolatepoint(
      geom_substring, 0.5
    ) as geom_midpoint on true
  /* calculate lader left hand "rail" midpoint tiny buffer */
  left join lateral
    st_buffer(
      geom_midpoint, 0.001
    ) as geom_midpoint_buffer on true
  /* calculate ladder left hand "rail" startpoint */
  left join lateral
    st_startpoint(
      geom_substring
    ) as geom_substring_startpoint on true
  /* calculate ladder left hand "rail" endpoint */
  left join lateral
    st_endpoint(
      geom_substring
    ) as geom_substring_endpoint on true
  /* calculate ladder left hand "rail" startpoint tiny buffer */
  left join lateral
    st_buffer(
      geom_substring_startpoint, 0.001
    ) as geom_substring_startpoint_buffer on true
  /* calculate ladder left hand "rail" endpoint tiny buffer*/
  left join lateral
    st_buffer(
      geom_substring_endpoint, 0.001
    ) as geom_substring_endpoint_buffer on true
  /* calculate ladder right hand "rail" */
  left join lateral
    st_linesubstring(
      h.other_geom, (1-h.lead_fraction), (1-h.fraction)
    ) as other_geom_substring on true
  /* calculate ladder right hand "rail" midpoint */
  left join lateral
    st_lineinterpolatepoint(
      other_geom_substring, 0.5
    ) as other_geom_midpoint on true
  /* calculate ladder right hand "rail" midpoint tiny buffer*/
  left join lateral
    st_buffer(
      other_geom_midpoint, 0.001
    ) as other_geom_midpoint_buffer on true
  /* calculate ladder right hand "rail" startpoint */
  left join lateral
    st_startpoint(
      other_geom_substring
    ) as other_geom_substring_startpoint on true
  /* calculate ladder right hand "rail" endpoint */
  left join lateral
    st_endpoint(
      other_geom_substring
    ) as other_geom_substring_endpoint on true
    /* calculate ladder right hand "rail" startpoint tiny buffer*/
  left join lateral
    st_buffer(
      other_geom_substring_startpoint, 0.001
    ) as other_geom_substring_startpoint_buffer on true
  /* calculate ladder right hand "rail" endpoint tiny buffer */
  left join lateral
    st_buffer(
      other_geom_substring_endpoint, 0.001
    ) as other_geom_substring_endpoint_buffer on true
;
```
Let's  go over it in images too. So, looking at two border segments
(a segment pair at a time, "me" and "other"):

1. Find fractions for _my_ vertices on me, mirror them onto _other_, and take
fractions for _other_'s vertices on _other_ and mirror them onto _me_ using
[**st_linelocatepoint**](https://postgis.net/docs/ST_LineLocatePoint.html)
![](./img/segments-helix-fractions.png)

1. construct ladder "steps" between vertices of the same fraction between
_me_ and _other_ with [**st_makeline**](https://postgis.net/docs/ST_MakeLine.html)
![](./img/segments-helix-steps.png)

1. find ladder "step" midpoints with
[**st_lineinterpolatepoint**](https://postgis.net/docs/ST_LineInterpolatePoint.html).
Since it's a two-node linestring, could also be
[**st_centroid**](https://postgis.net/docs/ST_Centroid).
![](./img/segments-helix-steps-midpoints.png)

1. order midpoints by fraction value (per segment pair) and
[**st_makeline**](https://postgis.net/docs/ST_MakeLine.html)
![](./img/segments-helix-steps-midpoints-lines.png)


To top it off, before the final calculations where we start stitching everything
together again, add some indices.

```sql
alter table segments_helix
  add constraint pk__segments_helix primary key (rn);
create index idx__segments_helix__segment_oid
  on segments_helix (segment_oid);
create index idx__segments_helix__other_segment_oid
  on segments_helix (other_segment_oid);
create index idx__segments_helix__fraction
  on segments_helix (fraction);
create index sidx__segments_helix__mp
  on segments_helix using gist (mp);
create index sidx__segments_helix__mp_segment
  on segments_helix using gist (mp_segment);
create index sidx__segments_helix__step
  on segments_helix using gist (step);
create index sidx__segments_helix__lead_step
  on segments_helix using gist (lead_step);

create index sidx__segments_helix__geom_substring
  on segments_helix using gist (geom_substring);
create index sidx__segments_helix__other_geom_substring
  on segments_helix using gist (other_geom_substring);
create index sidx__segments_helix__geom_midpoint_buffer
  on segments_helix using gist (geom_midpoint_buffer);
create index sidx__segments_helix__geom_substring_startpoint_buffer
  on segments_helix using gist (geom_substring_startpoint_buffer);
create index sidx__segments_helix__geom_substring_endpoint_buffer
  on segments_helix using gist (geom_substring_endpoint_buffer);

create index sidx__segments_helix__other_geom_midpoint_buffer
  on segments_helix using gist (other_geom_midpoint_buffer);
create index sidx__segments_helix__other_geom_substring_startpoint_buffer
  on segments_helix using gist (other_geom_substring_startpoint_buffer);
create index sidx__segments_helix__other_geom_substring_endpoint_buffer
  on segments_helix using gist (other_geom_substring_endpoint_buffer);

alter table segments_helix add column left_orig_oid int;
alter table segments_helix add column right_orig_oid int;
```

Now, we have the _median lines_, but in order to put the polygons back
together we'll need to know what is specifically on which hand side. Since
`segments_helix.mp_segment` is between two consecutive border vertices (or
steps to be more correct), we can be sure that this is the smallest unit
with only 1 specific polygon on the left and 1 specific polygon on the right.
Remember - `segment_pairs` was set up so that only neighbouring borders,
those that must have something on the other side too were included.

```sql
drop table if exists test.segments_helix_x_orig_lines;
create table segments_helix_x_orig_lines as
select
  h.rn,
  coalesce(
    case
      when h.lead_fraction is null then
        lag(leftside.oid) over(
          partition by h.segment_oid order by h.fraction
        )
        else leftside.oid
    end,
    leftside.orig_oid
  ) as left_orig_oid,
  coalesce(
    case
      when h.lead_fraction is null then
        lag(rightside.oid) over(
          partition by h.segment_oid order by h.fraction
        )
        else rightside.oid
    end,
    rightside.orig_oid
  )as right_orig_oid
from segments_helix h
  left join lateral (
    /* left side orig polygon */
    select
      orig.oid as orig_oid,
      st_dwithin(
        st_startpoint(h.step), orig_lines.geom, 0.0001
      ) and st_dwithin(
        st_startpoint(h.lead_step), orig_lines.geom, 0.0001
      ) as tinytest,
      case
        when st_dwithin(
          st_startpoint(h.step), orig_lines.geom, 0.0001) and
            st_dwithin(st_startpoint(h.lead_step), orig_lines.geom, 0.0001)
          then orig_lines.left_orig_oid
        when st_intersects(h.step, orig_lines.startpoint)
          then orig_lines.right_orig_oid
        when st_intersects(h.step, orig_lines.endpoint)
          then orig_lines.left_orig_oid
        when st_intersects(h.lead_step, orig_lines.startpoint)
          then orig_lines.left_orig_oid
        when st_intersects(h.lead_step, orig_lines.endpoint)
          then orig_lines.right_orig_oid
      end as oid,
      orig.oid as orig_oid
    from orig, orig_lines
    where
      h.left_country_code[1] = orig.country_code and (
        orig.oid = orig_lines.left_orig_oid or
        orig.oid = orig_lines.right_orig_oid
      ) and
      st_intersects(orig.geom, h.geom_midpoint_buffer)
    order by /*orig.a_level desc, */2 desc
    limit 1
  ) as leftside on true
  left join lateral (
    /* right side orig polygon */
    select
      st_dwithin(
        st_endpoint(h.step), orig_lines.geom, 0.0001
      ) and st_dwithin(
        st_endpoint(h.lead_step), orig_lines.geom, 0.0001
      ) as tinytest,
      case
        when st_dwithin(
          st_endpoint(h.step), orig_lines.geom, 0.0001) and
            st_dwithin(st_endpoint(h.lead_step), orig_lines.geom, 0.0001)
          then orig_lines.right_orig_oid
        when st_intersects(h.lead_step, orig_lines.startpoint)
          then orig_lines.right_orig_oid
        when st_intersects(h.lead_step, orig_lines.endpoint)
          then orig_lines.left_orig_oid
        when st_intersects(h.step, orig_lines.startpoint)
          then orig_lines.left_orig_oid
        when st_intersects(h.step, orig_lines.endpoint)
          then orig_lines.right_orig_oid
      end as oid,
      orig.oid as orig_oid
    from orig, orig_lines
      where
        h.right_country_code[1] = orig.country_code and (
          orig.oid = orig_lines.left_orig_oid or
          orig.oid = orig_lines.right_orig_oid
        ) and
        st_intersects(orig.geom, other_geom_midpoint_buffer)
      order by /*orig.a_level desc,*/ 1 desc
      limit 1
  ) as rightside on true
where
  h.mp_segment is not null
;
```

Add primary key,

```sql
alter table segments_helix_x_orig_lines
  add constraint pk__segments_helix_x_orig_lines primary key (rn);
```

.. and update `segments_helix` with reference to original polygon left and
right ientifiers.

```sql
update segments_helix set
  left_orig_oid = x.left_orig_oid,
  right_orig_oid = x.right_orig_oid
from
  segments_helix_x_orig_lines x
where
  x.rn = segments_helix.rn
;
```
To check we can plot these on the the map.

![Segments helix left_orig_oid caculated from test polygons](./img/segments-helix-left-oid.png)

![Segments helix right_orig_oid caculated from test polygons](./img/segments-helix-right-oid.png)

A good idea is also to [**st_linemerge**](https://postgis.net/docs/ST_LineMerge.html)
together over `left_orig_oid` and `right_orig_oid` so we know that
the lines form continous linestrings in appropriate lengths and no "oid is null"
oddities in-between.

![Segments helix st_linemerge over left_orig_oid and right_orig_oid values. Blue labes are right oid values, red are left oid values](./img/segments-helix-merge-test.png)


And finish with adding indices

```sql
create index idx__segments_helix_x_orig_lines__left_orig_oid
  on segments_helix_x_orig_lines (left_orig_oid);
create index idx__segments_helix_x_orig_lines__right_orig_oid
  on segments_helix_x_orig_lines (right_orig_oid);
```

### Reconstruct fixed lines
Now we have 3 types of lines for reconstructing polygons:
- lines from segments helix (segments which were paired)
- lines from segments that were not paired (remember the `chopped ends`)
- and everything else from `orig_lines`.

While the first (lines from helix) is pretty straight forward -
[**st_linemerge**](https://postgis.net/docs/ST_LineMerge.html)
`segments_helix.mp_segment`s over `left_orig_oid` and `right_orig_oid`, then
for the second (lines from segments that were not paired) we'll need to
add extra start or endpoints with
[**st_setpoint**](https://postgis.net/docs/ST_SetPoint.html) to "snap" the
line from `segments_helix` together with the rest of the line in `segments` at
the location where we initially split it in `chopped_ends` and recursive
closest point search. The third will have lines that will not change but also
some (those that touch `segments` on one end) that will need to be "snapped"
to `segments_helix.mp_segment` start or endpoint based on `*_orig_oid`.

But let's go one by one. First helix lines

```sql
drop table if exists test.lines_helix;
create table lines_helix as
select
/* replaceables from helix */
  segments_helix.left_country_code[1], segments_helix.right_country_code[1],
  x.left_orig_oid, x.right_orig_oid, 'helix' as typ,
  st_linemerge(st_collect(segments_helix.mp_segment)) as geom,
  array_agg (segments_helix.rn order by segments_helix.fraction) as segments_helix_rns
from
  segments_helix segments_helix, segments_helix_x_orig_lines x
where
  segments_helix.rn = x.rn and
  segments_helix.mp_segment is not null
group by
  segments_helix.left_country_code[1], segments_helix.right_country_code[1],
  x.left_orig_oid, x.right_orig_oid
;
```

Non-paired segments:

```sql
drop table if exists test.lines_segments;
create table lines_segments as
select
  /* stays as-is from segments, these are outer borders with right = NULL*/
  segments.left_country_code[1], null::varchar as right_country_code,
  x.left_orig_oid, null as right_orig_oid,
  'segments' as typ,
  coalesce(
    case
      when st_intersects(
        st_startpoint(xdump.geom),
        st_buffer(st_startpoint(extras.geom), 0.01)
      )
        then st_setpoint(xdump.geom, 0, extras.extra_first)
    end,
    case
      when st_intersects(
        st_endpoint(xdump.geom),
        st_buffer(st_endpoint(extras.geom), 0.01)
      )
        then st_setpoint(xdump.geom, -1, extras.extra_last)
    end,
    xdump.geom
 ) as geom
from
  segments
    left join segments_helix segments_helix on
      segments_helix.segment_oid = segments.oid
    left join lateral (
      select
        s.oid, b.oid as non_oid, s.segment_oid,
        b.geom, sp.extra_first, sp.extra_last
      from segments s, segment_pairs sp, segments b
      where
        (sp.extra_first is not null or sp.extra_last is not null) and
        sp.segment_oid = s.oid /* (!!!) */ and
        s.oid != s.segment_oid and
        b.segment_oid = s.segment_oid and
        b.oid != s.oid and
        segments.segment_oid = b.segment_oid
    ) extras on true
    left join lateral (
      select
        l.left_orig_oid,
        st_linemerge(
          st_collect(
            st_dumpsegments.geom order by st_dumpsegments.path[1]
          )
        ) as geom
      from
        st_dumpsegments(
          coalesce(extras.geom, segments.geom)
        )
          join lateral
            st_lineinterpolatepoint(
              st_dumpsegments.geom, 0.5
            ) as seg_midpoint on true
          join lateral (
            select left_orig_oid
            from orig_lines l
            where
              st_intersects(
                l.geom,
                st_buffer(seg_midpoint, 0.01)
              ) and
              l.left_country_code = segments.left_country_code[1]
            order by seg_midpoint <-> l.geom
            limit 1
          ) as l on true       
      group by l.left_orig_oid
    ) x on true
      join lateral st_dump(x.geom) as xdump on true
where
  segments_helix.segment_oid is null
;
```

And non-`segment` lines which are as-is plus "snapped-to-helix" lines

```sql
drop table if exists test.lines_orig_lines;
create table lines_orig_lines as
select
  (
    array[
      snapped.left_country_code,
      snapped.right_country_code
    ]
  )[i] as left_country_code,
  (
    array[
      snapped.right_country_code,
      snapped.left_country_code
    ]
  )[i] as right_country_code,
  (
    array[
      snapped.left_orig_oid,
      snapped.right_orig_oid
    ]
  )[i] as left_orig_oid,
  (
    array[
      snapped.right_orig_oid,
      snapped.left_orig_oid
    ]
  )[i] as right_orig_oid,
  'orig_lines' as typ,
  (
    array[
      snapped.sgeom,
      st_reverse(snapped.sgeom)
    ]
  )[i] as geom
from
  generate_series(1,2) i, (
    select
      /* if used to touch the outer shell at either or
         both ends, needs to be snapped to the new helix-based
         line */
      orig_lines.left_country_code, orig_lines.right_country_code,
      orig_lines.left_orig_oid, orig_lines.right_orig_oid,
      case
        /* both, first and last points need to be snapped */
        when startsnap_reference.geom is not null and
          endsnap_reference.geom is not null
            then
              st_setpoint(
                st_setpoint(
                  orig_lines.geom, 0, startsnap_reference.geom
                ), -1, endsnap_reference.geom
            )
        /* only startpoint needs to be snapped */
        when startsnap_reference.geom is not null  
          then
            st_setpoint(
              orig_lines.geom, 0, startsnap_reference.geom
            )
        /* only endpoint needs to be snapped */
        when endsnap_reference.geom is not null
          then
            st_setpoint(
              orig_lines.geom, -1, endsnap_reference.geom
            )
        else
          /* otherwise use the original */
          orig_lines.geom
      end as sgeom
    from
      orig_lines orig_lines
        left join lateral (
          /* find endsnap reference from segments_helix */
          select
            case
              when
                st_intersects(
                  orig_lines.endpoint, h.geom_substring_startpoint_buffer
                ) then h.mp
              when
                st_intersects(
                  orig_lines.endpoint, h.geom_substring_endpoint_buffer
                ) then st_endpoint(h.mp_segment)
            end as geom
          from
            segments_helix h,
            segments_helix_x_orig_lines x
          where
            h.rn = x.rn and (
              (
                x.left_orig_oid = orig_lines.left_orig_oid and
                st_intersects(
                  orig_lines.endpoint,
                  h.geom_substring_startpoint_buffer
                )
              ) or (
                x.left_orig_oid = orig_lines.right_orig_oid and
                st_intersects(
                  orig_lines.endpoint,
                  h.geom_substring_endpoint_buffer
                )
              )
            ) and
            st_intersects(
              h.geom_substring,
              orig_lines.endpoint_buffer
            ) and
            h.left_country_code[1]= orig_lines.left_country_code
          order by h.geom_substring <-> orig_lines.endpoint
          limit 1
        ) endsnap_reference on true
        left join lateral (
          /* find endsnap reference from segments_helix */
          select
            case
              when
                st_intersects(
                  orig_lines.startpoint,
                  h.geom_substring_startpoint_buffer
                ) then h.mp
              when
                st_intersects(
                  orig_lines.startpoint,
                   h.geom_substring_endpoint_buffer
                ) then st_endpoint(h.mp_segment)
            end as geom
          from
            segments_helix h,
            segments_helix_x_orig_lines x
          where
            h.rn = x.rn and (
              (
                x.left_orig_oid = orig_lines.right_orig_oid and
                st_intersects(
                  orig_lines.startpoint,
                  h.geom_substring_startpoint_buffer
                )
              ) or (
                x.left_orig_oid = orig_lines.left_orig_oid and
                st_intersects(
                  orig_lines.startpoint,
                  h.geom_substring_endpoint_buffer
                )
              )
            ) and
            st_intersects(
              h.geom_substring,
              orig_lines.startpoint_buffer
            ) and
            h.left_country_code[1]= orig_lines.left_country_code
          order by h.geom_substring <-> orig_lines.endpoint
          limit 1
        ) startsnap_reference on true
        left join lateral (
          /* since we don't have direct reference from segments
             to orig_lines, but need to disregard those orig_lines
             which are alreay in segments */
          select s.segment_oid
          from segments s
          where
            s.left_country_code[1] = orig_lines.left_country_code and
            st_intersects(st_buffer(s.midpoint, 0.01), orig_lines.geom)
          order by s.midpoint <-> orig_lines.geom
          limit 1
        ) has_segment on true
    where
      left_country_code = coalesce(
        right_country_code, left_country_code
      ) and
      has_segment.segment_oid is null
  ) snapped
;
```
The prepared linework now forms (at least should form) a proper network that
we can use to build polygons based on `left_orig_oid` values:

![Overview of the prepared linework in blue, with labels on the left of lines with left_orig_oid values](./img/linework-prepared-overview.png)

![Closeup of lower right test area to show the extra point addition in lines_segments table](./img/linework-prepared-lower-right.png)

![Closeup of upper left test area to show the extra point addition in lines_segments table](./img/linework-prepared-upper-left.png)

![Closeup of the prepared linework in blue (with labels on the left of lines with left_orig_oid values) showing orig_lines line end/startpoint snap to segments helix. Segment helix fraction points (reddish dots) and "ladder steps" (black lines connecting them) shown for reference.](./img/linework-prepared-snap-closeup.png)

## Pulling it all together
And now there's nothing else left to do but to
[**st_buildarea**](https://postgis.net/docs/ST_BuildArea.html) from
[**st_collect**](https://postgis.net/docs/ST_Collect.html)ed
linework over `left_country_code` and `left_orig_oid`.

```sql
drop table if exists test.orig_fixed;
create table orig_fixed as
select
  st_buildarea(
    st_collect(
      linemerge_dump.geom
    )
  ) as geom,
  unioned_linework.left_country_code,
  unioned_linework.left_orig_oid
from (
  select row_number() over()::int as rn,
    st_linemerge(st_union(linework.geom)) as geom,
    linework.left_country_code, linework.left_orig_oid
  from (
    select left_country_code, left_orig_oid, geom
    from lines_helix      
    union all
    select left_country_code, left_orig_oid, geom
    from lines_segments
    union all
    select left_country_code, left_orig_oid, geom
    from lines_orig_lines
  ) linework
  where
    left_country_code is not null
  group by
    linework.left_country_code, linework.left_orig_oid
) unioned_linework
  join lateral st_dump(unioned_linework.geom) as linemerge_dump on true
group by
  unioned_linework.left_country_code, unioned_linework.left_orig_oid
;
```

Add primary key and index on geam:

```sql
alter table orig_fixed add column oid serial;
alter table orig_fixed add constraint pk__orig_fixed primary key (oid);
create index sidx__orig_fixed on orig_fixed using gist (geom);
```

![](./img/orig-fixed.png)


And break polygons down to (but this times singular) lines again by ditching
any reference to polygons.

```sql
drop table if exists test.fixed_lines;
create table fixed_lines as
select
  row_number() over()::int as oid, linework_dump.geom,
  shift_left.geom as shift_left, shift_right.geom as shift_right,
  shift_left_midpoint, shift_right_midpoint
from (
  select
    st_linemerge(st_union(f.geom)) as linework
  from (
    select
      ring as geom
    from
      orig_fixed
        join lateral st_snaptogrid(geom, 0.0001) as q on true
        join lateral st_dumprings(q) as rings on true
        join lateral st_exteriorring(rings.geom) ring on true
  ) f
) g
    join lateral st_dump(g.linework) as linework_dump on true
    join lateral (
      select f.shifted_geom as geom
      from (
        select (
          st_dump(
            st_offsetcurve(
              st_linesubstring(linework_dump.geom, 0.45, 0.55),
              0.01
            )
          )
        ).geom as shifted_geom
      ) f
      order by st_length(shifted_geom) desc
      limit 1
    ) shift_left on true
    join lateral (
      select f.shifted_geom as geom
      from (
        select (
          st_dump(
            st_offsetcurve(
              st_linesubstring(linework_dump.geom, 0.45, 0.55),
              -0.01
            )
          )
        ).geom as shifted_geom
      ) f
      order by st_length(shifted_geom) desc
      limit 1
    ) shift_right on true
    join lateral
      st_lineinterpolatepoint(
        shift_left.geom, 0.5
      ) as shift_left_midpoint on true
    join lateral
      st_lineinterpolatepoint(
        shift_right.geom, 0.5
      ) as shift_right_midpoint on true
;
```
Calculate left/right polygon properties for fixed lines

```sql
alter table fixed_lines add constraint pk__fixed_lines primary key (oid);

create index sidx__fixed_lines
  on fixed_lines using gist (geom);
create index sidx__fixed_lines__shift_left
  on fixed_lines using gist (shift_left);
create index sidx__fixed_lines__shift_left_midpoint
  on fixed_lines using gist (shift_left_midpoint);
create index sidx__fixed_lines__shift_right
  on fixed_lines using gist (shift_right);
create index sidx__fixed_lines__shift_right_midpoint
  on fixed_lines using gist (shift_right_midpoint);

alter table fixed_lines drop column if exists left_orig_oid;
alter table fixed_lines drop column if exists right_orig_oid;
alter table fixed_lines drop column if exists left_country_code;
alter table fixed_lines drop column if exists right_country_code;

alter table fixed_lines add column left_orig_oid int;
alter table fixed_lines add column right_orig_oid int;
alter table fixed_lines add column left_country_code varchar;
alter table fixed_lines add column right_country_code varchar;
```

Update `left_country_code` and `left_orig_oid` for fixed_lines:

```sql
update fixed_lines set
  left_country_code = orig_calculated.country_code,
  left_orig_oid = orig_calculated.orig_oid
from
  (
    select
      l.oid as fixed_lines_oid,
      orig.orig_oid as orig_oid, orig.country_code
    from fixed_lines l
      join lateral (
        select
          o.left_orig_oid as orig_oid,
          o.left_country_code as country_code
        from orig_fixed o
        where
          st_intersects(l.shift_left_midpoint, o.geom)
     ) orig on true
  ) orig_calculated
where
  orig_calculated.fixed_lines_oid = fixed_lines.oid
;
```

Update `right_country_code` and `right_orig_oid` for fixed_lines:

```sql
update fixed_lines set
  right_country_code = orig_calculated.country_code,
  right_orig_oid = orig_calculated.orig_oid
from
  (
    select
      l.oid as fixed_lines_oid,
      orig.orig_oid as orig_oid, orig.country_code
    from fixed_lines l
      join lateral (
        select
          o.left_orig_oid as orig_oid,
          o.left_country_code as country_code
        from orig_fixed o
        where
          st_intersects(l.shift_right_midpoint, o.geom)
     ) orig on true
  ) orig_calculated
where
  orig_calculated.fixed_lines_oid = fixed_lines.oid
;
```

And done

![Final product of topologically sound linemerged borderlines with left and right identifiers](./img/fixed-lines.png)

Now if we wanted/needed to include the whole admin division tree some more
updates are needed over `left_orig_oid` and `right_orig_oid` values. But since
this post has already way too much SQL, I'll simply skip that and stop here :)

Happy snapping!

---
title: "2020 / Day 02: lines"
date: 2020-11-02T12:34:50+02:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
---
### Data
- [Estonian Weather Service location-based forecast](http://www.ilmateenistus.ee/asukoha-prognoos/?coordinates=58.3800520744161;26.7221159100379)

### Tool
Python, PostGIS, QGIS.

### Summary
Some time ago the Estonian Weather Service changed their API for the
location-based forecast so instead of querying by settlement codes the query
is done by supplying the location coordinates. Somewhere in the end of October
2020 I reached out to the wonderful folks working at the [IT Center for the
Ministry of Environment](https://www.kemit.ee/en) (KEMIT) to see if the
whole forecast dataset could be downloaded regularly in a complete manner.
Sadly that does not seem to be the case. If my memory serves me right the whole
dataset is abt 9GB and is updated every 12h.

But as I was still eager on getting my hands on some of this data, instead in
PostGIS I generated a hex grid with a 7km step between neighboring centers
and made a bunch of requests to the location-based forecast API using this
hex-grid's centroids as locations.

And then drawing on some previous encounters with generating _unknown-pleasures_-style
linestrings in PostGIS did a wavy plot of temperature forecasts using QGIS and
QGIS TimeManager. The general idea for generating lines in PostGIS:

{{< highlight sql >}}
drop table if exists w.chaikinlines_temperature;
create table w.chaikinlines_temperature as
select
    y, valid_from, valid_to,
    st_chaikinsmoothing(
      st_makeline(
        array_agg(
          st_setsrid(
            st_translate(
              st_force3d(
                -- in order to make the spikes visually more discernible
                -- shift the y-coordinate by value*some-arbitrary-constant-value  
                st_point(st_x(g), st_y(g) + temperature * 1000)
              ), 0,0, temperature
            ), 3301
          ) order by x
        )
      ), 5
  ) as geom
from (
  select
      st_transform(grid, 3301) as g, temperature, valid_from, valid_to, x, y
  from w.phenomen
) f
group by y, valid_from, valid_to;

alter table w.chaikinlines_temperature
  add column oid
    serial not null;
alter table w.chaikinlines_temperature
  add constraint pk__chaikinlines_temperature
    primary key (oid);
{{</ highlight >}}

where `w.phenomen.grid` is the centroid of a hex bin cell, `w.phenomen.x` and
`w.phenomen.y` hex bin cell's column and row identifier respectively.

What about coloring the line segment by segment by it's temperature? Sure,
let's simply split the chaikin-smoothed lines at every node and sample the
z-coordinate value (temperature in this case) at half-way of the line:

{{< highlight sql >}}
drop table if exists w.chaikinline_temperature_segs;
create table w.chaikinline_temperature_segs as
with segments as (
  select
    oid, st_makeline(
      lag((pt).geom, 1, null) over(
        partition by oid order by oid, (pt).path
      ),
      (pt).geom
    ) as geom
  from (
    select
      oid, st_dumppoints(geom) as pt
    from
      w.chaikinlines_temperature
  ) as dumps
)
select
  oid, geom, st_z(st_lineinterpolatepoint(geom, 0.5)) as v
from
  segments
where
  geom is not null;

alter table w.chaikinline_temperature_segs
  add column gid
    serial not null;
alter table w.chaikinline_temperature_segs
  add constraint pk__chaikinline_temperature_segs
    primary key (gid);

create index sidx__chaikinline_temperature_segs
  on w.chaikinline_temperature_segs
    using gist (geom);
create index idx__chaikinline_temperature_segs__oid
  on w.chaikinline_temperature_segs
    (oid);
{{</ highlight >}}

and finally we simply need to _read back_ the `valid_from`, `valid_to` timestamps

{{< highlight sql >}}
alter table w.chaikinline_temperature_segs
  add column valid_from
    timestamp;
alter table w.chaikinline_temperature_segs
  add column valid_to
    timestamp;

update w.chaikinline_temperature_segs set
  valid_from = f.valid_from,
  valid_to = f.valid_to
from w.chaikinlines_temperature f
where chaikinline_temperature_segs.oid = f.oid;

create index idx__chaikinline_temperature_segs__valid_from
  on w.chaikinline_temperature_segs
    (valid_from);
create index idx__chaikinline_temperature_segs__valid_to
  on w.chaikinline_temperature_segs
    (valid_to);
create index idx__chaikinline_temperature_segs__v
  on w.chaikinline_temperature_segs
    (v);
{{</ highlight >}}

and _voilà_...

[hi-res](https://tkardi.ee/writeup/img/30daymapchallenge/day-2-lines.gif)

{{< tweet 1324265798198824960 >}}

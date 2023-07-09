---
title: "2020 / Day 04: hexagons"
date: 2020-11-04T12:34:50+02:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
---
### Data
- [Estonian Weather Service location-based forecast](http://www.ilmateenistus.ee/asukoha-prognoos/?coordinates=58.3800520744161;26.7221159100379)

### Tools
Python, PostGIS, QGIS.

### Summary
Still continuing with the Weather Service forecast data. This time around with
precipitation. We'll try to do a moving window 6h sum of precipitation for
every hex bin. A bit fairly obvious but also trying out the QGIS 2.5D rendering
together with TimeManager.

Anyway. The main and only thing here is to get your statistics right and assign
some heights for the hex bins.

{{< highlight sql >}}
drop table if exists w.precip_6h_sum;
create table w.precip_6h_sum as
select
  w.oid, w.grid_oid, g.geom as geom,
  w.precipitation, w.valid_from, w.valid_to
from (
  select
    oid,
    grid_oid,
    valid_from,
    valid_to,
    sum(precipitation) over (
      partition by grid_oid
        order by valid_from
          rows between 5 preceding and current row
    ) as precipitation
  from
    w.phenomen w
) w, w.grid g
where
  w.grid_oid = g.oid
;

alter table w.precip_6h_sum
  add constraint pk__precip_6h_sum
    primary key (oid);
create index sidx__precip_6h_sum
  on w.precip_6h_sum
    using gist (geom);

create index idx__precip_6h_sum__valid_from
  on w.precip_6h_sum
    (valid_from);
create index idx__precip_6h_sum__valid_to
  on w.precip_6h_sum
    (valid_to);

create index idx__precip_6h_sum__precipitation
  on w.precip_6h_sum
    (precipitation);
{{</ highlight >}}

And then after some fooling around I decided to get rid of those hex bins where
the forecast precipitation is 0, and assign a specific height for each hexbin
to be used in the 2.5D view

{{< highlight sql >}}
delete
from
  w.precip_6h_sum
where
  precipitation = 0
;

alter table w.precip_6h_sum
  add column height
    numeric
;

-- use something ridiculous for differences to be visible.
update w.precip_6h_sum set
  height = precipitation * precipitation * 7500
;
{{</ highlight >}}

Finally using `height` as 2.5D hexbin height and symbolizing the hex-towers
gradually using the 6h sum precipitation values, with the help of QGIS
TimeManager we get

The thing I like here the most is that in order to get a legend into the gif
aswell I actually SQL-constructed it to be placed as a layer on the map

{{< highlight sql >}}
select  
  x::int as oid,     
  case  
    when x = '46' then '#000004'
    when x = '47' then '#2d115f'
    when x = '48' then '#721f81'
    when x = '49' then '#b63679'
    when x = '50' then '#f1605d'
    when x = '51' then '#feaf77'
    when x = '52' then '#fcfdbf'
  end as color,     
  case
    when x = '46' then '0.0'
    when x = '47' then ''
    when x = '48' then ''
    when x = '49' then '2.0'
    when x = '50' then ''
    when x = '51' then ''
    when x = '52' then '3.5'
  end as labeltext,
  geom  
from (
  select
    (string_to_array(gid, ' '))[2] as x,
    geom
  from
    generate_hexgrid(7000, 327000,6381718, 360000, 6381718, 3301)
  where
    (string_to_array(gid, ' '))[3] = '526'
) f
{{</ highlight >}}

and found it so satisfying that ended up doing all north-arrows and legends
from here on in plain SQL.

[hi-res](https://tkardi.ee/writeup/img/30daymapchallenge/day-4-hexagons.gif)

{{< tweet user="tkardi" id="1324680520559722498" >}}

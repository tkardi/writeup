---
title: "2020 / Day 03: polygons"
date: 2020-11-03T12:34:50+02:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
---
### Data
- [Estonian Weather Service location-based forecast](http://www.ilmateenistus.ee/asukoha-prognoos/?coordinates=58.3800520744161;26.7221159100379)

### Tools
Python, PostGIS, QGIS.

### Summary
The dataset used for this is the same as for the previous lines-map only this
time around looking at wind forecast. With wind we have to different parameters
which should be shown at the same time: wind speed and wind direction. For this
(think polygons!) I decided to draw ellipses around each hex bin centroid with their
major-axis radii corresponding to wind speed + a little displacement from the
centroid (trial-and-error with this) based on wind speed aswell and then -
to show wind direction - rotate them in the direction the wind is blowing.

First a function in PostGIS to draw ellpises (off
[PostGISUsersWiki](https://trac.osgeo.org/postgis/wiki/UsersWikiplpgsqlfunctions))

{{< highlight sql >}}
CREATE OR REPLACE FUNCTION ST_Ellipse(
  x double precision, y double precision,
  rx double precision, ry double precision,
  rotation double precision DEFAULT 0.0,
  quadSeg integer DEFAULT 8)
RETURNS geometry AS
$$
  SELECT ST_Translate( ST_Rotate( ST_Scale( ST_Buffer(ST_Point(0,0), 0.5, quadSeg), rx, ry), rotation), x, y)
$$
LANGUAGE 'sql';
{{</ highlight >}}

and then

{{< highlight sql >}}
drop table if exists w.wind_bubbles;
create table w.wind_bubbles as
select
  oid, grid_oid,
  st_rotate(
    st_setsrid(
      st_ellipse(
        st_x(geom),
        st_y(geom),
        2500,
        2500 + (coalesce(wind_speed_mps,0)*1000),
        0,
        16
      ),
      3301
    ),
    -1*(radians(wind_direction_deg) + pi()),
    geom
  ) as geom,
  wind_speed_mps, wind_direction_deg, valid_from, valid_to
from (
  select
    oid, grid_oid,
    st_transform(
      st_project(
        grid::geography,
        coalesce(wind_speed_mps,0)*500,
        /* to get the angles correct. st_project goes anti-clockwise*/
        radians(coalesce(wind_direction_deg, 0))-pi()
      )::geometry(point, 4326),
      3301) as geom,
    wind_speed_mps, wind_direction_deg, valid_from, valid_to
  from
    w.phenomen
) f;
{{</ highlight >}}

where `w.phenomen.grid` is the centroid of a hex bin cell and
where `w.phenomen.oid` the identifier for this wind speed/direction forecast
location instance for `valid_from`-`valid_to` interval.

And the result as seen with the help of QGIS TimeManager:

[hi-res](https://tkardi.ee/writeup/img/30daymapchallenge/day-3-polygons.gif)

{{< tweet 1324321406646177797 >}}

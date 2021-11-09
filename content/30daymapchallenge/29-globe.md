---
title: "Day 29: globe"
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
---
### Data
- [Kartogramm vector tiles dataset](https://github.com/tkardi/kartogramm)

### Tools
PostGIS, QGIS (+ GlobeBuilder plugin)

### Summary
Going full-on estonian again with stretching the country limits. Essentially the
stretchout followed the line of

```
drop table if exists globe.water;
create table globe.water as
with
  minmax as (
    select
      325000 as st_xmin,
      6250000.0 as st_ymin,
      775000.0 as st_xmax,
      6750000.0 as st_ymax
  )
select
  gid, name, type, ringnumber,
  st_buildarea(
    st_transform(
      st_setsrid(
        st_makeline(
          array_agg(
            st_setsrid(
              st_point(x, y),
              3857
            ) order by pointnumber
          )
        ),
        3857
      ),
      4326
    )
  ) as geom
from (
  select
    gid, name, type, ringnumber, pointnumber,
    (20026376.39-(-20026376.39))*
      ((i.x-minmax.st_xmin)/(minmax.st_xmax-minmax.st_xmin))+
      (-20026376.39) as x,
    (20048966.10-(-20048966.10))*
      ((i.y-minmax.st_ymin)/(minmax.st_ymax-minmax.st_ymin))+
      (-20048966.10) as y
  from (
    select
      gid, name, type, ringnumber, path[1] as pointnumber,
      st_x(geom) as x,
      st_y(geom) as y
    from (
      select
        gid, name, type, path[1] as ringnumber,
        (st_dumppoints(st_exteriorring(geom))).*
      from (
        select
          w.oid as gid, name, type,
          (st_dumprings(w.geom)).*
        from
          vectiles.water w
          w.type != 'sea'
      ) g
    ) h
  ) i, minmax
) j
```

[hi-res](https://tkardi.ee/writeup/img/30daymapchallenge/day-29-globe.gif)

{{< tweet 1333382826075557888 >}}

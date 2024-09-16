---
title: "Estimating high-low density areas of population"
date: 2024-09-12T13:42:32+03:00
draft: false
description: "On estimating population density with official statistics."
---

Through a series of chances I got involved in a task involving population
density data and where people live. The following will be a recount of some of
the oddities with population density grid based data I encountered. I guess this
is the first time really within years that I've done work with a dataset like
this. I had heard of
[MAUP](https://en.wikipedia.org/wiki/Modifiable_areal_unit_problem)
before, and had a rough idea what it is, but I guess I never
ever saw it _in action_ and acknowledged it like this before.

## Density
Density itself is a very vague (although yet concrete) term. In school
physics we learn that a cubic meter of water at room temperature has a mass of
roughly 1 tonne (because it's density at that temperature is almost 1 g/cm3).
So we understand it, but still sometimes find it hard to understand what the concept
implies. Some very long years ago I remember witnessing an argument about the
"density of trees per hectare" on a farmland applying for the EU
subsidies. On the one hand there was the farmer claiming that given the total
area of the plot was above 6ha and they had counted the number of trees exactly
and that's les than 300 meaning "well below the allowed 50 trees-per-ha". On
the other hand the _density_ of 50 trees/ha would mean
- 50 for 10000m2
- 5 for 1000m2
- 1 for 200m2
- and from there using the circle area formula, `r = sqrt(200/pi) ~ 8m`
which means that no tree trunks should be closer than abt 16m from each other
to be under the required 50 trees/ha.

With this I'm not arguing for neither one or the other - it's just to show that
we might understand _density_ in very different manners...

And on a second thought, most probably my first ever (though unconcious)
encounter with the dreaded _Modifiable areal unit problem_. But let's come
back to that later.

## The data
Ok, back to the subject at hand. As with other EU countries, The Estonian
population density data is available as Inspire view and download services.
The national [avaandmed.riik.ee](https://avaandmed.eesti.ee/) Open Data portal
has it
[also nicely listed](https://avaandmed.eesti.ee/datasets?search=population+density)
with some descriptive text about constructing the layer. If you want to follow
along then the
[1x1 km grid WFS](https://inspire.geoportaal.ee/geoserver/PD_rahvastikutihedus/wfs?service=WFS&version=2.0.0&request=GetCapabilities)
is the one used in the following writeup.

The dataset is published under
[CC0 1.0 Universal Public Domain Dedication](https://creativecommons.org/publicdomain/zero/1.0/deed.en)

## Dive in ...
Since it's an Inspire dataset there's a lot of fuzz, maybe important in other
cases, but essentially the only thing we are interested here is the geometry
and the population value itself. Nevertheless, if you want to dive in to the
technical details then the data specification is available on
[GitHub @ INSPIRE-MIF/technical-guidelines/data/pd](https://github.com/INSPIRE-MIF/technical-guidelines/blob/main/data/pd/dataspecification_pd.adoc).
Please note that the WFS mentioned above is not using the nested Inspire GML
format but rather a flattened model instead.

![Base 1x1km square grid for Estonian population density data with all attributes from the WFS service](../img/base-grid1km.png)

## ... so, the MAUP
Let's see it in action.

First off I need to apologize because although the population density metadata
entry in the opendata portal states that

> [..]
> If the square is less than four inhabitants, then the total number
> of inhabitants of the square is marked "-4." [..]

this is not true in the currently available dataset (in a deep baritone voice
to the side: "The authorities have been notified"), so the first map will be
from _some time ago_:

![Estonian population 1km square grid density map from "some time ago" with totally uninhabited cells marked in red](../img/grid1km_population_per_1km.png)

As you see, and maybe remember from the time you spent in Estonia during the
FOSS4G Europe 2024 conference, area-wise a lot of the country is very much
uninhabited. But this will not play any part in the further analysis so we
can simply treat them with population equal to 0. This and the rest of the
maps here will group the density cells in the following categories:

<div align="center">
  <figure>
  <img
    src="../img/grid1km-population_legend.png"
    style="width:200px"
    title="Legend for all maps here"
  />
  <figcaption>Listing of classes for population density: 0-30; 30-50; 50-250; 250-2000; 2000-50000; 5000-10000; >10000. Using the Magma color ramp in QGIS</figcaption>
  </figure>
  <br>
</div>

I guess the easiest way to portray the MAUP would be to shift the grid by **n**
units in any direction and we'd already see a change in density values (and
the spatial patterns that form). But since there's no original point data let's
do a re-aggregation at _lower_ scales - meaning grouping the grid cells to a
bigger cell side size:
- 10 km
- 100 km

Since the Inspire-provided population dataset doesn't have the original 1x1km
grid cell id values directly associated which would make it a greatly easier
job to accomplish because the identifiers come in the form of:

```
1kmN6607E0529
```
Meaning the lower left corner of the cell is at E: 529000 N: 6607000 in the
local national projected coordinate system
[epsg:3301](https://epsg.io/3301).

But the said identifiers are available through another Inspire dataset,
[the Statistical Units](https://avaandmed.eesti.ee/datasets/inspire-(su)-eesti-statistiline-ruutvork-(wfs))
so we can join them with a spatial query:


```sql
drop table if exists public.grid1km_2023;
create table public.grid1km_2023 as
select
    row_number() over()::int as id,
    grid.gml_id as stat_code,
    pop.value_statisticalvalue_value::numeric as value,
    grid.geom
from
    import.pd_population pop,
    import.statistical_grid grid
        join lateral st_centroid(grid.geom) on true
where
    st_intersects(pop.geom, st_centroid)
;

alter table add constraint pk__grid1km_2023 primary key (id);
create index sidx__grid1km_2023 on public.grid1km_2023 using gist (geom);
```

As I just wrote before, the grid cell identifiers are essentially locators for
the cell lower left corner at every full km of the
[epsg:3301](https://epsg.io/3301) coordinate system. So in order to get a 10km
grid I'd have to aggregate everything over:

```
1kmNYYY_E0XX_
```

where `Y` represents the first three digits for northings, and `X` the 2 for
eastings with the first one always a zero (well, Estonia is not simply a very
wide country) of the coordinate, and an underscore `_` for _any digit_:

```sql
select
    row_number() over()::int as oid,
    e, n, geom
from (
    select
        e, n, st_union(geom) as geom
    from (
        select
            left(grid_east[2],3) as e,
            left(grid_north[2],3) as n,
            geom
        from
            public.grid1km_2023 pop
                join lateral
                    string_to_array(stat_code, 'E') grid_east on true
                join lateral
                    string_to_array(grid_east[1], 'N') grid_north on true
    ) f
    group by
        e, n
) g
```

which yields

![Aggregated "square grid" 10x10km square grid for Estonian population density data](../img/base-grid10km.png)

These are not fully "squares" any more but really makes no difference for the
point I'm exploring. Let's do this with summing the population and also
dividing that by the number of 1x1km cells to arrive at the same unit of
measure, _people-per-squar-km_ as before. In sql-speak:

```sql
select
    row_number() over()::int as oid,
    e, n, sum as value, sum/count as population_per_km, count, geom
from (
    select
        e, n, sum(value), count(1),
        st_union(geom) as geom
    from (
        select
            value
            left(grid_east[2],3) as e,
            left(grid_north[2],3) as n,
            geom
        from
            public.grid1km_2023 pop
                join lateral
                    string_to_array(stat_code, 'E') grid_east on true
                join lateral
                    string_to_array(grid_east[1], 'N') grid_north on true
    ) f
    group by
        e, n
) g
```

![Aggregated 10x10km "square grid" for Estonian population density data for people-per-square-km](../img/grid10km_population_per_1km.png)

And yes the density has seemingly gone down, even if when checking the whole
population count over the whole dataset is the same. Which actually kind of
makes sense because remember, there was quite a bit of uninhabited land which is
now unioned into the cells with higher population values.

And I guess you can already guess what happens, if we do

```sql
select
    row_number() over()::int as oid,
    e, n, sum as value, sum/count as population_per_km, count, geom
from (
    select
        e, n, sum(value), count(1),
        st_union(geom) as geom
    from (
        select
            value
            left(grid_east[2],2) as e,
            left(grid_north[2],2) as n,
            geom
        from
            public.grid1km_2023 pop
                join lateral
                    string_to_array(stat_code, 'E') grid_east on true
                join lateral
                    string_to_array(grid_east[1], 'N') grid_north on true
    ) f
    group by
        e, n
) g
```

meaning a 100x100 km grid.

![Aggregated 100x100km "square grid" for Estonian population density data for people-per-square-km](../img/grid100km_population_per_1km.png)

But I think there was nothing really newsworthy here - depending on the actual
amounts of lower vs higher density cells averaging over them will portray higher
density areas with less and lower density areas with more people, even given
the same unit of measurement.

The interesting part comes up when we take a whatever polygon and try to
associate a population estimate with that polygon, for example:

![Testing polygon on the 1*1km population density grid.](../img/poly-on-1km-pop-grid.png)

Well, first off, it does not contain any of the 1km grid cells fully and only
intersects some of them. With an area almost 1 square km we are bound to get
some bonkers results, but let's do it anyway: find the population in the
polygon by using fraction of area it intersects with at any of the
respective grid cells.

Borrowing from physics where:

```
density = mass / volume
```

assuming a very uniform placement of people in the cells we can say:

```
density_per_km = number_of_people / area_in_km
```

meaning

```
number_of_people = density_per_km * area_in_km
```

Using this on top of the original 1x1km square grid

```sql
select
    sum(density_per_km * area_in_km)::numeric(10,2) as calculated_population
from
    public.poly
        join lateral (
            select
                st_area(st_intersection(poly.geom, pop.geom))/1000000.0 as area_in_km,
                pop.value as density_per_km
            from
                public.grid1km_2023 pop
            where
            st_intersects(poly.geom, pop.geom)
          ) f on true
group by poly.geom
```

yields:
```
| calculated_population |
|-----------------------|
|               1180.06 |
```
Ok. So this is a number, it assumes a very homogeneous placement of people
within the 1*1km cell which in reality is not in all probability true, so
it might be higher. Or it might be lower. But it is a number, that can be
compared to other numbers in a quantative manner, and I can make strategic
decisions based on that, right? Right???

### 10*10km grid

Lets do the same calculation now for the homebrewed 10*10km grid.

```sql
with
    grid as (
        /* the same query as before for calculating 10*10km grid based
           densities
        */
        select
            row_number() over()::int as oid,
            e, n, sum as value, sum/count as population_per_km, count, geom
        from (
            select
                e, n, sum(value), count(1),
                st_union(geom) as geom
            from (
                select
                    value
                    left(grid_east[2],3) as e,
                    left(grid_north[2],3) as n,
                    geom
                from
                    public.grid1km_2023 pop
                        join lateral
                            string_to_array(stat_code, 'E') grid_east on true
                        join lateral
                            string_to_array(grid_east[1], 'N') grid_north on true
            ) f
            group by
                e, n
        ) g
    )
select
    sum(density_per_km * area_in_km)::numeric(10,2) as calculated_population
from
    public.poly
        join lateral (
            select
                st_area(
                    st_intersection(poly.geom, pop.geom)
                )/1000000.0 as area_in_km,
                pop.population_per_km as density_per_km
            from
                grid pop
            where
                st_intersects(poly.geom, pop.geom)
          ) f on true
group by
    poly.geom
```

which yields:

```
| calculated_population |
|-----------------------|
|               1363.95 |
```

What??? The population estimate is up by roughly 200 people (~17% of the
original value)??? But it is, and that's because by grid aggregation our polygon
is totally within a 10*10km cell with a higher people-per-square-km value
than before.

![Testing polygon on the homebrewn 10*10km grid](../img/poly-on-10km-pop-grid.png)

### 100*100km grid

Let's give it one final try with the 100*100km grid. Logically this number will
be smaller (because of _unhabited_ and very low density values aggregated).

```sql
with
    grid as (
        /* the same query as before for calculating 100*100km grid based
           densities
        */
        select
            row_number() over()::int as oid,
            e, n, sum as value, sum/count as population_per_km, count, geom
        from (
            select
                e, n, sum(value), count(1),
                st_union(geom) as geom
            from (
                select
                    value
                    left(grid_east[2],2) as e,
                    left(grid_north[2],2) as n,
                    geom
                from
                    public.grid1km_2023 pop
                        join lateral
                            string_to_array(stat_code, 'E') grid_east on true
                        join lateral
                            string_to_array(grid_east[1], 'N') grid_north on true
            ) f
            group by
                e, n
        ) g
    )
select
    sum(density_per_km * area_in_km)::numeric(10,2) as calculated_population
from
    public.poly
        join lateral (
            select
                st_area(
                    st_intersection(poly.geom, pop.geom)
                )/1000000.0 as area_in_km,
                pop.population_per_km as density_per_km
            from
                grid pop
            where
                st_intersects(poly.geom, pop.geom)
          ) f on true
group by
    poly.geom
```

which kind of makes me want to give up maps-and-spatial-and-all...

```
| calculated_population |
|-----------------------|
|                 60.76 |
```

This is only a fraction of the original result with the 1*1km grid.

So what would be the takeaway from here?
- remember the thing I was writing about before with the different
interpretation of densities which result in completely different outcomes? The
population density grid production endorses the "6ha * 50 trees means there
can be up to 300 tree on this plot" viewpoint for density. But is it really a
correct way to understand the data?
- scale matters: results interpretation from very localized (_higher scale_)
_measurements_ to more general (_lower scale_) is hard, and the other way may
bring about very strange assumptions. Does an individual event have all the
characteristics of a whole "population of events"?
- aggregation and averaging, although they offer better overview, fuzzies
the reality. They show overall trends, and ease out the noise but make
a horrible base for high scale (localized) decisions. This is an important
aspect to keep in mind, especially as the metadata description for the 1x1km
population density in the Open Data portal somehow endorses this with:
> [..] The grid-based data are the basis for competent decision-making in the
> preparation of social plans and development plans, including the regional
> development plan. The grid-based data is used in scientific studies, in the
> private sector mainly in the selection of the best location and in the
> definition of the target group. [..]

- but most important of all: don't assume uniformity where there is none. And
most natural and spatial phenomenon, even if they might show correlation, don't
have it: clear and cleancut borders, or monotonous substance.

... because you know, most probably in that polygon I used there is no-one living
there at all, unless maybe for a few lions, zebras, elephants, exotic birds,
reptiles, and others because that's roughly the territory of the Tallinn Zoo

![Testing polygon location on OpenStreetMap base. Used under CC BY-SA 2.0](../img/poly-location-on-osm.png)

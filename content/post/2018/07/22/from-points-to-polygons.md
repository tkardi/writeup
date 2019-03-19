---
title: "From points to polygons"
date: 2018-07-22T17:01:43+03:00
draft: true
---

Zip code areas. The Estonian zip codes are assigned to addresses based on
local administrative division settlements. Usually a zip code is the same for
all addresses in a settlement but in some cases like larger cities there might
be many of them. And since zip codes are used to optimize postal flow (rather
than addressing) it makes perfect sense. The zip codes are assigned based on
street name and house number so it's absolutely OK for all the house numbers
on a street to have the same zip. It might also be that even house numbers
have one zip and odd numbers on a street have another zip. Or as a combination
of ranges. So it might end up fairly complicated if we wanted to draw the areas
as a map layer. The zip code areas could be calculated as a union of voronoi
cells around addresspoints but in that case they will not follow street
centerlines among other things.

It could be argued that zipcodes are not actually geospatial phenomena
nevertheless seeing their areal extent on a map and taking it into
account in further analysis can help to _optimize_ zipcodes assigned to
addresses (essentially a question of _optimizing_ the postal flow).

In the following writeup I will cover the process of creating _nice_ zip code
areas based on streets, rivers, settlement polygons, cadastral parcel data, and
addresspoints with zip codes assigned. Basically the presented method could
be used in different cases when point data needs to be _polygonized_ in a way
that takes into account natural or humanmade (linear) phenomena.

It's still a very much work in progress and I'm mostly writing this down as a
way to document it for myself. If you find it useful or interesting or have
suggestions then feel free to ping me.

### Addendum 19.03.2019
After initially abandoning this post I saw a tweet by @tjukanov

--embed

and was intrigued to still finish this although most probably it will be a great
mess and very hard to understand.

## Input data

- Settlement level admin units that are "stretched out" to the sea (self-made).
See [subdividing space](../../../07/21/subdividing-space/) for a discussion
how it could be done.
- Streets and railroads data from the Estonian Land Board's homepage
- Cadastral parcels from the Estonian Land Board's homepage
- River centerlines from the Ministry of Environment's open access WFS service
- Address points with zip codes assigned

## Processing
The basic gist of all this is to create a bunch of polygons that cover the whole
area that you're interested in with no gaps and no ovelaps in between these
polygons and then simply _classify_ those using the point data at hand.

### Breaking the lines and then assembling again
The first task to do is take all the the linestring data (we'll gently refer
to this dataset as `noise` from this point on.) and topologically break
them at intersections (just like when preparing road network data for routing).
And then we end up with **a lot of lines** which we can compose into whatever
polygons.

![_Noise_ for Tartu county in Estonia](../img/tartumaa-noise.png)
![polygonized _noise_ for Tartu county in Estonia](../img/tartumaa-noisemerged.png)

There are different possibilities for doing this but PostGIS is a
really powerful tool for dataprocessing like this. The following SQL uses a
loop to process the data one settlement (identified by `akood` value) at a time.
`noise` will our linestring features table and `asustusyksus` where the
settlements are

![_Noise_ and settlement areas](../img/noise-plus-settlements.png)
![_Noise_ and settlement areas closeup](../img/noise-plus-settlements-2x.png)

So, in a PostGIS database:

{{< highlight sql >}}
-- create the table to hold the result
drop table if exists noise_merged;
create table noise_merged (
    oid integer,
    geom geometry(Geometry, 3301),
    akood varchar(4) -- this is the settlement identifier
);

do
$$
declare
    r record;
begin
    truncate table noise_merged;
    for r in
        -- loop by one settlement at the time
        select oid, akood, geom
        from asustusyksus
        order by oid
    loop
        begin
            insert into zip.noise_merged (
                oid, akood, geom
            )
            select
                -- let PostGIS do its magic by creating some polygons
                -- out of our noise. Dump them to singleparts.
                r.oid, bar.akood,
                (st_dump(
                    st_polygonize(
                        bar.geom
                    )
                )).geom
            from (
                -- node the noise and settlement units, and
                -- dump them to singleparts
                -- don't forget a reasonable snaptogrid here.
                select
                    (st_dump(
                        st_node(
                            st_collect(
                                st_snaptogrid(
                                    foo.geom,
                                    0.005
                                )
                            )
                        )
                    )).geom as geom, akood
                from (
                    -- get the intersection of noise in this settlement
                    select
                        r.akood, (st_dump(
                            st_intersection(n.geom, r.geom)
                        )).geom
                    from noise n
                    where st_intersects(n.geom, r.geom)
                    union all
                    -- and add settlement area geometries' all rings
                    -- dumped to linestrings
                    select
                        r.akood, st_exteriorring(
                            (st_dumprings(r.geom)).geom
                        ) as geom
                ) foo
                where geometrytype(foo.geom) = 'LINESTRING'
                group by akood
            ) bar
            group by akood;

            if r.oid % 100 = 1 then
                -- raise a message to show how far we've come after every 100
                -- settlements that have been processed
                raise notice '%, %', clock_timestamp(), r.oid;
            end if;
        exception
            when others then
                raise warning 'Loading of record % failed: %', r.akood, SQLERRM;
        end;
    end loop;
end
$$;

alter table noise_merged add column gid serial;
alter table noise_merged add constraint pk__noisemerged primary key (gid);
create index sidx__noisemerged on noise_merged using gist(geom);
{{</ highlight >}}

Which might take some time but will eventually output a result like in the
following images

![_Noise_ and settlement area boundaries polygonized](../img/noise-merged.png)

![_Noise_ and settlement area boundaries polygonized (closeup)](../img/noise-merged-2x.png)

These form the basis for the classification we'll uptake next, and therefore
to keep some context with the data, refer to them as `quartiers`. It would be
advisable to topo-check this layer aswell with something like GRASS'
[`v.clean`](https://grass.osgeo.org/grass77/manuals/v.clean.html) for example.

### Classifying the quartiers

### Diving deeper width cadastral parcels

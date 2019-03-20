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

and was intrigued to still get it finished because there is some similarity
between these tasks of creating areal data out of point-based data.

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

Still. The good thing now is that any point data you have and need to
turn into polygons can be done based on this same set of `quartiers`.

### Classifying the quartiers
Now with zipcodes the first thing we'll check is whether all address points
in a settlement (that is unit identified by `akood`) have the same zip. With
this we'll have a pretty good amount of `quartiers` already settled.

{{< highlight sql >}}
-- create table listing all possible combinations
-- of adminunits-to-zipcodes
drop table if exists distinct_zip;
create table distinct_zip as
select
    distinct mkood as a1, okood as a2, akood as a3,
    postiindeks as zip
from addrespoint;

-- keeping track which quartier has a zip assigned
alter table noise_merged add column got_zip boolean;

-- and make a new table based on those settlements
-- that only have one zip code
drop table if exists zip_areas;
create table zip_areas as
select
    d.a1, d.a2, d.a3, d.zips[1] as zip, nmt.gid as nm_gid,
    nmt.geom
from (
    select
        a1, a2, a3, array_agg(zip) as zips,
        count(1) as zip_count
    from distinct_zip
    group by a1, a2, a3
    having count(1) = 1
    order by a1, a2, a3
) d, noise_merged nmt
where nmt.akood = d.a3;

-- mark these quartiers as used
update noise_merged set
    gotzip = true
from zip_areas
where zip_areas.nm_gid = noise_merged.gid;
{{</ highlight >}}

Next up get all overleft quartiers which have only one zipcode available (
through e.g. `st_within` spatial relation). Precalculate distincts

{{< highlight sql >}}
drop table if exists distinct_zip_noise;
create table distinct_zip_noise as
select gid, a3, array_agg(zip) as zips, count(1) as zip_count
from (
    select nmt.gid, nmt.akood as a3, pa.postiindeks as zip
    from noise_merged nmt, addresspoint pa
    where st_within(pa.geom, nmt.geom)
    and nmt.gotzip is null
    group by nmt.gid, nmt.akood, pa.postiindeks
) foo
group by gid, a3;
{{</ highlight >}}

And now add some more `quartiers` as zip areas

{{< highlight sql >}}
insert into zip_areas (
    a1, a2, a3,
    zip, nm_gid, geom
)
select
    ay.mkood as a1, ay.okood as a2, ay.akood as a3,
    d.zips[1] as zip, nmt.gid as nm_gid, nmt.geom
from (
    select
        n.gid, n.a3, n.zips
    from zip.pa_distinct_zip_noise n
    where zip_count = 1
) d, zip.noise_merged_test nmt, (
    select
        distinct akood, okood, mkood
    from asustusyksus) ay
where nmt.gid = d.gid and d.a3 = ay.akood and nmt.gotzip is null;

update noise_merged set
    gotzip = true
from zip_areas
where zip_areas.nm_gid = noise_merged.gid and
noise_merged_test.gotzip is null;
{{</ highlight >}}

Now looking at the layer we can see, that tgere area still quite many
unclassified quartiers - part of them is missing and addresspoint within
them, and the other part has more than one zip.

![Quartiers with unique zips colored by zip code](../img/unique-by-zip.png)

With the first part (no addrespoint) we cannot do anything much right now. But
if there are more than one distinct zip codes present with addresspoints in the
`quartier`, then we could either divide the quartier through voronoi polygons,
or use some inner-`quartier` features, e.g. cadastral parcels.

### Dividing some more with cadastral parcels
Luckily, the Estonian Land Board has made cadastral parcel data opendata with
a permissive licence so you can freely download it and use it. So as before
we'll start by selecting only tose parcels that have address points with one
distinct zip only.

{{< highlight sql >}}
drop table if exists parcel_noise;
create table parcel_noise as
select
    *
from (
    select
        k.gid, nmt.gid as nm_gid, k.geom as geom
    from parcel k, noise_merged nmt
    where
        st_intersects(k.geom, nmt.geom) and
        st_touches(k.geom, nmt.geom) = false and
        nmt.gotzip is null
) foo
where geometrytype(geom) = 'POLYGON';

alter table parcel_noise add column oid serial;
alter table parcel_noise add constraint pk__kataster_noise primary key (oid);
create index sidx__parcel_noise on zip.parcel_noise using gist (geom);
{{</ highlight >}}

which outputs a layer like

![Parcels split by quartiers](../img/parcel-noise-2x.png)

Let's record the distinct zipcodes available in these "parcels split by
`quartiers`"

{{< highlight sql >}}
alter table parcel_noise add column zips varchar[];

update parcel_noise set
    zips = n.zips
from (
    select
        kn.oid, array_agg(distinct pa.postiindeks) as zips
    from parcel_noise kn, addresspoint pa
    where st_within(pa.geom, kn.geom)
    group by kn.oid
) n
where n.oid = parcel_noise.oid;
{{</ highlight >}}

Now taking out first those that have have only one distinct zip present

{{< highlight sql >}}
drop table if exists zip.parcel_noise_pazip;
create table zip.parcel_noise_pazip as
select
    array_agg(kn.oid) as kn_oid, kn.nm_gid, kn.pa_zips[1] as zips,
    st_union(kn.geom) as geom
from parcel_noise kn
where array_length(kn.pa_zips, 1) = 1
group by kn.nm_gid, kn.zips;
{{</ highlight >}}

Those parcels that have more than one distinct zip present could be then further
divided based on something. For this layer's creation we decided to keep
a comma-separated list of zips for these parcels to be further refined later on
case-by-case by a specialist.

{{< highlight sql >}}
drop table if exists zip.parcel_noise_zip;
create table zip.parcel_noise_zip as
select n.*
from (
    select
        bar.knoid, bar.nm_gid, zip, (st_dump(
            st_intersection(bar.geom, nmt.geom)
        )).geom as geom
    from (
        select
            array_agg(knoid) as knoid, nm_gid, zip,
            st_union(geom) as geom
        from (
            select
                n.knoid, kn.nm_gid, kn.geom as geom,
                array_to_string(kn.zips,',') as zip
            from (  
                select
                    unnest(array_agg) as knoid
                from zip.parcel_noise_pazip
                union
                select
                    oid
                from zip.parcel_noise
                where array_length(zips, 1) > 1
            ) n, zip.parcel_noise kn
            where kn.oid = n.knoid
        ) foo
        group by nm_gid, zip
    ) bar, zip.noise_merged nmt
    where bar.nm_gid = nmt.gid
) n
where geometrytype(geom) = 'POLYGON';

alter table zip.parcel_noise_zip add column oid serial;
alter table zip.parcel_noise_zip add constraint pk__parcel_noise_zip primary key (oid);
{{</ highlight >}}

Remove all parcels from this game that don't have an addresspoint

{{< highlight sql >}}
alter table zip.parcel_noise_zip add column has_ap boolean;

update zip.parcel_noise_zip set
    has_ap = true
from zip.addrespoint
where st_within(addresspoint.geom, parcel_noise_zip.geom);

delete from zip.parcel_noise_zip where has_ap is null;
{{</ highlight >}}

And then marvel at the output.

![Parcels colored by their associated zip code](../img/parcel-noise-zip-2x.png)

And if we add the already classified areas to the map:

![Parcels, settlement and quartier-based zip areas colored by their associated zip code](../img/parcel-noise-zip-plus-2x.png)

So there are still gaps in between the classified areas. One way to overcome
this is by dividing the space within a quartier between the parcels that are
there already. The steps of the process are the same as described in
[this post](../../21/subdividing-space/) so I wont go into any details with that
here. We'll pick up where we're finished with the area subdivision

![Enlarged parcel, settlement and quartier-based zip areas colored by their associated zip code](../img/parcel-noise-zip-plus-2x.png)

---
title: "2022 / Day 08: Data: OpenStreetMap"
date: 2022-11-07T15:03:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Reversed challenge: turning your spatial road network data to OSM XML"
---
There are some tools out there to do exactly this (e.g.
[osmosis](https://wiki.openstreetmap.org/wiki/Osmosis)) on an industrial scale,
and you should definetely use the right tools for this job. But let's say
(for the sake of this #30DayMapChallenge theme) I want to do this by hand. And
when I say by hand I mean compose a SQL query in PostgreSQL/PostGIS that would
(based on my input data) output a correctly formatted XML. Except for data
content/tagging of course :).

As a side-gain doing it yourself, without any 3rd party software,
will surely help understand the
[OSM XML](https://wiki.openstreetmap.org/wiki/OSM_XML) data model too. The best
resource for diving into OpenStreetMap tagging is the
[OSM wiki Tags page](https://wiki.openstreetmap.org/wiki/Tags).

For the past year or so I've been involved in a project utilizing
[OSRM](https://github.com/Project-OSRM/osrm-backend) running custom profiles,
on custom street network data (meaning _non-OpenStreetMap pbf_). For checking
if the data is converted correctly (i.e. mostly meaning if I have understood
the OSM specification correctly, and for example that I have not misspelled
`conditional`, which btw happened last November leading me on a 4-hour bug
hunting spree of _why-oh-why are conditional restrictions not picked up_), and
do custom profiles work as expected, I ended up composing this _mini-graph_ in
multiple tag forms to check if the returned route from
[OSRM](https://github.com/Project-OSRM/osrm-backend)
is the one that I was really expecting.

![My custom homemade matrix for "unit-testing" OSRM profiles and my own
understanding of OpenStreetMap tagging rules. The SQL query
below will build exactly this matrix](../img/test-osm-matrix.png)

And with a gazillion of corresponding OSM XML-format files I could test on the
go very quickly:

- are the correct maxspeed values processed per profile,
- are the correct restrictions picked up per per profile,
- does access tagging work?
- what about conditional restrictions? do they work?
- ... and if I change _this little bit_, will all of it still function as
expected or come dumbling town.

These files are pretty small and you want to keep them to the bare minimum in
them - so for example if testing for `access` / `acess:forward` /
`access:backward` / `access:backward=no emergency:backward=yes` you don't want
other things (like `no_turn` restrictions) messing with your result. It's
really easier to manage them by hand (and yes. Edit XML by hand. You read
correctly!) because you'll only be changing a few tag values per specific test.
But in essence it would be possible to set up a pipeline from a
PostgreSQL/PostGIS database based on a query like todays (see below).

Although I myself am not a huge fan of xml based formats because of their
verbosity and poor streaming capabilities (here's looking at you,
[Inspire](https://inspire.ec.europa.eu/) :D ), nevertheless [PostgreSQL has
some fantastic XML processing functions](https://www.postgresql.org/docs/current/functions-xml.html)
which are absolutely priceless for a job like this. The ones that are used most
frequently during todays query are:

- `xmlelement` which produces an xml element with a given name, attributes and
content
- `xmlattributes` which creates attributes for the xml element, and
- `xmlagg` which aggregates mutiple xml elements together (think of this as
aggregating with [`st_collect`](https://postgis.net/docs/ST_Collect.html)
in case of geometries).

And sorry, todays _output image_ will be a png screenshot file of XML file
contents. Which is just as good as

> Yes, sure we have locations of our shops. Please find them in the
> attachment of this email which is a jpg file of a photo of a paper map we took
> where we have marked all the locations with a ballpoint pen.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-8-openstreetmap.png)

**P.S.** pretty-print added to the file by hand :)

{{< highlight sql >}}
with
    /* nodes are the basic building blocks of all OSM data.
       Everything is either a node or is made up of a series of nodes.
       Node id values start from the north and follow left-to-right ordering
    */
    nodes as (
        select
            node.id, st_snaptogrid(st_transform(node.geom, 4326), 0.000001) as geom
        from (
            values
                (
                    1,
                    st_setsrid(
                        st_point(614500, 6502500 + sqrt(500*500 + 500*500)),
                        3301
                    )
                ),(
                    2,
                    st_setsrid(
                        st_point(614500, 6502500),
                        3301
                    )
                ),(
                    3,
                    st_setsrid(
                        st_point(614000, 6502000),
                        3301
                    )
                ),(
                    4,
                    st_setsrid(
                        st_point(615000, 6502000),
                        3301
                    )
                ),(
                    5,
                    st_setsrid(
                        st_point(614500, 6501500),
                        3301
                    )
                ),(
                    6,
                    st_setsrid(
                        st_point(614500, 6501500 - sqrt(500*500 + 500*500)),
                        3301
                    )
                )
            ) node(id, geom)
    ),
    /* Construct ways out of nodes with some basic attributes (highway, oneway,
       access, maxspeed). For easier understanding (w/o looking at the map)
       way_id values are concatenated from node id values, so you'd know that
       way_id=12 is between nodes 1 and 2.
    */
    ways as (
        select
            way.*
        from (
            values
                (
                    12,array[1,2],'primary',false,
                    true,true,50,50
                ),(
                    23,array[2,3],'primary',false,
                    true,true,50,50
                ),(
                    24,array[2,4],'primary',false,
                    true,true,50,50
                ),(
                    25,array[2,5],'primary',false,
                    true,true,1,1
                ),(
                    35,array[3,5],'primary',false,
                    true,true,50,50
                ),(
                    45,array[4,5],'primary',false,
                    true,true,50,50
                ),(
                    56,array[5,6],'primary',false,
                    true,true,50,50
                )
        ) way(
            id, node_ids, highway, oneway,
            access_fwd, access_bck, maxspeed_fwd, maxspeed_bck
        )
    ),
    restrictions as (
        /* Construct restrictions for the ways. These will be encoded as
           "relations" in the XML.
        */
        select
            restriction.*
        from (
            values
                (
                    1223, 12, 2, 23,
                    'no_right_turn @ (Mo-Fr 07:00-08:00;Sa-Su 19:00-20:00)',
                    'emergency'
                ),(
                    6554, 65, 5, 54,
                    'no_right_turn @ (Mo-Fr 07:00-08:00;Sa-Su 19:00-20:00)',
                    'emergency'
                ),(
                    1225, 12, 2, 25,
                    'no_turn',
                    null
                )
        ) restriction (
            id, from_way, via_node, to_way,
            val, except_mode  
        )
    ),    
    meta as (
        /* a general section of various meta that we'll write into the xml*/
        select
            session_user as user, r.oid as uid,
            bounds.minx, bounds.miny, bounds.maxx, bounds.maxy,
            to_char(clock_timestamp(), 'YYYYMMDDHH24MISS') as changeset,
            0 as version
        from (
            select
                st_envelope(st_collect(nodes.geom)) as e
            from
                nodes
        ) b
            join lateral (
                select
                    min(st_x(pts.geom)) as minx, min(st_y(pts.geom)) as miny,
                    max(st_x(pts.geom)) as maxx, max(st_y(pts.geom)) as maxy
                from st_dumppoints(b.e) pts
            ) bounds on true
            left join
                pg_roles r on
                    r.rolname=session_user
    )
/* ... and bringing it all together*/
select
    xmlelement(
        name osm,
        xmlattributes(
            0.6 as version
            '@tkardi for #30DayMapChallenge 2022 | Day 8: OpenStreetMap' as generator,
            'Reversed challenge: turning your spatial road network data to OSM XML' as attribution,
            'https://tkardi.ee/writeup/30daymapchallenge/08-openstreetmap-2022/' as license
        ),
        xmlagg(d.content)
    )
from (
    /* the BOUNDS tag for the output file*/
    select
        xmlelement(
            name bounds,
            xmlattributes(
                round(minx::numeric,6) as minlon,
                round(miny::numeric,6) as minlat,
                round(maxx::numeric,6) as maxlon,
                round(maxy::numeric,6) as maxlat
            )
        ) as content
    from
        meta
    union all
    /* the NODE tags for the output*/
    select
        xmlelement(
            name node,
            xmlattributes(
                nodes.id as id,
                meta.version as version,
                meta.uid as uid,
                meta.user as user,
                meta.changeset as changeset,
                round(st_y(nodes.geom)::numeric,6) as lat,
                round(st_x(nodes.geom)::numeric,6) as lon
            )
        ) as content
    from
        nodes, meta
    union all
    /* the WAY tags for the output*/
    select
        xmlelement(
            name way,
            xmlattributes(
                ways.id as id,
                meta.version as version,
                meta.uid as uid,
                meta.user as user,
                meta.changeset as changeset
            ),
            nd.v,
            access.val,
            maxspeed.val,
            xmlelement(
                name tag,
                xmlattributes(
                    'highway' as k,
                    ways.highway as v
                )
            )
        )
    from
        meta,
        ways

            /* Add node.id references to way*/
            join lateral (
                select
                    xmlagg(
                        xmlelement(
                            name nd,
                            xmlattributes(n.id as ref)
                        ) order by n.i
                    ) v

                from
                    unnest(ways.node_ids)
                        with ordinality n(id, i)
            ) nd on true

            /* prepare access tag and possible combinations depending
               on direction
            */
            join lateral (
                select
                    xmlagg(
                        xmlelement(
                            name tag,
                            xmlattributes (
                                k.key as k,
                                (array[
                                    coalesce(ways.access_fwd,true),
                                    coalesce(ways.access_bck,true)
                                ])[k.i] as v
                            )
                        )
                    ) as val
                from (
                    select
                        case
                            when coalesce(ways.access_fwd,true) =
                                coalesce(ways.access_bck, true) then
                                array['access']
                            else
                                array['access:forward','access:backward']
                        end as k
                ) a
                    join lateral unnest(a.k) with ordinality k(key, i) on true
            ) access on true

            /* prepare maxspeed tag and possible combinations depending
               on direction
            */
            join lateral (
                select
                    xmlagg(
                        xmlelement(
                            name tag,
                            xmlattributes (
                                k.key as k,
                                (array[
                                    coalesce(ways.maxspeed_fwd,0),
                                    coalesce(ways.maxspeed_bck,0)
                                ])[k.i] as v
                            )
                        )
                    ) as val
                from (
                    select
                        case
                            when coalesce(ways.maxspeed_fwd,0) =
                                coalesce(ways.maxspeed_bck, 0) then
                                array['maxspeed']
                            else
                                array['maxspeed:forward','maxspeed:backward']
                        end as k
                ) a
                    join lateral
                        unnest(a.k) with ordinality k(key, i) on true
            ) maxspeed on true
    union all
    /* the RELATION tags for the output*/
    select
        xmlelement(
            name relation,
            xmlattributes(
                restrictions.id as id,
                meta.version as version,
                meta.uid as uid,
                meta.user as user,
                meta.changeset as changeset
            ),
            xmlelement(
                name member,
                xmlattributes(
                    'way' as type,
                    restrictions.from_way as ref,
                    'from' as role
                )
            ),
            xmlelement(
                name member,
                xmlattributes(
                    'node' as type,
                    restrictions.via_node as ref,
                    'via' as role
                )
            ),
            xmlelement(
                name member,
                xmlattributes(
                    'way' as type,
                    restrictions.to_way as ref,
                    'to' as role
                )
            ),
            xmlelement(
                name tag,
                xmlattributes(
                    'restriction'||k.c as k,
                    restrictions.val as v
                )
            ),
            ex.v,
            xmlelement(
                name tag,
                xmlattributes(
                    'type' as k,
                    'restriction' as v
                )
            )
        )
    from
        meta,
        restrictions
            join lateral (
                /* is it a conditional restriction or simple*/
                select
                    case
                        when restrictions.val ~ '@' then ':conditional'
                        else ''
                    end as c
            ) k on true
            join lateral (
                /* parse excepts for restrictions*/
                select
                    case
                        when restrictions.except_mode is not null then
                            xmlelement(
                                name tag,
                                xmlattributes(
                                    'except' as k,
                                    restrictions.except_mode as v
                                )
                            )
                        else ''
                    end as v
            ) ex on true
) d
;
{{</ highlight >}}

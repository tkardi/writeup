---
title: "2022 / Day 05: Ukraine"
date: 2022-11-04T18:25:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
description: "#30DayMapChallenge: Building non-overlapping point buffers in PostGIS."
---
Most probably the easiest way to create non-overlapping buffers to points in
PostGIS is to buffer the points using
[st_buffer](https://postgis.net/docs/ST_Buffer.html) to whatever distance is
needed (could be variable width aswell, e.g. by calculating distances to the
closest _other points_ for example by triangulating with
[st_delaunaytriangles](https://postgis.net/docs/ST_DelaunayTriangles.html)),
followed by intersecting the buffers with voronoi polygons built on the same
points using
[st_voronoipolygons](https://postgis.net/docs/ST_VoronoiPolygons.html).

The next SQL snippet includes Ukrainian first level administrative division
unit centroids calculated from
[NaturalEarth](https://www.naturalearthdata.com/downloads/10m-cultural-vectors/)
large scale cultural vector data. The buffers are created with a width that's
defined by a `generate_series` to get concentric ring buffers.

[output image](https://tkardi.ee/writeup/img/30daymapchallenge/2022/day-5-ukraine.png)

{{< highlight sql >}}
with
    x as (
        select
            name::varchar,
            st_transform(geom::geometry(point, 4326), 6385) as geom
        from (
            values
                (
                    'Чернігівська область',
                    '0101000020E610000008AA5D3060FF3F40E377ECD791AD4940'
                ),(
                    'Волинська область',
                    '0101000020E61000009C392B2635D83840BD7A93E5A68F4940'
                ),(
                    'Рівненська область',
                    '0101000020E610000090DFE64462623A4024819CBC437C4940'
                ),(
                    'Житомирська область',
                    '0101000020E61000009950FEA0DB763C400CD4C37BF94E4940'
                ),(
                    'Київ',
                    '0101000020E6100000D9FBE7556FB43E40B1000C5D7B2A4940'
                ),(
                    'Вінницька область',
                    '0101000020E6100000493EAEE827B93C405D439F1571784840'
                ),(
                    'Львівська область',
                    '0101000020E610000070ECEE1B93093840C9C59B2590D74840'
                ),(
                    'Сумська область',
                    '0101000020E6100000788F10426E284140049C2645F79D4940'
                ),(
                    'Харківська область',
                    '0101000020E610000004655774073B42403F3C648F5EBD4840'
                ),(
                    'Луганська область',
                    '0101000020E61000007D5CFBAA70804340E679BBBBF4774840'
                ),(
                    'Закарпатська область',
                    '0101000020E6100000655F41D4106337400FEDE13AE23E4840'
                ),(
                    'Донецька область',
                    '0101000020E6100000C8B2FAE48BE542408476F217DE064840'
                ),(
                    'Чернівецька область',
                    '0101000020E6100000E077689337373A4080C5636117194840'
                ),(
                    'Івано-Франківська область',
                    '0101000020E610000080FA3FABED9638407C7B3AF3054F4840'
                ),(
                    'Одеська область',
                    '0101000020E6100000A757AE74CEC03D40C8EF7109E45B4740'
                ),(
                    'Автономна Республіка Крим',
                    '0101000020E6100000E87C5CAEAD474140B4F4188C58A64640'
                ),(
                    'Херсонська область',
                    '0101000020E61000008CA5642C2CB240401112CF2636534740'
                ),(
                    'Запорізька область',
                    '0101000020E6100000BCB5735C2DDB414077430D51A9984740'
                ),(
                    'Миколаївська область',
                    '0101000020E6100000771FA7AE3CAC3F405CAC84B7A7A94740'
                ),(
                    'Полтавська область',
                    '0101000020E6100000FC35BA3DDAE24040740EB68DBED24840'
                ),(
                    'Хмельницька область',
                    '0101000020E6100000472BFE03C9043B40B7859AA57AC24840'
                ),(
                    'Тернопільська область',
                    '0101000020E610000030AD8A29368B3940ECA8389C9FB14840'
                ),(
                    'Дніпропетровська область',
                    '0101000020E6100000B0F814CD4C7A414004135D5B0B2A4840'
                ),(
                    'Черкаська область',
                    '0101000020E6100000B0B8ECF25F393F407FE744101BAD4840'
                ),(
                    'Кіровоградська область',
                    '0101000020E610000048BA66072DD43F40B473E2C698404840'
                ),(
                    'Київ',
                    '0101000020E610000027B174DE19843E40678B33EEAA2B4940'
                )
        ) x(name, geom)
    )
select
    row_number() over()::int as oid, w.name, s as buffer_size,
    st_intersection as geom
from
    generate_series(5000, 150000, 5000) s, (
        select
            st_collect(geom)
        from
            x
    ) d            
        join lateral
            st_buffer(
                st_collect,
                s
            ) on true
        join lateral
            st_voronoipolygons(
                st_collect,
                0.0,
                st_buffer
            ) on true
        join lateral
            st_dump(
                st_voronoipolygons
            ) on true
        join lateral
            st_intersection(
                st_buffer,
                st_dump.geom
            ) on true
        join lateral (
            select x.name
            from x
            where st_within(x.geom, st_intersection)
            order by x.geom <-> st_intersection
            limit 1
        ) w on true
;
{{</ highlight >}}

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

In the following writeup I will cover the process of creating _nice_ zip code
areas based on streets, rivers, settlement polygons, cadastral parcel data, and
addresspoints with zip codes assigned. Basically the presented method could
be used in different cases when point data needs to be _polygonized_ in a way
that takes into account natural or humanmade (linear) phenomena.

It's still a very much work in progress and I'm mostly writing this down as a
way to document it for myself. If you find it useful or interesting or have
suggestions then feel free to ping me.

## Input data
- Settlement level admin units that are "stretched out" to the sea (self-made).
See [subdividing space](../../../07/21/subdividing-space/) for a discussion
how it could be done.
- Streets data from the Estonian Land Board's homepage
- Cadastral parcels from the Estonian Land Board's homepage
- River centerlines from the Ministry of Environment's open access WFS service

## Processing
As a first step we'd have to create some meaningful

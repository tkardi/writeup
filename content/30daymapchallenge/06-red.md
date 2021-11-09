---
title: "Day 06: red"
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
---
### Data
- [addresses from the Estonian Land Board](https://geoportaal.maaamet.ee/eng/Spatial-Data/Address-Data-p313.html)
- [Omniva postoffice locations](https://www.omniva.ee/private/map/locations)
- [OSRM routing based on OpenStreetMap contributors](https://www.openstreetmap.org/)

### Tools
Python, PostGIS, OSRM, QGIS.

### Summary
Back about a year ago I had a task of calculating closest postoffices with
shortest path routes for all building addresses in Estonia. Back then some
further analysis followed but now it's possible to take an easy stroll with this
calculation result:

- add routing lines to QGIS. These cover each other in a lot of places so in
order make it visually obvious that some routes are more frequently traveled
others less assign lines a 1% opacity and 0.1 mm width.
- add addresspoints to QGIS. Symbolize it as quite small (e.g. 0.4 mm) and
25% opacity.

Export, and...

[hi-res](https://tkardi.ee/writeup/img/30daymapchallenge/day-6-red.png)

{{< tweet 1325070797812232193 >}}

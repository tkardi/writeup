---
title: "Drawing density lines in Leaflet.js"
date: 2018-02-19T12:00:58+02:00
draft: false
---

{{< load-leaflet >}}

First off, I'm a very inefficient javascript writer so my apologies if
something seems amiss or strange.

Some weeks ago I got a bunch of environmental monitoring data and a task to
somehow organize the data and make it openly available on our public GeoServer
as W*S services.

The amount of different datasets was quite big so I decided to concentrate on a
smaller subset of 1*1 km2 gridded data on air emissions. This consisted of
87346 cells - a 367*238 rectangular grid in the local national coordinate
reference system [L-EST'97](http://epsg.io/3301). For every cell had the
attributes of population count estimates by Statistics Estonia and a wide array
of air pollutant emissions (e.g NOx, SO2, CO, PMs by different diameters and
so on).

In order to make sure I was on (an approximately) right track with this thing I
decided to have a go and play a bit with the possible outcome of the services
in a [Leaflet.js](http://leafletjs.com) powered map aswell.

Rendering 90K points as markers on a webmap is painful. But I guess that's not
really anything new. But more importantly - considering the essence of
this data, rendering it point by point yields nothing really particular. So what
are the options?

## Heatmaps
Sure. There are a couple of plugins for drawing heatmaps in Leaflet, the one
that I checked out was [heatmap.js](https://www.patrick-wied.at/static/heatmapjs/)
[plugin for Leaflet](
https://www.patrick-wied.at/static/heatmapjs/plugin-leaflet-layer.html):

{{< leaflet-heat
    id="1"
    height="350px" width="100%"
    lon="24.905" lat="58.681"
    z="6"
    dataURL="../population.geojson"
    dataAttribution=""
    dataKey="population"
    dataMin="4"
    dataMax="5000" >}}

The configuration of the heatmap layer here is pretty straightforward and leaves
ample room for adjustments, incl. gradient colors ([as outlined here](
https://www.patrick-wied.at/static/heatmapjs/example-full-customization.html) and
in an example below). The only thing that requires some work is the data itself,
we'll need to map GeoJSON geometries as `longitude`, `latitude` properties
(names don't matter, these are configurable). But this can easily be achieved
by

{{< highlight javascript >}}
var features = [];
// @param data: is our GeoJSON FeatureCollection variable
// @param key: is the property name we need to count
// @param minValue: is for discarding background data
data.features.filter(function(feature) {
        return feature.geometry.type == 'Point' &&
            feature.properties[key] >= minValue;
    }).forEach(function(feature) {
        features.push({
            lng:feature.geometry.coordinates[0],
            lat:feature.geometry.coordinates[1],
            count:feature.properties[key]
        });
    });
{{< / highlight >}}

So for example instead of population we could do a NOx concentration
heatmap aswell using a nitrous violet gradient this time:

{{< leaflet-heat
    id="2"
    height="350px" width="100%"
    lon="24.905" lat="58.681"
    z="6"
    dataURL="../nox.geojson"
    dataRadius="0.05"
    dataAttribution=""
    dataKey="nox"
    dataMin="0.11"
    dataMax="15"
    gradient="{\"0.4\":\"#1D022D\",\"0.7\":\"#9F07F7\",\"1\":\"#F7C9FF\"}" >}}

This looks eye-catching but one of the issues here is that coming
from a strict _choropletic_ yes-or-no mindset it does not visually depict
real values ("NOx concentration at this click location is this-and-that much")
but rather the hotspots where values are higher or lower.

So if it's going to be vague-ish anyway, then why not something like those
weather report barometric pressure maps with isolines. Or elevation maps - they
use isolines aswell.

## Isolines
One really cool library for javascripting geodata that I've been meaning to
dig into is [turf.js](http://turfjs.org/). And along other things turf.js
offers the possibility of creating [isolines](http://turfjs.org/docs#isolines).
Here's an excerpt from our would-have-been conversation:

turf-isolines: Your data should be a point grid.

Me: Cool. I have that. I'll just transform it to WGS'84 geographical coordinates.

![1km*1km point grid in L-EST'97](/img/20180219/km_grid_3301.png)

< transforms data >

Me: Here you go ...

![1km*1km point grid in WGS84](/img/20180219/km_grid_4326.png)

turf-isolines: Errr... right, this should've been an equirectangular point grid
in geographic coordinates.

Me: Oh wait, but I can use [turf.interpolate](http://turfjs.org/docs#interpolate)
to make that on the fly.

< multiple browser crashes later >

Me: Ok. I get it - 90K points is way too much in one go.

So the options here would be to either a) prepare the data already offline, or
b) do the processing in smaller chunks.

As I'm not really interested in any preprocessing server-side because that
would take away the fun of javascripting. So lets give it a go with the
smaller chunks workaround.

Everything seems to work out fine performance-wise as long as the number of
points to be interpolated is kept around a few thousand tops and the resolution
of the output grid is not too dense. I found it useful to start with  sparser
grids (say 50km resolution considering the spatial extent of Estonia ;)) and
then decreasing it to see at what size the waiting time becomes unbearable or
browser crash may occur.

A few scenarios were tested how to go about this, but the main problem with
all of them was that as soon as the 0-values were left out (and for the NOx
dataset these make up around 90% of the data) the output isolines lost all
connection with the real situation - literally the whole country seemed to be
very highly polluted with &lt;insert the pollutant name you're mapping&gt;.

In the end as a final grasp just before calling it quits, I decided to try to
(automatically) _cherry pick_ points to represent the 0-values. As these could
be considered as the background values, it seemed plausible to go with for
example: "lets use 1000 points out of `n` total, so give me every `n/1000`th
point" (NB! this dataset is spatially ordered). Something in the lines of:

{{< highlight javascript >}}
// @param data: is our GeoJSON FeatureCollection variable
// @param key: is the property name we are mapping
// @param numZeros: is the number of zeros we want to retain
var fc = data.features.filter(function(feature) {
        return feature.properties[key] > 0}
    ),
    zeros = turf.getCluster(data, {key: 0}),
    l = zeros.features.length;

for (var i=0;i<numZeros;i++) {
    var j = parseInt(i*l/numZeros);
    fc.push(zeros.features[j]);
}
{{< / highlight >}}

Add some value-based styling using a function, and the result we get looks like
this using 2.5K _zero-values_:

{{< leaflet-iso
    id="5"
    height="350px" width="100%"
    lon="24.905" lat="58.681"
    z="6"
    dataURL="../nox.geojson"
    dataKey="nox"
    dataBreaks="[-1, 0, 0.1, 1, 10, 100, 1000, 10000]"
    dataStyle="{\"1000-10000\":[\"#800026\",0.8],\"100-1000\":[\"#BD0026\",0.8],\"10-100\":[\"#FC4E2A\",0.8],\"1-10\":[\"#FEB24C\",0.8],\"0.1-1\":[\"#FFEDA0\",0.8],\"0-0.1\":[\"white\",0.0],\"-1-0\":[\"white\",0.0]}"
    dataAttribution=""
    >}}

There is some inherent spatial bias in this representation. Remember, the
background zero-values were picked out based on their order in the dataset.
Which means that larger areas with `> 0` values will less likely have
background zeros present. A better solution here is needed but it will have
to be another day to try this out.

## Density lines?

So really, what we're looking for is a way we could squash the data without
any implications on the map's performance. For example, if instead of 367*238
points we had only 238 linestrings which are 367 vertices long to work with?
In that case we could style them using color gradients based on values or line
thickness or something similiar...

Or we could draw linechart-like lines. Which reminded me
of a blogpost about a global [population-lines](http://spatial.ly/2014/08/population-lines/)
map by [James Cheshire](https://twitter.com/spatialanalysis) I read some odd years
ago and back then thought this-is-really-nice and
pity-i-don't-know-any-place-to-reuse-this.

The idea itself being really simple - offset points' latitude coordinate values
by whatever other value you are mapping. We can offset it north or south,
calculate the new location at some angle or so on. So for the NOx data we could
get something in the line of

{{< leaflet-unknwn-plsrs
    id="3"
    height="350px" width="100%"
    lon="24.905" lat="58.681"
    z="6"
    dataURL="../nox.geojson"
    dataKey="nox"
    type="lines"
    dataWidth="367"
    dataHeight="238"
    dataSlope="5"
    dataDenominator="10"
    dataSaturateAt="0.1"
    dataStyle="{\"weight\":0.2,\"color\":\"red\"}"
    >}}

In the background for the calculation we're simply looping through
all the features and offsetting their geometry's y-coordinate by some
arbitrary scaled value. The way we do it here is:

{{< highlight javascript >}}
// @param feature: the GeoJSON feature at hand
// @param key: the property name we're mapping
val = feature.proprties[key];
val = slope * (val / denominator);
sign = val < 0 ? -1 : 1;
saturateAt = 0.1;
val = Math.abs(val) > saturateAt ? sign * saturateAt + val * 0.01 : val;
{{< / highlight >}}

The values for `slope` and `denominator` depend on the original
values that we want to plot, so they'll need a bit fiddling around to get the
data looking OK. Just have to keep in mind that we are mapping in EPSG:4326
decimal degrees so there is a chance these calculated values will shoot a
couple of rounds around the globe. To combat this, the previous example
saturates the newly calculated y-coordinate deltas at 0.1 degrees (`saturateAt`)
and uses a lower slope for those values.

During recalculation one of the things that we'll do extra is add the original
value for every point coordinate as a z-coordinate (maybe some onmouseoverfun
later?)

The order of features in the file starts in the lower left corner, is 367
features wide, follows a left-to-right direction, row-by-row for a total of
238 rows. If the features were unordered or ordered in some direction additional
sorting would be needed as well because otherwise we'll end up with some scribbles
instead.

{{< highlight javascript >}}
var w = 367,
    h = 238,
    key = "nox",
    denominator = 10,
    slope = 5,
    saturateAt = 0.1,
    lines = [];

for (var i=0;i<h;i++) {
    var features = data.features.slice(i*w, i*w+w),
        coords = features.map(function(feature, j, arr) {
            var val = feature.properties[key],
                // val is the value that we'll add to the y-coordinate
                val = slope * (val / denominator),
                // saturate value if needed
                sign = val < 0 ? -1 : 1,
                val = Math.abs(val) > saturateAt ? sign * saturateAt + val * 0.01 : val,
                coordinates = feature.geometry.coordinates;
            return [
                coordinates[0],
                coordinates[1] + val,
                feature.properties[key]
            ];
        });
    if (coords.length > 0) {
        lines.push(turf.lineString(coords));
    }
}
// and reverse lines so the northernmost will be drawn first
lines = lines.reverse();
{{< / highlight >}}

Some other things that could be done here: remove all zero-values so that
we'll have only the spikes. Now in order not to have these funny "triangles-only"
we'll discard the feature if it's mappable value is 0 the value before it in
the same row was 0 and the next value in the same row is 0. Otherwise retain it.
And instead of LineStrings, we'll process these into Polygons:

{{< highlight javascript >}}
for (var i=0;i<h;i++) {
    var features = data.features.slice(i*w, i*w+w),
        coords = features.map(function(feature, j, arr) {
            var val = feature.properties[key],
                prv = arr[j-1] ? arr[j-1].properties[key] : 0,
                nxt = arr[j+1] ? arr[j+1].properties[key] : 0,
                nxtNxt = arr[j+2] ? arr[j+2].properties[key] : 0,
                prvPrv = arr[j-2] ? arr[j-2].properties[key] : 0,
                val = slope * (val / denominator),
                sign = val < 0 ? -1 : 1
                val = Math.abs(val) > saturateAt ? sign * saturateAt + val * 0.01 : val,
                coordinates = feature.geometry.coordinates;
            if (val == 0  && (prv == 0 && prvPrv == 0 && nxt == 0 && nxtNxt == 0)) {
                // if current value is 0
                // as well as the previous and the one before
                // and the next and the one after that
                // then this point will be discarded.
                return;
            } else if (val != 0 && (j == 0 || j == w - 1) ) {
                // now this is completely wrong
                // and should not be done in real life, but still:
                // if current value != 0 but we are either at the first point
                // in the grid's left side or last point at the grid's right side
                // just surpress the value, return as 0.
                return [coordinates[0], coordinates[1], 0];
            }
            return [
                coordinates[0],
                coordinates[1] + val,
                feature.properties[key]
            ];
        });
    var ring = [];
    coords.forEach(function(e, idx, arr) {
         var nxt = arr[idx+1];
         if (e !== undefined) {
             ring.push(e);
         }
         if (nxt === undefined && ring.length > 2) {
             // we're at the last point for this
             // soon-to-be-polygon
             lines.push(turf.lineToPolygon(turf.lineString(ring)));
             ring = [];
         }
    });
}
// again, draw northernmost row first
lines = lines.reverse();

{{< / highlight>}}

With polygons we can also do fills and also gradient fills like this.

{{< leaflet-unknwn-plsrs
    id="4"
    height="350px" width="100%"
    lon="24.905" lat="58.681"
    z="6"
    dataURL="../nox.geojson"
    dataKey="nox"
    type="polygons"
    dataWidth="367"
    dataHeight="238"
    dataSlope="5"
    dataDenominator="10"
    dataSaturateAt="0.1"
    dataStyle="{\"weight\":0.2, \"fillOpacity\":0.9, \"color\":\"url(#nitrogen)\"}" >}}

It would be cool to have the gradient map to an actual value here, but currently
this is out of the scope for this writeup.

## Thank you
Leaflet, Turf and Heatmap libraries (+ maintainers) for a crash course in
JavaScript. :)

The source of this writeup is available at https://github.com/tkardi/writeup if
you're interested.
